---
title: "PSA: Python Float Overflow"
date: 2018-01-24T08:56:36-08:00
tags: ["python", "language-internals"]
draft: false
---
Python 2 and Python 3 provide arbitrary precision integers. This means that you don't need to worry about overflowing or underflowing the integer datatypes, _ever_. `2**10000`? No problem. It makes it pretty convenient to write code. Floats, however, are not arbitrary precision:

```python
>>> a = 2 ** 10000
>>> a * .1
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
OverflowError: int too large to convert to float
```

Luckily, float overflow throws an exception to alert you to the problem. In Java or C/C++, overflow will silently roll over into the negatives. In Javascript, it just becomes `infinity`. It's important to watch out for this whenever your code deals with truly large numbers, especially when raising numbers to arbitrary powers. I've seen this occur in backoff methods like the following:

```python
def backoff_time(retry_count, initial_backoff, max_backoff):
  return min(max_backoff, initial_backoff*(2**retry_count))

```

It will work fine as along as `initial_backoff` is integral, but if someone wants to have an `initial_backoff < 1`, you're at risk of overflow.

Happy coding!