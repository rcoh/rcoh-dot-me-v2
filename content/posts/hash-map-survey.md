---
title: "A Survey of Hash Map Implementations in Popular Languages"
date: 2017-12-30T17:39:23-05:00
draft: true
---
Few data-structures are more ubiquitous in real-world development than the hash table. Nearly every major programming features an implementation in its standard library or built into the runtime. Yet, there is no conclusive best strategy to implement one and the major programming languages diverge widely in their implementations! I did a survey of the
Hash map implementations in Go, Python, Ruby, Java, and Scala to compare and contrast how they were implemented.

**Note: The rest of this post assumes a working knowledge of how hash tables work along with the most common schemes for implementing them.** If you need a refresher, [Wikipedia](https://en.wikipedia.org/wiki/Hash_table) provides a fairly readable explanation. Beyond the basics, the sections on [chaining](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining) and [open addressing](https://en.wikipedia.org/wiki/Hash_table#Open_addressing) should provide sufficient background.

## Summary
Though all implementations differed significantly, certain commonalities remained:

- Open addressing with double hashing (Python & Ruby) and chaining (Java, Scala, Go) were represented about equally
- No "exotic" implementations like cuckoo hashing, etc in the surveyed languages although most implementations included varying degrees of optimizations that complicated the code significantly.
- All languages attempt to add entropy to the hash code by mixing the lower and higher order bits at some point in the process. Interestingly, all the languages featured contained a primitive type with a low-entropy hash function that lead to this being a necessity.
- All grow by 2x. Most guarantee that the size is always a power of 2.

On to the details:

### Python (CPython)
[Source](https://github.com/python/cpython/blob/master/Objects/dictobject.c)
[Implementers Notes](https://github.com/python/cpython/blob/master/Objects/dictnotes.txt)

**Scheme:** Open Addressing with custom sequence. The sequence is essentially `j = ((5*j) + 1) mod TABLE_SIZE` but during the first few locations searched, the original hash code is progressively shifted and mixed in to increase key entropy. Not including the perturbation shift, this scheme expands to $$ \sum\_{n=0}^{i} 5^i \mod T $$

**Growth rate:** At least 2x, and size is always a power of 2. In the case where there a no deletions, it will double in size. Since deletions are tombstoned, it's possible that we have no usable space even though the we haven't hit the load factor. The growth rate is `used*2+capacity/2` to account for this.[^3]

**Load factor:** 2/3

Other bits of note:

 - Although Ruby uses a different perturbation strategy, they both use the same underlying probing scheme of `next = (prev * 5) + 1 mod TABLE_SIZE)`
 - The implementation special cases maps where the codes are exclusively unicode strings[^2]. The motivation for this comes from the fact that so many of the Python internals rely on dictionaries with unicode keys (eg. looking up local variables).[^5]
   - With only string keys, a separate key array, `ma_keys` is stored along with `ma_values`
   - With non string keys: `ma_values` is null and objects within `ma_keys` contain a pointer to `me_value`
 - Designed to work with badly behaved hash code functions because for integers in Python `hash(i) == i`
 - Tuned empirically with lots of magic numbers[^4]
 - Items that are removed must be tombstoned (marked `IDEX_DUMMY`). In open addressing that is the only way to signal that you need to look further down the chain if items have been deleted. Python uses a separate array mapping containing indices into the array of key-value pairs. This allows them to dodge a complexity in the Ruby hash table where they must ["escape"](https://github.com/ruby/ruby/blob/trunk/st.c#L310-L312) user provided hash codes that happen to collide with the chosen `DUMMY` constant.
 - The growth rate changed from `used*4` to `used*2` in `3.3.0`.[^3]

## Ruby
[Source](https://github.com/ruby/ruby/blob/trunk/st.c) [Implementers Notes](https://github.com/ruby/ruby/blob/trunk/st.c#L1-L96)

**Scheme:** Open addressing using `j = ((5*j) + 1) mod TABLE_SIZE` with perturbation. This is the same general structure as Python but they use a different perturbation strategy.

**Growth rate:** 2x. The number of slots is always a power of 2.

**Load factor:** .5

Other bits of note:

- Old implementation used chaining. New implementation is reportedly 40% (!) faster[^1]
- Entries array (for fast iteration) is split from bins array for hash lookup
- Very small arrays have no bins and use linear scanning instead.

## Java

**Scheme:** Chaining, with linked lists that convert into TreeMaps when the length of lists > 8. This conversion is most helpful if either:

  - K implements `Comparable<>`
  - The hash codes collide mod the table size but are not equal.

**Growth rate:** 2x. The number of slots is always a power of 2

**Load factor: 0.75**

Other bits of note:

- Like Python, Java does some spreading to ameliorate the fact that it only reads the low bits when the hashtable is small:
  `return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);`
- When resizing, elements go in one of two buckets, `k` or `k+oldSize`. This is a
  convenience of factor-of-two resizing.
- The code is really hard to follow, primarily due to the fact that chains can flip between trees and linked lists.

## Scala Immutable Hash Map

[Source](https://github.com/scala/scala/blob/2.12.x/src/library/scala/collection/immutable/HashMap.scala)
Most hash map usage in Scala uses the immutable hash map, so I'll discuss that first.

**Scheme:** Hash Trie with chaining. A hash trie is a recursive data structure (so it's hash tries all the way down). The [Scala doucmentation](http://docs.scala-lang.org/overviews/collections/concrete-immutable-collection-classes.html#hash-tries) provides a decent explanation. For more depth, [Phil Bagwell's paper](https://infoscience.epfl.ch/record/64398/files/idealhashtrees.pdf) is an excellent resource. I'll provide a brief summary:

For maps of size 0 to 4, it uses hardcoded maps. For larger maps, it uses a HashTrie. Each level of the hash trie considers some subset of bits of the hash code. When inserting or retrieving, the implementation recurses into the branch of the trie matching the bits, using the next subset of bits as an argument. Since
The Scala hash trie implementation has a branching factor of 32, each level considers 5 bits of the hash code (2^5 = 32).  Since hash codes in Java/Scala are 32-bit integers, this means that if all hash codes are unique, the hash trie will store 2^32 elements without collision.

If the hash codes are identical, chaining is used, wrapped within the [`HashMapCollision`](https://github.com/scala/scala/blob/2.12.x/src/library/scala/collection/immutable/HashMap.scala#L239) data structure.

Scala also provides a mutable hash map. Since it lacks the optimizations of the other languages
I looked at, it was the only one that was straightforward.

## Scala Mutable Hash Map
[Source](https://github.com/scala/scala/blob/2.12.x/src/library/scala/collection/mutable/HashTable.scala)

**Scheme:** Chaining with linked lists

**Growth rate:** 2x

**Load factor:** 0.75

Bits of note:

- This is what I naively expected a hash map implementation to look like. It's under 500 lines, and the core is under 100 lines. It's straightforward, without complications and easy to read.
- Like many other implementations, it attempts to increase the entropy of the incoming hash codes with some mixing:

```scala
   var h: Int = hcode + ~(hcode << 9)
   h = h ^ (h >>> 14)
   h = h + (h << 4)
   h ^ (h >>> 10)
```

## Golang
[Source](https://github.com/golang/go/blob/master/src/runtime/hashmap.go)

**Scheme:** Chaining, with some optimizations. The chains are composed of buckets. Each bucket has 8 slots. Once all 8 slots are consumed, an overflow bucket is chained to the first bucket. Storing 8 key-value pairs in contiguous reduces the amount of memory accesses and memory allocations when reading and writing to the map.

**Growth Rate:** 2x. When a lot of deletions occur, a map of the same size is allocated to garbage collect the unused buckets.

**Load factor**: 6.5! Not 6.5%, but rather 6.5x. This means that on average, the hash map will resize when each bucket has 6.5 items. This is a major contrast with the other hash map implementations which all use a load factor less than 1.

Bits of note:

  - In all other implementations, the work of copying the elements from the old array to the new array is performed during the single insert that triggered the resize. In Golang, the resize operations of moving to the new map are done incrementally as more keys are added!
    For each new element that's added / updated, 2 keys are moved from the old map to the new map, ensuring that no single write incurs `O(n)` performance. Once all the keys have been [`evacuated`](https://github.com/golang/go/blob/fbfc203/src/runtime/hashmap.go#L1006) from the old array, the old array can be deallocated.

  - 2 conditions can trigger resizing:

    1. The number of elements >= 6.5x the size of the array, the new array is the same size as the old array.
    2. The number of buckets is too large.

    In the case of #2, the newly allocated array is the same size as the old array. This seeming nonsensical behavior comes from this [commit](https://github.com/golang/go/commit/9980b70cb460f27907a003674ab1b9bea24a847c). In the case of deletions, allocating and slowly migrating to a new array means that we'll garbage collect the old buckets instead of slowly leaking them. They chose this approach to ensure that iterators continued to work properly.

### Wrap Up

I find it fascinating that there are so many different implementations for hash tables used in the production languages. I found Ruby's shift from
chaining to open addressing especially interesting since it apparently improved on benchmarks quite a bit. I would be interesting to write an open-addressed hashtable for Java or Go and compare performance.

Did I miss your favorite language? Let me know in the comments or by email, `*+hashtables@*.me` where `* = rcoh`.

[^1]: https://github.com/ruby/ruby/blob/fc939f6/st.c#L93-L94
[^2]: https://github.com/python/cpython/blob/60c3d35/Objects/dictobject.c#L54-L63
[^3]: https://github.com/python/cpython/blob/60c3d35/Objects/dictobject.c#L398-L408
[^4]: https://github.com/python/cpython/blob/master/Objects/dictnotes.txt#L70
[^5]: Perhaps surprisingly starting the Python interpreter and running a couple of non-dictionary related commands incurs about 100 dictionary lookups.
