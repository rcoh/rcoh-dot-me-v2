---
title: "Postgres Indexes Under the Hood"
date: 2018-01-21T20:32:06-08:00
draft: true
tags: ["postgres", "databases", "algorithms"]
---
More than many commonly used primitives, I find database indexes to be among the least well-understood by many software engineers. In this post I'll explain:
- How indexing works in Postgres using B-Trees
- What B-Trees are
- Why are they a good fit for this problem 

### Indexes in Postgres
Postgres actually offers 4 different kinds of indexes for different use cases. In this post I'll be focusing on the "normal" index, the kind you get by default when you type `create index`. These indexes are implemented internally by Postgres using a datastructure called a "B-Tree". B-Trees are a generalization of binary search trees that allowing parents to have more than 2 children. In Postgres, each B-Tree node fills an 8KB "Page"[^1]. The effective branching factor varies based on the size of the data your indexing, but to ballpark it, it will be between a few hundred to a few thousand items per page. I'll talk more about why this is a a good choice later on. Like any binary search tree you would actually want to use, B-Trees are self balancing, giving them `log(n)` worst case performance.

### B-Tree Overview
Before I go further into Postgres, I'll start with a more in-depth look into B-Trees.
{{<figure src="/images/btree.svg" attr="By CyHawk - Own work based on original PNG, CC BY-SA 3.0" attrlink="https://commons.wikimedia.org/w/index.php?curid=11701365" >}}

In binary search trees, each node has two pointers, one for all lesser values, and one for all greater values. B-Trees generalize this idea -- between each node (and the edges) lies a pointer that points to all nodes within that range of values. Consider the root node in the above example, which has 3 pointers.

- The first pointer, between the edge and 7 points to values less than 7 
- The second pointer, between 7 and 16 points to values between 7 and 16
- The third and final pointer, between 16 and the end points to values greater than 16.

Retrieving an item from a B-Tree is straightforward.


[^1]: The default blocksize of 8KB is mentioned in passing [here](https://wiki.postgresql.org/wiki/FAQ#How_much_database_disk_space_is_required_to_store_data_from_a_typical_text_file.3F). In the code its a bit hard to chase down, because it's configured when Postgres is [being compiled](https://github.com/postgres/postgres/blob/2082b37/configure#L1513-L1514).