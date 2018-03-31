---
title: "Things I learned writing my first few thousand lines of Rust"
date: 2018-03-24T12:56:00-07:00
draft: true
tags: ["rust"]
---
To say my [first foray](/posts/rust-linked-list-basically-impossible) into Rust was a frustrating struggle would be a mild understatement. I picked a terrible first project that left me neck deep in Rust's trickiest areas right off the bat. I was excited to try again. A few years ago I wrote [Sumoshell](https://github.com/SumoLogic/sumoshell), a CLI-App for log analysis. I'd wanted to improve it for a while, so porting it to Rust seemed like a nice way to kill two birds with one stone. Like Go, Rust can also compile statically linked binaries which makes it ideal for distributing CLI apps; indeed CLI App support is a [priority](https://internals.rust-lang.org/t/announcing-the-cli-working-group/6872) for Rust. Before I get into the rest of the post, [here's the end result](https://github.com/rcoh/angle-grinder).

![Angle grinder demo](/images/agrind-demo.gif)


With a few small hiccups, my second try at learning Rust was really a joy. It combines the expressiveness and type system I love from Scala with the small memory footprint and performance you get from compiling directly to assembly.[^1] More than other languages I've used, I saw a lot of pragmatism in the design of the language and the standard library. I should caveat this with a note that I haven't tried to write macros or use `Futures` yet, [two major pain-points](https://brandur.org/fragments/rust-brick-walls).

## Things That Were Awesome
At nearly every turn that I've thought, "wow it would be great if...", Rust has a well thought off solution that strikes just the right balance between idealism and pragmatism.

### Great Docs
The Rust documentation is some of the best technical writing I've ever read. After writing code like:
```rust
let parsed = lang::parse_query(&query);
let query = match parsed {
    Ok((_input, query)) => query,
    Err(s) => return Result::Err(format!("Could not parse query: {:?}", s)),
};
```
My first thought was, "huh, I wonder if you could write a macro for that...". That quickly lead me to [this doc](https://doc.rust-lang.org/book/first-edition/error-handling.html) which not only perfectly answer my question ("yes, with the try!/? macros") as well as guiding me through the progression of the best practices for handling errors in Rust.

```rust
let parsed = lang::parse_query(&query).map_err(|e| format!("Could not parse query: {:?}", e));
let (_, query) = parsed?;
```

Across the board, the docs are clear, easy to read, and, in another stroke of pragmatism, there are actually [tests that all the examples compile](https://doc.rust-lang.org/stable/rustdoc/documentation-tests.html)!

**Watch out for stale docs! A lot of google searches lead you to the first edition of the Rust book. It isn't incorrect (to my knowledge), but there is sometimes a better way.**

### Variable shadowing encouraged
Unlike nearly every language I've ever used, Rust actually _encourages_ variable shadowing. I thought this was a really interesting and pragmatic design decision. It avoids code like:

```scala
val foo = "..."
val fooParsed = parse(foo)
val fooEscaped = escape(fooParsed)
...
// Whoops. Should have used fooEscaped...
doSomethingWith(fooParsed)
```
I've seen multiple production bugs caused by someone intending to replace a value with another, only to have the old value used later.

In Rust you can make the old value unusable by simply shadowing it:
```rust
let foo = "...";
let foo = parse(foo);
let foo = escaped(foo);
...
doSomethingWith(foo);
```

It turns shadowing from a frequent cause of bugs into something that prevents bugs!

### Easing into Ownership
Ownership is tricky, but Rust has an escape hatch: `clone()`. If I couldn't figure out an ownership issue after a minute or two, I could just use `.clone()` or `.cloned()` and effectively punt the issue until I cared about the performance I lost. 

Angle grinder allows you to transform your log data through a series of operators. Initially, each operator would `borrow` the records it processed.
```rust
fn process(&self, rec: &Record) -> Option<Record> {
    ...
    // clone rec and make some changes...
}
```
This was fine, but I ended up doing a lot of unnecessary cloning.

Later, as I began to get a better handle on what patterns worked better and gained a better understanding of ownership, I could improve things. Since each `Record` is moving linearly through the series of operators, it actually makes more sense to have `process` take ownership:
```rust
fn process(&self, rec: Record) -> Option<Record> {
    ...
    // mutate rec, no cloning required
}
```

### Expressive Abstractions
One of my major frustrations when originally writing Sumoshell in Go was running into stack overflow answers like [this](https://stackoverflow.com/questions/21362950/golang-getting-a-slice-of-keys-from-a-map) (How do you get the keys out of map? Use a for-loop and append them into a list. Oy.) Probably not that annoying coming from C, but coming from Scala and Python I felt like Go was actively working against me. Needless to say, I was pleasantly surprised to find that Rust has all the functional programming paradigms I enjoyed in Scala (`map`, `flat_map`, `fold`, etc.). 

The type system of Rust great. Scala's type system is fine, but I frequently end up needing type classes which are a bit of a kludge in Scala. Rust brings type classes (called `Traits` in Rust) as _the way_ of code reuse. Being able to easily add features to an existing class is great. 

## Things that might be awesome in the future

### Crates Don't Have Great SEO
TLDR: If you're looking for a crate, search for it on [https://crates.io](https://crates.io). Many great crates don't show up Google! Another side note: A lot of great, well loved, crates don't have a lot of Github stars.

Coming from Python and Scala, where googling "Python thing_I_want" always finds you the relevant Python package, that Rust crates didn't have as good SEO. As an example: I was wanted to add percentile support to angle grinder. Percentiles are pretty key for most monitoring workflow -- angle grinder lets you do things like ```* | parse "status_code=*] as status_code | parse "response_ms=*]" as response_ms | pct90(response_ms) by status_code```. For this use case, a streaming, approximate percentile implementation is key so you don't leak memory. Googling `Rust percentile` gives you a few options, but none of them are great.[^3] I actually ended up finding a much better crate [quantiles](https://crates.io/crates/quantiles) when I searched for [CKMS](http://ieeexplore.ieee.org/document/1410103/?tp=&arnumber=1410103&url=http:%2F%2Fieeexplore.ieee.org%2Fxpls%2Fabs_all.jsp%3Farnumber%3D1410103). But if I had just [searched crates.io](https://crates.io/search?q=percentile), I would have found it right away. 

### Macro errors are the worst.
Angle grinder is essentially an extremely simple functional programming language wrapped in a CLI App. So, naturally, it needs to parse the aforementioned programing language. Being familiar with parser-combinator style parsing from Scala, I decided to use [Nom](https://github.com/geal/nom), a similar library for Rust. Nom is based on macros. This is great when it allows you to write a lot less code. But it's less nice when you forget an `>` and get an error like:
```rust
error: no rules expected the token `i1`
  --> src/lang.rs:95:1
   |
95 | / named!(json<&str, InlineOperator>, ws!(do_parse!(
96 | |     tag!("json") >
97 | |     from_column_opt: opt!(preceded!(tag!("from"), ident)) >>
98 | |     (InlineOperator::Json { input_column: from_column_opt.map(|s|s.to_string()) })
99 | | )));
   | |____^
```
Once you're familiar with a library, this isn't a big problem. However, error messages like these make for a really nasty learning curve. I'm not sure what the right answer here is.[^2] Scala's macros have this problem when you're writing them, but once they're written, you don't typically get internal compile errors because of how expansion works.

### You Can't Sort Floats
Rust tries, as much as possible, to avoid surprising behavior. Here is some surprising Python behavior:
```python
>>> sorted([3, 4, float('NaN'), 1, 2])
[3, 4, nan, 1, 2]
>>> sorted([3,4,1,2])
[1, 2, 3, 4]
```
In order to prevent this in Rust, it simply doesn't define `Ord` for floats:
```rust
#[derive(PartialOrd, Ord)]
struct FloatHolder {
    f float64
}
```
```rust
error[E0277]: the trait bound `f64: std::cmp::Ord` is not satisfied
   --> src/operator.rs:376:5
    |
376 |     f: f64
    |     ^^^^^^ the trait `std::cmp::Ord` is not implemented for `f64`
    |
    = note: required by `std::cmp::Ord::cmp`
```
I admit I was of two minds on this. Initially, it was infuriating, mostly because my specific use-case at the time didn't enable me to use the escape hatch suggested on the internet, `partial_cmp(...).unwrap_or(Ordering::Less)`. But I think this is a better choice than Python's surprising behavior. I'm looking forward to progress being made on [#1249](https://github.com/rust-lang/rfcs/issues/1249), which proposes adding wrapper types that implement an IEEE complete order on floats. At the end of the day, though, this is the kind of property that can make it frustrating when first getting to know a language.

## Conclusions

At the end of the day, I really enjoyed writing angle grinder in Rust. I spent significantly more time solving software problems than I spent solving language problems. I'm not sure


[^1]: Don't get me wrong, I'm as pro-JVM as anyone, it just really sucks to wait 1-2 seconds to start up a CLI app.
[^2]: One semi-answer is that nightly Rust can show you the backtrace where this failed. This might help if your the macro author, but as a macro user, I can't see this being too helpful.
[^3]: The first result, https://crates.io/crates/histogram only supports unsigned ints, which I found pretty annoying.