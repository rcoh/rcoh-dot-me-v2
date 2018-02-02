---
title: "My Favorite Algorithm: Cache Oblivious Data structures"
date: 2018-01-29T22:44:14-08:00
draft: true
tags: ["algorithms"]
---
If you read my recent [post](/posts/postgres-indexes-under-the-hood) about Postgres you may have noted that Postgres operates primarily with fixed-size blocks of memory called "pages." By default, Postgres pages are 8KB. This number is tuned to match operating system page sizes which are tuned to match hardware cache sizes. If you were to run Postgres on hardware with different hardware than Postgres was tuned for, you may be able to pick a better page size.[^1]

Along these lines, today's post explores the question:

> Can we design data structures that perform optimally regardless of underlying cache sizes?

If we could do this, our code would optimally utilize all cache layers without tuning. I'll give a spoiler here: We can, both in theory and in practice. In the literature, algorithms and data structures are referred to as *cache oblivious*. If you want to know more about anything I'm talking about here, Erik Demaine's [2002 survey paper](http://erikdemaine.org/papers/BRICS2002/paper.pdf) covers the subject with far more rigor and detail than I'll be able to accomplish in this post. 

## The Memory Model
In standard algorithmic analysis, we implicitly assume that reading from memory is directly proportional to amount of bytes we read. Read one byte, pay for one byte. But the real world doesn't work this way: the layers of software and hardware below your programming language don't work in individual bytes; they work in blocks.[^2] If you want to read one byte, you need to pay for the whole block. If you read more bytes from that block, those reads are nearly free. To analyze our algorithms, we'll develop a model that's closer to how things work in the real world.

In developing algorithms that perform optimally regardless of cache size, we'll assume that we have 2 areas of memory: the small fast layer, and the big slow layer. Memory can only be moved from the slow layer to the fast layer in fixed size blocks of size `\(B\)`. The slow layer is infinitely sized, and the fast layer has size `\(M\)`. We'll assume that the fast layer is a fully-associative cache (meaning it acts like a hash table; we can put any block anywhere).[^3] There are more assumptions to make the math all work that I've left out for simplicity.[^4] When we analyze our algorithms we'll analyze them in terms of `\(M\)`, the size of our cache, and `\(B\)`, the size of blocks in the cache. Our analyses must hold for any values of `\(M\)` and `\(B\)`. Futhermore, the unit of work we'll be analyzing is loading a block from slow memory into fast memory -- other work is considered much faster and not included.

## Warm up: Scanning an array

Let's start with a straightforward analysis: scanning an array to find the maximum element. Our algorithm will be to store our array in a block of contiguous memory and iterate through the elements in sequence. You'll note that the algorithm bears no mention of `\(B\)` or `\(M\)`. Our goal here is to show that this algorithm has the same big-O complexity in terms of `\(B\)`, `\(M\)`, and `\(N\)`, as an optimal algorithm that knows those sizes a priori.

No matter how the blocks are split in the array, we'll need to read at least `\(\frac{N}{B}\)` blocks. In the worst case, both the first block and last block will contain exactly 1 element. This leads to `\(\left \lfloor{\frac{N}{B}}\right \rfloor +2\)` as the minimum number of blocks read by this algorithm, or, `\(O(\left \lceil{\frac{N}{B}}\right \rceil)\)`. If we knew `\(M\)` and `\(B\)` ahead of time, we couldn't possibly do better.

This example was a bit anticlimactic: The obvious algorithm is already cache oblivious! Let's do something a bit more interesting.

## Getting serious: Static Binary Trees

For a more interesting case, consider a static binary search tree. Static binary search trees aren't as useful as dynamic binary search trees, but they form the basais of other cache-oblivious datastructures like B-Trees.

### Deriving a Lower Bound
The salient issue in implementing a static BST will be how we layout the nodes in memory. Our task will be to layout the nodes of the tree in memory such that we access as few blocks as possible. The minimum possible number of blocks that must be accessed `\(log_bN+O(1)\)`[^5] We assume that every byte of every block we read is useful, and apply an information-theory based approach which I won't include here. Obviously, we also need our static tree to require no more than `\(O(\log n)\)` comparisons as well.



### Data Layout
In designing cache-oblivious data structures and algorithms, a divide-and-conquer strategy frequently bears fruit. This stems from the fact that divide an conquer algorithms naturally break the work in subproblems of increasingly smaller sizes -- one of those sizes will be close to `\(B\)` and constrain the number of blocks you need to read. We'll take a similar approach in laying our our cache-oblivious static BST.

To start, split the tree at half height. If the height of the tree is `\(\log(n)\)`, the top section will contain `\(2^{log_2(n)/2}=\sqrt{n}\)` nodes. Each of the child nodes will also contain `\(\approx \sqrt{n}\)` elements. This splitting process is recursive -- each section of the tree is split until there are no nodes left to split.

The key property of our layout strategy is that all "blocks" are laid out contiguously in memory. Critically, all "super blocks" are also contiguous. This means that no matter what `\(B\)` is, we'll always be reading useful data.

![Van Emde Boas Layout](/images/van-layout.svg)

The figure above demonstrates a cache-oblivious memory layout for a binary search tree with 5 levels. The blocks are circled in green and the super-blocks are circled in orange. I've omitted the right branch to remove some visual clutter.

It's worth noting that the order the data is layed out within the blocks is not important. The key detail is that all blocks, super-blocks, super-super-blocks etc. lay in contiguous memory.

### Proof Sketch
I'll sketch a proof for why this layout meets the lower bound above. I will refer to system blocks as "Blocks" and tree sections as "chunks". Suppose the system operates with blocks of size `\(B\)`. It follows then that there is a chunk size within the tree such that `\(C \leq B\)`. The height of each chunk is `\(\log_2C \geq 1/2\log_{2}B\)`. Since the height of the tree is `\(\log_2N\)`, we must therefore read at most 

`$$\frac{log_2(N)}{1/2 \log_2(B_system)}=2 \log_BN$$`

Since each chunk could fall in two blocks, we're left with a final result of `\(4 \log_BN \)`.


**Maybe do a quick performance test of this layout vs. another and see if this is faster?**

## Cache Oblivious Data structures IRL
A static search tree is not especially useful, except as a building block to other data structures. Specifically, they allow building a cache oblivious B-Tree! Cache oblivious B-Trees are actually useful in practice. **a few more sentences here**


I won't go into detail here, but you can read about them in [Demaine's survey paper](http://erikdemaine.org/papers/BRICS2002/paper.pdf), section 5.1. 

[^1]: One example is running Postgres on SSDs. For various reasons, SSDs to better with smaller block sizes: https://blog.pgaddict.com/posts/postgresql-on-ssd-4kb-or-8kB-pages
[^2]: As you move from CPU caches (L1, L2, L3), to operating system level page caches, these blocks go from bytes in the L1 cache to kilobytes at the operating system level. 
[^3]: This isn't how real CPU caches work. They are usually 2-8x associative. If a cache is N-way associative, then we have N different places in our cache this block is allowed to go. Fully associative means blocks can go anywhere, which just isn't real. But we've got to start somewhere!
[^4]: Specifically, block replacements are optimal (we always evict the block that will be used as far as possible in the future), and the cache is "tall" (much larger than the size of the blocks). The paper contains reasonable justifications for why these are OK.
[^5]: The proof is based the number of bits of information required to locate an element divided by the number of bits yielded per block read. See section 4.1 of [the paper](http://erikdemaine.org/papers/BRICS2002/paper.pdf) for details.