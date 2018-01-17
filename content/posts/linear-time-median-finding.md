---
title: "My Favorite Algorithm: Linear Time Median Finding"
date: 2018-01-15T22:07:18-08:00
draft: true
---
**todo** intro

Finding the median in a list seems like a trivial problem, but doing in provably linear time turns out to be tricky. In this post I'm going to walk through one of my favorite algorithms, the median-of-medians approach to find the median of a list in deterministic O(n) time. Along with a discussion and proof of algorithm, I'll include code, and some real-world performance comparisons to other algorithms. Although proving that this algorithm runs in linear time is a bit tricky, this post is targeted at readers with a basic level of algorithmic analysis.

To best understand this algorithm we'll start with the simplest (and slowest) algorithms for finding medians.

### Finding the median in O(nlogn)

The most straightforward way to find the median is in `O(n logn)`. Simply sort the list, then pick the appropriate element:

```python
def median(l):
    l = sorted(l)
    if len(l) % 2 == 1:
        return l[len(l)/2+1]
    else:
        return l[len(l)/2]/2 + l[len(l)/2+1] 
```

The simplicity of this method comes at a cost -- since sorting is O(nlogn), we'll be able to do much better.

### Finding the median in average O(n)

Our next step will be to _usually_ find the median within linear time, assuming we don't get unlucky. This algorithm called "quick-select" works by choosing a pivot (often at random). Once we have the pivot, we split our list into elements greater than our pivot and elements less than our pivot. Based on the index that we're selecting, we know which section our target is in and what subindex within that target.

Below, is an example of the algorithm in practice:
```
[9,1,0,2,3,4,6,8,7,10,5]
len(11), looking for the 6th smallest element 
Pick index 3 at randomm, v=2 
[1,0], 2, [3,4,6,8,7,10,5]
We want the 6th element, so we know it's the 3rd smallest element in the right array

[3,4,6,8,7,10,5]
len(7), looking for 3rd smallest element
Pick index 2 at random, v=6
[3,4,5] 6 [7,10]
We want the 3rd smallest element, so we know it's the 3rd smallest element in the left array

[3,4,5]
len(3) looking for 3rd smallest element
Pick index 1 at random, v=4
[3] 4 [5]
Third element is 5! We're done
```

To find the median with quick-select, we'll extract quick-select as a separate function that is used by our function to find the median.

```python
import random
def quickselect_median(l):
    if len(l) % 2 == 1:
        return quickselect(l, len(l)/2)
    else:
        # For even length lists, just select the two correct elements
        return quickselect(l, len(l)/2-1)*.5  + quickselect(l, len(l)/2)/2.0 


def quickselect(l, k):
    """
    Select the kth smallest element in `l` (0 based)
    """
    # If there is only one element in the list, then it must be our element.
    if len(l) == 1:
        # If we coded quickselect properly, we will only be in
        # this case when we're looking for the 0th index
        assert k == 0
        return l[0]

    # Choose a pivot
    pivot = random.choice(l)
    
    # Segment on the pivot. Note that the list of lows includes
    # any values equal to the pivot
    lows = [el for el in l if el <= pivot]
    highs = [el for el in l if el > pivot]

    # Recurse in the appropriate list
    if k < len(lows):
        return quickselect(lows, k)
    else:
        return quickselect(highs, k - len(lows))
```

Quick-select is great in practice. Why is this linear time? On average, the pivot will split the list into 2 approximately equal-sized pieces. Each subsequent recursion will operate on 1/2 the data of the previous step. 

$$C=n+n/2+n/4+n/8+...=2n=O(n)$$

How does this math work out? Consider a pie with area `2n`. First, eat half the pie. That is a slice of size `n`. Then eat half of the remaining pie. This is a slice of size `n/2`. If you continue taking slices of half the remaining pie, you'll never run out of pie, but you'll eventually consume a total of `2n`.[^1] Dropping the constant factor, we can see that on average, quick-select is an `O(n)` algorithm.

But what if we aren't happy to be average?

### Deterministic O(n)

In the section above, I described quick-select, an algorithm with _average_ `O(n)` performance. Average here means that _on average_ the algorithm will run in `O(n)`. Technically, you could get extremely unlucky; at each step, your pivot could only split 1 item off the main list. In this case, you'd actually have `O(n^2)` performance.

With that in mind, what follows is an algorithm for _picking pivots_. Our goal will be to pick a pivot in linear time that removes enough elements in the worst case to satisfy `O(n)` performance. Rather than walk through the algorithm in text, I'll walk through it in code below:

```python
def pick_pivot(l):
    """
    Pick a good pivot within l
    """
    assert len(l) > 0

    # If there are < 5 items, just return the median
    if len(l) < 5:
        return nlogn_median(l)

    # First, we'll split `l` into groups of 5 items. O(n)
    chunks = chunked(l, 5)

    # For simplicity, we can drop any chunks that aren't full. O(n)
    full_chunks = [chunk for chunk in chunks if len(chunk) == 5]


    # Next, we sort each group. Each group is a fixed length, so each sort
    # takes constant time. Since we have n/5 groups, this operation is also O(n)
    sorted_groups = [sorted(chunk) for chunk in full_chunks]

    # The median of each chunk is at index 2
    medians = [chunk[2] for chunk in sorted_groups]

    # It's a bit circular, but I'm about to prove that finding
    # the median of a list can be done in provably O(n).
    # Finding the median of list list of length n/5 is therefore also O(n)
    median_of_medians = quickselect_median(medians, pick_pivot)
    return median_of_medians
```

Let's prove why the median of median is a good pivot. To help, consider this visualization of our pivot-selection algorithm:

![Pivot selection visualization](/images/median-of-medians.svg)

The red oval surrounds medians of the chunks, and the green circle surrounds the median of medians. For the purpose of demonstration, the chunks are sorted by their median, although the algorithm doesn't need to do this.


[^1] For a more formal treatment, it's hard to do much better than the [Master Theorem](https://en.wikipedia.org/wiki/Master_theorem_(analysis_of_algorithms)). For direct math explanation, see [here](https://en.wikipedia.org/wiki/1/2_%2B_1/4_%2B_1/8_%2B_1/16_%2B_%E2%8B%AF).