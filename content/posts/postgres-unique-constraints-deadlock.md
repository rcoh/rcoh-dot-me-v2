---
title: "How Postgres Unique Constraints Can Cause Deadlock"
date: 2017-12-29T22:49:36-05:00
draft: false
tags: ["postgres", "databases", "devops", "deadlock"]
---



A recent outage lead me to investigate Postgres unique constraints more deeply. Postgres implements unique constraints by creating a unique index -- an index that can only contain unique values.[^4] It turns out that unique indices and concurrent transactions can interact in nasty and surprising ways. Before I get into the "why", here are the implications:

**When two transactions insert the same value into a unique index, one transaction will wait for the other transaction to finish before proceeding. Moreover, you can cause deadlock using only `inserts`.**

This means:

1. Avoid inserting duplicate keys into tables with unique constraints and relying on the unique constraint to save you. This is especially important if multiple transactions do this concurrently.
2. Avoid inserting large numbers of records during the same transaction into a table with a unique constraint, especially if you broke rule #1.
3. Avoid long transactions involving tables with unique constraints especially if you broke rule #1.

Failure to follow these guidelines can result in your application leaking databases connections and eventually hanging. Here's how:

1. `application thread 1` in `transaction 1` inserts `a`.
2. `application thread 2` in `transaction 2` inserts `a`. **`transaction 2` and `application thread 2` are now both blocked until `transaction 1` is committed or rolled back**
3. `application thread 1` now waits for something from `application thread 2`. This seems far fetched, but I've seen it multiple times in practice in seemingly reasonable code. A similar behavior can occur if `application thread 1` is blocked for an unrelated reason eg. making a long running API call.

**You now have an application level deadlock. The Postgres deadlock detector cannot save you. In most cases you will leak database connections until your application hangs. This is the worst case scenario.**

If you were a bit luckier, you could have created deadlock within Postgres. The Postgres deadlock detector will detect the deadlock after `deadlock_timeout` (typically one second)[^1] and cancel one of the transactions.[^5]. It's not uncommon to be in the worst case scenario described above, until, by sheer luck, insert ordering triggers the deadlock detector and cancels the transactions.

# Why does this happen?

In the case where only one transaction is writing to a unique index, the process is straightforward. Postgres simply looks up the tuple we're attempting to insert in the unique index. If it doesn't exist, we're good. If it already exists, our insertion will fail because we violated a unique constraint. [Here's the relevant function in the Postgres source.](https://github.com/postgres/postgres/blob/382ceff/src/backend/executor/execIndexing.c#L639)

If two transactions are writing to the index concurrently, the situation is more complex. For the purposes of our example we'll have `transaction a` and `transaction b`.

Both transactions attempt to insert value `v` into the same table. Suppose `transaction a` has inserted `v` first but has not yet committed.

`transaction b` inserts `v` which causes Postgres to check the unique index. A Postgres index stores not only committed data, but also stores data written by ongoing transactions. Postgres will look for the tuple we're attempting to insert in both the committed and "dirty" (not-yet-committed) sections of the index. In this case, it will find that another in-progress transaction (`transaction a`) has already inserted `v`.

Postgres handles this situation by having `transaction b` wait until `transaction a` completes. Every transaction holds an exclusive lock on its own transaction ID while the transaction is in progress. If you want to wait for another transaction to finish, you can attempt to acquire a lock on that transaction ID, which will be granted when that transaction finishes.[^3]

In our example, Postgres will determine the transaction ID of the other transaction (`transaction a`) and `transaction b` will attempt to acquire a lock on the transaction ID of `transaction a`.[^2]

`transaction b` is now blocked waiting for `transaction a` to finish. There are 3 possible outcomes, ordered best to worst:

### 1. Benign resolution
`transaction a` finishes promptly. `transaction b` fails with the message `duplicate key v violates unique constraint "..."`

### 2. Postgres detects deadlock
If the two transactions are each inserting multiple rows into the table, `transaction a` may attempt to insert a key previously inserted by `transaction b`. This causes `transaction a` to attempt to acquire a lock on `transaction b`. Now `b` is waiting on `a` and `a` is waiting on `b`. Deadlock! Postgres detects this after `deadlock_timeout` and one of the transactions is aborted with this somewhat confusing deadlock error:

```
ERROR:  deadlock detected
DETAIL:  Process 81941 waits for ShareLock on transaction 924071; blocked by process 81944.
Process 81944 waits for ShareLock on transaction 924069; blocked by process 81941.
HINT:  See server log for query details.
CONTEXT:  while inserting index tuple (0,20) in relation "..."
```

### 3. Infinite wait, connection pileup, or other bad things (tm)
Here's where things can go downhill. If `transaction a` is never committed, `transaction b` will remain stalled. This can happen for a few reasons, the most likely being that due to the stalled transactions, some sort of application level, outside-of-Postgres, deadlock occurs. One way this could happen is if the thread running `transaction a` is actually waiting for output from the thread running `transaction b`. Any future transactions that attempt to insert `v` (perhaps a retry?) will join the pileup. If you're lucky, the situation eventually resolves itself via #1 or #2 before you run out of database connections. If you aren't lucky, your application will run out of database connections and hang forever.

While unique constraints are great for maintaining application level consistency, they need to be used with care. Besides adding an additional index that must be maintained on-disk and in memory, they can lead to lock contention within Postgres, stalled transactions, and leaked connections.

Thanks to Leah Alpert for being unlucky enough to run into this in production, and figuring out what was going on. Her research was the basis for this post.

[^1]: https://www.postgresql.org/docs/9.1/static/runtime-config-locks.html
[^2]: [Postgres source](https://github.com/postgres/postgres/blob/382ceff/src/backend/executor/execIndexing.c#L796) `xwait` is the transaction that is also attempting to insert the same value. It's not obvious to me why Postgres implements unique indices in this way. One could imagine an implementation that let both transactions continue and the second one to commit would fail. One possible explanation is that this prevents the database from doing wasted work.
[^3]: From https://www.postgresql.org/docs/9.3/static/view-pg-locks.html: Every transaction holds an exclusive lock on its virtual transaction ID for its entire duration. If a permanent ID is assigned to the transaction (which normally happens only if the transaction changes the state of the database), it also holds an exclusive lock on its permanent transaction ID until it ends. When one transaction finds it necessary to wait specifically for another transaction, it does so by attempting to acquire share lock on the other transaction ID (either virtual or permanent ID depending on the situation). That will succeed only when the other transaction terminates and releases its locks.
[^4]: https://www.postgresql.org/docs/9.4/static/indexes-unique.html
[^5]: https://www.postgresql.org/docs/9.1/static/explicit-locking.html: PostgreSQL automatically detects deadlock situations and resolves them by aborting one of the transactions involved, allowing the other(s) to complete. (Exactly which transaction will be aborted is difficult to predict and should not be relied upon.)
