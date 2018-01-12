---
title: "Resource Pools and Deadlock"
date: 2018-01-11T09:17:30-08:00
draft: false
tags: ["deadlock", "python", "scala"]
---
In the last 2 months, at two separate clients, I ran into the same type of deadlock, so I figured it was worth writing about to save someone else from the same fate.

**In short: When you have a pool of resources, the same thread _must not_ acquire a second resource while already holding a resource.** Otherwise, multiple threads can deadlock when they have all acquired their first resource but none can acquire their second resource. This deadlock is possible whenever `number_of_threads >= number_of_resources`. Here's some example code:

```python
pool = SomeFixedPoolOfResources(4)

def my_unsafe_function():
  resource1 = pool.request_resource()
  ...
  nested_function()
  pool.release_resource(resource1)

def nested_function():
  resource2 = pool.request_resource()
  ...
  pool.release_resource(resource2)

```

It doesn't look like the standard academic deadlock. There aren't even any locks! But consider the case where 4 threads run `my_unsafe_function` concurrently. All 4 threads successfully get `resource1`. But none of the threads can acquire `resource2` because no more resources are available, causing a deadlock. This can continue forever, or until `request_resource` times out.

## Real Life Examples
In case you think this can only happen in blogs, here's some real world code.

### Deadlock With Database Sessions
Here's some Python code using SQLAlchemy that can lead to deadlock:

```python
def start_worker(worker_id):
  # Implicitly acquires a database session from the pool
  with db_session() as session:
    worker = session.query(Worker).get(worker_id)
    bootstrap_worker(worker_id, worker.name)


# In another file, out of sight and out of mind...
def bootstrap_worker(worker_id, name):
  # only take worker_id as a parameter to make it easier for the caller
  with db_session() as session:
    worker = session.query(Worker).get(worker_id)
    worker.start_time = time.now()
    log.info('Bootstrapping {}'.format(name))
    session.commit()
```

If you have a connection pool that allows 10 concurrent sessions, this code will work great until you run `start_worker()` concurrently from 10 different threads, at which point everything will grind to a screeching halt. After that, you'll get a fun exception like

```
TimeoutError: QueuePool limit of size 5 overflow 10 reached, connection timed out, timeout 30 after 30 seconds.
```

If possible, it's best to make your session handling code fail fast if you attempt to open a nested session from within the same thread.

### Deadlock with threads
Here's another example I ran into recently, this time in Scala:

```scala
implicit val executionContext = new FixedSizeExecutionContext(...);

def computeSomething: Future[String] = {
  // `Future` will run blocking code asynchronously by using an
  // executor from the execution context.
  Future {
    // Author didn't realize there was runCommandAsync
    runCommandSync("ls")
  }
}

def runCommandSync(s: String)(implicit ex: ExecutionContext): String = {
  Await.result(runCommandAsync(s))
}

def runCommandAsync(command: String)(implicit ec: ExecutionContext) {
  // mapping the result of a future needs to
  // use a thread from the execution context
  doAsyncWork().map(result => result.toString)
}
```

This is especially nefarious in Scala because the shared resource pool, which, in this case is the `ExecutionContext` is passed implicitly. Nasty stuff. Scala provides `blocking` as a way to mark running blocking commands in Future which helps a bit, however, not all `ExecutionContexts` actually support it so it's at best a Hail Mary. Best to just be careful, especially when refactoring asynchronous code.

### In Conclusion
When writing code involving a set of shared resources, be vigilant to ensure that 1 thread never requests a second resource while already holding a resource. The problem won't show up in testing, where you typically only have 1 thread. It probably won't show up immediately on prod. It will show up the next time there is a load spike and the critical threshold of concurrent threads is reached to cause deadlock and your code will hang at the worst possible moment.

If this was interesting to you, you might also find it interesting that Postgres has a deadlock detector. It's super cool and you can read about it [here](https://github.com/postgres/postgres/tree/master/src/backend/storage/lmgr). Search for "Deadlock Detection algorithm"
