---
title: "Why Writing a Linked List in (safe) Rust is So Damned Hard" 
date: 2018-02-20T08:55:56-08:00
draft: false
tags: ["rust", "algorithms"]
---
Before I start this post, let me preface it by saying that I'm not an experienced Rustacean by any means. The excellent foks at [/r/rust](https://www.reddit.com/r/rust/) gave a lot of helpful feedback and cleared up some misconceptions I had. Futher errata and corrections are appreciated. This post is aimed at helping other fledgling rust-learners avoid my mistake. First, by helping Rust learners pick good introductory projects that will fit naturally in idiomatic rust. Second, by helping Rust learners start building Rust-friendly design intuition.

I'd heard about Rust and it's inscrutable borrow checker for years, but after reading a few blog posts about compiler error improvements, I figured it might be user-friendly enough to give it a try. I read a few chapters of the [book](https://doc.rust-lang.org/book/second-edition/) and then set about my first project: I wanted to build an [x-fast trie](https://en.wikipedia.org/wiki/X-fast_trie). Specifically, I was curious about the its performance in practice (hence the choice of a low-level language). I'll save the details of what is a really cool data structure for a later post, and cut to the chase: an x-fast trie is a trie with values at the leaves. To enable `\(O(1)\)` predecessor and successor searches once you have a leaf node, **the values are stored in a doubly linked list**. 

I was coding along fairly successfully with a few small hiccups until I hit the doubly linked list. But when I started implementing it, the compiler let me know, in no uncertain terms that what I was doing wasn't ok. I tried and tried poking and prodding in different ways before I realized that I was trying to make Rust do something it fundamentally would not let me do.

To explain why, consider a simple Node class for a doubly-linked list:
```rust
pub struct Node {
    value: u64,
    next: Option<Box<Node>>,
    prev: Option<Box<Node>>,
}
```

Each node has a 64-bit value, and optional `next` and `prev` nodes. Before I get into the parts of Rust that make this impossible, let me talk about the parts that make this awesome. It just turns out the awesome parts are impossible to provide in this case. 

- `next` and `prev` must be `Optional` because there is no such thing as a null pointer in Rust. As the witnesser of many a segfault, this is awesome.
- `next` and `prev` recursively refer to `Node`, so we can't put them directly into the struct.[^1] [`Box`](https://doc.rust-lang.org/std/boxed/struct.Box.html), the simplest of Rust's "smart pointers" will heap allocate it's contents when `Box::new()` is called.

So far so good. The compiler happily accepts our `struct`. The problems start if we try to actually use it.

### Rust and Ownership
To understand intuitively why this won't work, we have to understand ownership. Rust's safety features don't come for free, and ownership is one of the costs. Rust has 3 simple rules of ownership with complex and sweeping implications:

1. Each value in Rust has a variable that’s called its owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value will be dropped.

I won't attempt to describe these implications here -- there are too many and I don't understand most of them. The one that is important to note, however, is that if you pass a value directly to a function, the function (and the return value of the function, if there is one) now owns your value. You can't use it anymore.

Our first try at implementing this will hit problems immediately.
```rust
// must be mut so we can modify it
let mut head = Node {
    value: 5,
    next: None,
    prev: None,
};
let next = Node {
    value: 6,
    next: None,
                 // next takes ownership of head!!!
    prev: Some(Box::new(head)),
};
// I actually don't understand why the line below compiles. 
// Since `head` was moved into the box, I'm not sure why I can mutate it.
head.next = Some(Box::new(next));
```
So far so good! But what if we want to print head:
```rust
println!("{:?}", head)
```

```rust
  --> src/lib.rs:27:26
   |
24 |             prev: Some(Box::new(head)),
   |                                        ---- value moved here
...
27 |         println!("{:?}", head);
   |                          ^^^^ value used here after move
   |
   = note: move occurs because `head` has type `ll::Node`, 
     which does not implement the `Copy` trait
```

Of course! We can't use head anymore because it was moved into prev. What about printing `next`?
```rust
error[E0382]: use of moved value: `next`
  --> src/lib.rs:27:26
   |
26 |         head.next = Some(Box::new(next));
   |                                        ---- value moved here
27 |         println!("{:?}", next);
   |                          ^^^^ value used here after move
   |
   = note: move occurs because `next` has type `ll::Node`, 
     which does not implement the `Copy` trait
```
Same problem! When we set `head.next = next`, head took ownership, and we don't have it anymore. You can try to sidestep this problem in a couple of ways:

- Making `Node` keep borrowed boxes instead of valued boxes:

    ```rust
    Node {
        // ...
        next: Option<&Box<Node>>
    }
    ```

    This helps until you want to mutate them. Borrowing is like a read-write lock for mutation. You can borrow immutably in multiple places, but if you want to borrow mutably, all your immutable borrowers must return the item. This isn't going to work since our list is composed of immutable borrows.


- Rust has `Rc`, a ref-counted pointer which seems like it could do the trick, but it prohibits mutation. This is obvious in hindsight: how could Rust let you have multiple references to something but also let you mutate them?

There are 3 solutions I'm aware of:

- Use `RefCell`, a runtime-checked borrow system ~~(which still only works on nightly if you want to mutate)~~. `RefCell::borrow_mut` allows mutable borrowing of the cell's contents. 
- Eschew safe Rust altogether and wade into the magical swamp of unsafe Rust. This isn't nearly as big a deal as I initially thought -- even things like `Vec<>` are built on unsafe Rust.[^2] While it's not ideal, unsafe Rust is really just "normal mode" in most other programming languages. 
- Keep pointers as indices into a `Vec<>` instead of pointers, using something like the [indextree crate](https://github.com/saschagrunert/indextree)

If you want to follow someones detailed quest to write linked lists I highly recommend [Learning Rust With Entirely Too Many Linked Lists](http://cglab.ca/~abeinges/blah/too-many-lists/book/).

### Conclusion and Takeaways

In hindsight and with a deeper understanding, it's not surprising why a doubly linked list is so problematic. Each variable can only have _1_ owner. If `prev` and `next` both hold pointers to a middle node, which is the owner? They can't both be the owner. If one is the owner, another can borrow. But Rust won't let you mutate while someone else is borrowing, so neither could actually modify the list! In general, if you have loops in your object graph you're out of luck.

I find a bit of solace in the fact that implementing a data structure like this in a non-garbage collected language _without_ Rust is also quite tricky; you need to carefully keep track of when a node has no more references and can be freed. In the end, I gave up on Rust for this project and implemented it in Go instead. Never has coding in Go felt so effortless ;-). In the mean time, I'm getting started on my next learn-rust project, a port of my [Sumoshell](https://github.com/SumoLogic/sumoshell) tool. Since I ran into tons of race conditions in the initial implementation, I'm hopeful that Rust will help me get it right the first time.

Did I get something totally wrong? Please let me know, or just [send me a pull request!](https://github.com/rcoh/rcoh-dot-me-v2/blob/master/content/posts/rust-linked-list-basically-impossible.md)

***
{{% subscribe %}}

[^1]: To stack allocate it, Rust would need to know exactly how much space it would take. Since it's a recursive datastructure, there is no way to know!
[^2]: Building on an unsafe foundation: https://www.youtube.com/watch?v=rTo2u13lVcQ
