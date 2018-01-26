---
title: "Postgres Indexes Under the Hood"
date: 2018-01-26T20:32:06-08:00
draft: true
tags: ["postgres", "databases", "algorithms"]
---
Many software engineers use database indexes every day, but few of us really understand how they work. In this post I'll explain:

- How indexing works in Postgres using B-Trees
- What B-Trees are
- Why they are a good fit for this problem 

### Indexes in Postgres
Postgres actually offers 4 different kinds of indexes for different use cases. In this post I'll be focusing on the "normal" index, the kind you get by default when you type `create index`. These indexes are implemented internally by Postgres using a data structure called a B-Tree. B-Trees are a generalization of binary search trees that allow nodes to have more than 2 children. The, number of children, or branching factor, per node is up to the implementer of the data structure. Databases typically have branching factors in the low thousands.  Like any binary search tree you would actually want to use, B-Trees are self balancing, giving them `log(n)` worst case performance.

### B-Tree Overview
Before I go further into Postgres, I'll start with a more in-depth look into B-Trees.
{{<figure src="/images/btree.svg" attr="By CyHawk - Own work based on original PNG, CC BY-SA 3.0" attrlink="https://commons.wikimedia.org/w/index.php?curid=11701365" >}}

In binary search trees, each node has two pointers, one for all lesser values, and one for all greater values. B-Trees generalize this idea -- between each node (and the edges) lies a pointer that points to all nodes within that range of values. Consider the root node in the above example, which has 3 pointers.

- The first pointer, between the edge and 7, points to values less than 7 
- The second pointer, between 7 and 16, points to values between 7 and 16
- The third and final pointer, between 16 and the end, points to values greater than 16.

Looking up items is straightforward and involves recursively following the correct child pointer until you hit your value or a leaf. Most implementations use binary search to choose the correct child although it isn't necessary in terms of big O (because the node sizes are constant), and it isn't always beneficial in practice (unless the nodes are huge).

I'll spare you the fairly complex description of how inserting and deletions work. It's straightforward until you need to split nodes, and then it gets complicated in a hurry. The really cool part is that splitting actually inserts the median of the child into the parent. This means that splitting also rebalances the tree, making the overall implementation simpler than something like a red-black tree with dozens of different types of rotations. Consider inserting a sorted sequence of numbers into a B-Tree with a branching factor of 5: 

![B-Tree rebalancing](/images/btree-balance.svg)

In the example above, inserting `5` causes the B-Tree to split. The split moves the median of the node into the parent (in this case there is no parent so it creates one). Selecting the median guarantees two equal sized branches.

There are two important takeaways from this section:

1. B-Trees have identical big-O performance to binary search trees with log(n) search, insertion, and deletion.
2. B-Trees are extremely shallow data structures. Because the branching factor is typically in the thousands, they can store millions of elements in only 2-3 layers. For the database, this means only 2-3 disk seeks are required to find any given item, greatly improving performance over the dozens of seeks required for a comparable on-disk binary search tree or similar data structure. 

### B-Trees in Postgres

Postgres B-Trees use 1 [page](https://en.wikipedia.org/wiki/Page_(computer_memory)) of memory per node. Postgres pages are configurable with a default size 8KB.[^1] By working in entire memory pages at a time, Postgres is able to allocate memory more efficiently from the operating system. Throughout this section I'll be using *page* and *node* interchangeably to refer to a single node in the B-Tree.

The use of fixed-size pages means the the branching factor depends on the size of each individual tuple stored in the index. Typical branching factors will be between a few hundred to a few thousand items per page. Most theoretical B-Trees assume a fixed number of keys per node. However, Postgres uses fixed-size pages and size of the keys are dependent on the items that are being indexed. A couple interesting bits fall out of this:

1. Smaller index keys will perform better than larger index keys by a small margin since they will allow a higher branching factor
2. Normally, a B-Tree is split based on the number of items in each bucket, however, the Postgres B-Trees are split to give an equal number of bytes in the resulting buckets.


Like many inventions of theoretical computer science[^3] B-Trees require some modifications to be efficient in practice. Postgres uses a flavor of B-Trees outlined in [Lehman and Yao's 1981 paper](https://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf). From the [Postgres B-Tree readme](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/README):

> Compared to a classic B-tree, L&Y adds a right-link pointer to each page,
to the page's right sibling.  It also adds a "high key" to each page, which
is an upper bound on the keys that are allowed on that page.  These two
additions make it possible detect a concurrent page split, which allows the
tree to be searched without holding any read locks (except to keep a single
page from being modified while reading it).

In a traditional B-Tree implementation, there is a race between getting the pointer to a child based on it's expected range and the child being split by another process. A demonstrative figure from Lehman and Yao's paper:

![B-Tree concurrent modification](/images/btree-conc1.png)

```python
# This is not how it works in postgres. This demonstrates the problem:
"Thread A, searching for 15"   |   "Thread b, inserting 9"
                               |   node2 = read(x); 
node = read(x);                |
"Examine c, 15 lies in y"      |   "Examine a, 9 belongs in y"
                               |   node2 = y; 
                               |   # 9 does not fit in y
                               |   # Split y into (8,9,10) and (12,15)
                               |   y = (8,9,10); y_prime = (12,15)
                               |   x.add_pointer(y_prime)
                               |   
"y now points to (8,9,10)!"    |
node = read(y)                 |
"15 not found!"                |
```

The "high-key" pointer allows readers to detect that this split has occurred: If you're looking for a value greater than the high key, you must follow the right-link! The right link allows the reader to traverse directly to the resultant split node where the new key lives.

This means that readers and writers can operate concurrently, _except_ when the writer is modifying the same node that the reader wants to read. In practice, it means that as long as you aren't reading and writing from keys that fall within the same page, you shouldn't run into concurrency issues. It is probably worth keeping in mind that a key that is being constantly modified will slow reads and writes for keys that fall into the same node. However, since these locks are held for a much shorter duration than the user-visible Postgres locking constructs, it's probably not worth your time worrying about it unless you see it in practice.

Multi-transaction concurrency is handled by B-Tree indexes by actually keeping multiple copies of the data in the index. Each entry is associated with it's transaction id. For example, in the case of "read committed" consistency, when a transaction is searching for data in the index, it is looking for index entries with a value less than or equal the maximum committed transaction at the time the transaction started.[^2]

### Alternative Indexing Strategies
A B-Tree isn't the only way to implement a database index. Databases like CockroachDB that use a sorted key-value store as a storage engine usually opt to just store a copy of the table sorted by the search key instead of the primary key. B-Trees, however, are the most popular, by far. In my brief survey of a few mainstream databases, all used B-Trees. Do you know of a mainstream database using an alternate strategy? Please tell me so I can include it here!


### In Conclusion
Hopefully this post helps to demystify what's going on under the hood when you create a database index! If you ever find yourself spelunking through the Postgres source like I did for this post, [the source code browser](https://doxygen.postgresql.org/) is incredibly helpful. It allows you to jump to the definitions of methods and variables, as if you were in an IDE.

***

Want to get emailed about new blog posts? I post about once a week on topics like [databases](/tags/databases), [language internals](/tags/language-internals), and [algorithms](/tags/algorithms). [Subscribe!](http://eepurl.com/dhk7F1)

*** 

Do you want to hire me? I'm available for engagements from 1 week to a few months. Email me at *@*.me, where * is rcoh.

[^1]: The default blocksize of 8KB is mentioned in passing [here](https://wiki.postgresql.org/wiki/FAQ#How_much_database_disk_space_is_required_to_store_data_from_a_typical_text_file.3F). In the code its a bit hard to chase down because it's configured when Postgres is [being compiled](https://github.com/postgres/postgres/blob/2082b37/configure#L1513-L1514).
[^3]: I'm looking at you, [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))
[^2]: https://momjian.us/main/writings/pgsql/internalpics.pdf, slide 59-62