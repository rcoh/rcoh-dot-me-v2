---
title: "Notes on CPython List Internals"
date: 2017-12-29T13:02:41-05:00
draft: false
tags: ["language-internals", "python"]
---
As I was learning to program, Python lists seemed totally magical to me. I imagined them as being implemented by some sort of magical datastructure that was part linked-list, part array that was perfect for everything.

As I grew as an engineer, it occurred that this was unlikely. I guessed (correctly) that rather than some sort of magical implementation, it was just backed by a resizable array. I decided to read the code and find out.

One of the nice things about CPython is the readable implementation. Although the [relevant file](https://github.com/python/cpython/blob/master/Objects/listobject.c) is over 2000 lines of C, it's mostly the sorting algorithm, and boilerplate to make the functions callable from Python code.[^1] The core list operations are short and straightforward.

Here are a few interesting things I found reading [the implementation](https://github.com/python/cpython/blob/master/Objects/listobject.c). Code snippets below come from the CPython source with explanatory comments added by me.

## List Resizing
If you append to a Python list and the backing array isn't big enough, the backing array must be expanded. When this happens, the backing array is grown by approximately 12%. Personally, I had assumed this growth factor was much larger. In Java `ArrayList` grows by 50% when expanded[^2] and in Ruby, `Array` grows by 100%.[^3]

```c
// essentially, the new_allocated = new_size + new_size / 8
new_allocated = (size_t)newsize + (newsize >> 3) +
    (newsize < 9 ? 3 : 6);
```
[link to CPython reallocation code](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L59)

I did a few performance experiments -- preallocating arrays with constructs like `[None]*500` doesn't seem to make any noticeable difference. In my unscientific benchmark of appending to a list 100 million times, Python 3 was much slower than Python 2 which was much slower than Ruby, however, a lot more research is required to determine the impact (if any) of the growth factor on insert performance.

## Inserting at the beginning of the list
Inserting at the beginning of a list takes linear time -- this isn't that surprising given the rest of the implementation, but it's good to know that `some_list.insert(0,value)` is rarely a good idea. A Reddit user reminded me of [Deques](https://docs.python.org/3/library/collections.html#collections.deque) which trade constant time insert and remove from both ends in exchange for constant time indexing.


```c
// First, shift all the values after our insertion point
// over by one
for (i = n; --i >= where; )
  items[i+1] = items[i];
// Increment the number of references to v (the value we're inserting)
// for garbage collection
Py_INCREF(v);
// insert our actual item
items[where] = v;
return 0;
```
[link to CPython insertion code](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L263-L267)

## Creating List slices
Taking a slice of a list eg. `some_list[6:50]`is also a linear time operation in the size of the slice, so again, no magic. You could imagine optimizing this with some sort of copy-on-write semantics but the CPython code favors simplicity:

```c
for (i = 0; i < len; i++) {
  PyObject *v = src[i];
  Py_INCREF(v);
  dest[i] = v;
}
```
[link to CPython slice code](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L447-L451)

## Slice Assignment
You can assign to a slice! I'm sure this is commonly known among professional Python developers, but I've never run into it in several years of python programming.
I only discovered when I came across the [`list_ass_slice(...)`](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L574) function in the code. However, watch out on large lists -- [it needs to copy all items deleted from the original list which will briefly double the memory usage.](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L576)

```python
>>> a = [1,2,3,4,5]
>>> a[2:4] = 'a'
>>> a
[1, 2, 'a', 5]

>>> a = [1,2,3,4,5]
>>> a[2:4] = ['replace1', 'replace2', 'replace3']
>>> a
[1, 2, 'replace1', 'replace2', 'replace3', 5]
```
## Sorting
Python arrays are sorted with an algorithm known as "timsort". It's wildly complicated and described in detail in a [side document in the source tree](https://github.com/python/cpython/blob/master/Objects/listsort.txt). Roughly, it builds up longer and longer runs of sequential items and merges them. Unlike normal merge sort, it starts by looking for sections of the list that are already sorted (`runs` in the code). This allows it to take advantage of input data that is already partially sorted.
An interesting tidbit: For sorting small arrays (or small sections of a larger array) of up to 64 elements[^4], timsort uses "binary sort". This is essentially insertion sort, but using binary search to insert the element into the correct location. It's actually an O(n^2) algorithm! An interesting example of real-world performance winning over algorithmic complexity in practice. [link](https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L1109)

Did I miss something cool about CPython lists? Let me know in the [comments](https://news.ycombinator.com/item?id=16012418).

Thanks to Leah Alpert for providing suggestions on this post.

[^1]: I came across the [boilerplate generation code](https://docs.python.org/3/howto/clinic.html) while I was writing this post and it's super cool! A python-based C-preprocessor generates and maintains macros to do argument parsing and munging between Python and C

[^2]: [Open JDK 6](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/ArrayList.java#183): `int newCapacity = (oldCapacity * 3)/2 + 1;` [Open JDK 8](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/ArrayList.java#l240) `int newCapacity = oldCapacity + (oldCapacity >> 1);`

[^3]: https://github.com/ruby/ruby/blob/0d09ee1/array.c#L392
[^4]: https://github.com/python/cpython/blob/1fb72d2ad243c965d4432b4e93884064001a2607/Objects/listobject.c#L1923
