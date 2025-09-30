---
layout: post
title:  "A shiny future with GATs"
date:   2022-05-04 10:30 -0400
categories: rust
---

This was a surprisingly difficult blog post to write. Between general life things getting in the way and feeling a bit of lost steam, this took much longer than I expected.

Before I go further, if you don't know what GATs (generic associated types) are, then I recommend reading [this blog post](https://blog.rust-lang.org/2021/08/03/GATs-stabilization-push.html) from August of last year.

A bit over [a year ago](https://github.com/rust-lang/rust/issues/44265#issuecomment-775423925), the traits working group started to talk seriously about stabilizing GATs. I and others worked to close issues and move that forward. In [June last year](https://github.com/rust-lang/rust/issues/44265#issuecomment-869888398), I estimated October for stabilization. Well, turns out there were some things that we needed to do first (besides fixing bugs), like [implementing an outlives lint](https://github.com/rust-lang/rust/pull/89970) and [changing the location of where clauses on GATs](https://github.com/rust-lang/rust/pull/90076). [In December](https://github.com/rust-lang/rust/issues/44265#issuecomment-990287168), once again, I said "we're close." Around Feburary or [March](https://github.com/rust-lang/rust/issues/44265#issuecomment-1066173042), we felt like we were finally in a place to stabilize. Except for one *small* thing: this blog post.

The goal of this blog post is to try to imagine a "shiny future" where we have GATs. It's to consider a couple of the patterns that people want GATs for, how we might want to integrate GATs into standard library traits, and a few issues that don't work *today* that we want to work in the future. We want this so that we have a future to strive for after stabilization. There are known shortcomings with GATs right now, but that's okay. They're already powerful and, as you'll see, we have some thoughts on how to make them more powerful and ergonomic in the future.

Like I said, it's been hard to write this. Partially because it's *a lot*, but also because it's difficult to think about the future and try to consider the intersection of a several different patterns. This isn't to say that I alone have come up with these thoughts: much of this has been talked about within the traits working group. A huge shoutout to Niko Matsakis for an unenumerable amount of feedback, banter, and guidance.

Well, I've finally finished this post. To go with this, I've posted a [stabilization PR](https://github.com/rust-lang/rust/pull/96709) for GATs to Github. It's happening, y'all.

## Getting our feet wet with higher-kinded types

To start with, I want to cover a pattern involving traits with GATs more-or-less representing the "Self" type. I'll start by introducing two separate but related examples. After I introduce them, I'll explain a bit about how they are similiar. Finally, I'll discuss a bit how a full "higher-kinded types" feature might solve this.

### Introducing the examples

The first example roughly follows the `Collection` trait [introduced](http://smallcultfollowing.com/babysteps/blog/2016/11/03/associated-type-constructors-part-2-family-traits/) by Niko Matsakis in a series of blog posts in 2016. Here I've made two changes. First, the type parameter `T` on `Collection` has been changed to an associated type - which more closely matches real collections like `Vec` or `Option`. Second, the `Family` trait has been removed also in favor of just an associated type on the `Collection` trait. Here's what the new trait looks like an an impl for `Vec`:

```rust
trait Collection {
    type Item;
    type Member<T>: Collection<Item = T>;
    type Iter<'iter>: Iterator<Item = &'iter Self::Item>
    where
        Self: 'iter;

    fn empty() -> Self;
    fn add(&mut self, value: Self::Item);
    fn iter<'iter>(&'iter self) -> Self::Iter<'iter>;
}

impl<T> Collection for Vec<T> {
    type Item = T;
    type Member<U> = Vec<U>;
    type Iter<'iter> = std::slice::Iter<'iter, T> where T: 'iter;

    fn empty() -> Self {
        Vec::new()
    }

    fn add(&mut self, value: T) {
        self.push(value)
    }

    fn iter<'iter>(&'iter self) -> Self::Iter<'iter> {
        use std::ops::Deref;
        self.deref().iter()
    }
}
```

You could also imagine you might have a function like

```rust
fn floatify<C>(ints: &C) -> C::Member<f32>
where
    C: Collection<Item=i32>,
{
    let mut res = C::Member::<f32>::empty();
    let mut iter = ints.iter();
    while let Some(v) = iter.next() {
        res.add(*v as f32);
    }
    res
}
```

The second example revolves around mimicking the `Functor` typeclass from Haskell and functional programming. In Haskell, a Functor allows you to map the element(s) inside some type of container into something else. In Rust, imagine `Option::map` or `Result::map`, but generic over many types. There have been a few blog posts published on this topic, so again I'll just put here a final trait and an impl (this time for `Option`):

```rust
trait Functor {
    type Inner;
    type This<B>: Functor;

    fn map<F, B>(self, f: F) -> Self::This<B>
    where
        F: FnOnce(Self::Inner) -> B;
}
impl<T> Functor for Option<T> {
    type Inner = T;
    type This<B> = Option<B>;

    fn map<F, B>(self, f: F) -> Self::This<B>
    where
        F: FnOnce(Self::Inner) -> B {
        self.map(f)
    }
}
```

### Similarities in the traits and a problem

If you compare these two traits, `Collection` and `Functor`, you might realize that they look oddly similar. Both have an associated item for the "item inside" (`Item` and `Inner`) and an associated item that represents the current type, but wrapping a generic item type (`Member<T>` and `This<B>`).

However, if you think about the way these traits are set up, you might notice a potential problem. There is nothing stopping someone from writing an impl like (assuming `Result: Functor` holds):

```rust
impl<T> Functor for Option<T> {
    type Inner = T;
    type This<B> = Result<B, ()>;

    fn map<F, B>(self, f: F) -> Self::This<B>
    where
        F: FnOnce(Self::Inner) -> B {
        self.map(f).ok_or(())
    }
}
```

Namely, we *expect* that `This<B>` to be equal to `Option<B>` (the current container type with a new item type), but instead it is a `Result`! Maybe this is okay, but it's certainly unexpected.

### Solving this with higher-kinded types

So, how can we solve this? This problem originates mostly because we're trying to *emulate* higher-kinded types.

I don't want to focus on the specific syntax that I'm using (because I'm basically making it up on the spot with little thought and it's not very relevant), but here's how you might write `Functor` with higher-kinded types:

```rust
trait Functor where Self<Inner> {
    fn map<F, B>(self, f: F) -> Self<B>
    where
        F: FnOnce(Inner) -> B;
}
impl<T> Functor for Option<T> {
    fn map<F, B>(self, f: F) -> Self<B>
    where
        F: FnOnce(T) -> B {
        self.map(f)
    }
}
```

Now, you might be asking how this relates to GATs. And, well, the answer is: *it doesn't*, really. The solution I show above is pretty far from the GATs feature itself. But I wanted to cover this for two reasons: 1) The initial examples *are* likely patterns you will see with GATs, even with the limitation described. 2) In theory, the fundamental compiler features needed to support GATs are likely the fundamentals of the ones needed to be able to "real" higher-kinded types - or maybe something similar but more Rust-like.

## A GATified `Iterator`

There are many cases today where people want to do something with `Iterator`, `Fn`, `Deref`, etc. that *isn't yet possible*. Instead, what they require is really a GAT version of these traits - often with a lifetime parameter.

The question then becomes: what is the best way to make GAT versions of these traits available? I want to explore here `Iterator`, since that is arguably the most well-known example case. I want to look first at the pattern that requires GATs. Then, I want to introduce a `LendingIterator`, and explain why might be a less-than-ideal solution. I'll show a bit about a couple of the things that it might take to replace `Iterator` with a GATified version (in a backwards-compatible manner). And I'll end off the section with a somewhat crazy thought I was actually able to get working as I was writing this blog post.

### An example case on returning an item that borrows from `Self`

Let's start by looking at what the current `Iterator` trait looks like:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Self::Item;
}
```

Pretty simple, but I've obviously left out a large chunk of methods with default implementations, like `for_each`, `map`, or `peekable`. I've left them out for simplicity, but they are important and I'll talk about that in a bit.

To introduce the pattern that requires GATs, I'll use the `std::io::BufRead` trait. This is implemented most clearly by the `std::io::BufReader` struct. In short, this trait represents a type that implements `std::io::Read` but *also* keeps an internal buffer, allowing it to be able to do extra things, like returning line by line.

I want to focus on a single method on `BufRead`:

```rust
pub trait BufRead: Read {
    fn lines(self) -> Lines<Self>;
}
```

This function returns a struct `Lines` which implements `Iterator` with an `Item` of `Result<String>`. Let's imagine it gets used like:

```rust
fn main() {
    let r: BufReader<_> = ...;
    let mut lines = r.lines();
    while let Some(line) = lines.next() {
        println!("{line}");
    }
}
```

Now what's *wrong* with this (aside from me being able to use a for loop - I'll get there). It might seem like a philosophical question, but there really is something suboptimal: even though we only look at one line at a time, we allocate a new `String` for each one. Of course, this is the exact reason we need GATs. If we wanted to be able to reuse the same memory between calls to `next`, then we would have to ensure that nobody else hold a reference to that memory. But  with the current definition of `Iterator::next`, *the return time can outlive the call*.


### Introducing `LendingIterator`

Let's introduce a `Iterator`-like trait that could handle this correctly:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Self::Item<'_>; 
}
```

Here, I've defined a new trait, `LendingIterator`, that mirrors `Iterator`, except for a new lifetime `'a` on `Item`. It does exactly what we want:

```rust
struct Lines<B: BufRead> { ... } // Stores the state needed to read lines

impl<B: BufRead> LendingIterator for Lines<B> {
    type Item<'a> = &'a str;

    // The lifetime of the return type is tied to the lifetime of the call
    fn next(&mut self) -> Self::Item<'_> { ... }
}

fn read_two_lines<B: BufRead>(mut lines: Lines<B>) {
    while let Some(line) = lines.next() {
        // `line` is only a `&str`
        // The reference is dropped before the next call
        println!("{line}");
    }
    // We also store two lines at a time:
    let line1 = lines.next();
    let line2 = lines.next(); // <- Error: the lifetime from `line1` is still active
    println!("{line1} {line2}");
}
```

So, this looks great. Why is this imperfect? Well, remember all those methods on iterator with default implementations? We now need to rewrite those. Also, any APIs that take `Iterator` but don't read more than one `Item` at a time would need to be to changed or duplicated to take `LendingIterator`.

### Reusing `Iterator` in a `LendingIterator` way

So, maybe we don't want a completely separate `LendingIterator` trait from `Iterator`. What can we do instead? Well, what if we *just* added a `'a` parameter to `Iterator::Item`. Let's go through some of the different problems that would have to be worked through.

To start, I want to actually just introduce some code, which will help me explain things after.

```rust
fn from_iter<A, I>(mut iter: I) -> Vec<A>
where
    I: Iterator<Item = A>,
{
    let mut v: Vec<I::Item> = vec![];
    while let Some(item) = iter.next() {
        v.push(item);
    }
    v
}
```

This is completely fine. In fact, it's mostly the same as the `FromIterator` implementation for `Vec`.

Let me also show what this would look like with a `LendingIterator`. Keep in mind that we *want* this to be just `Iterator` with a lifetime parameter on `Iterator::Item`. I'm keeping them separate here to keep comparisons easy to understand.

```rust
fn from_iter<A, I>(mut iter: I) -> Vec<A>
where
    I: for<'a> LendingIterator<Item<'a> = A>, // (1)
{
    let mut v/*: Vec<I::Item>*/ = vec![]; // (2)
    while let Some(item) = iter.next() {
        v.push(item);
    }
    v
}
```

This also works, but differs from the `Iterator` case in two places, which I'll talk about next.

Note: It's nice to see that borrow checker knows that the item being iterated over doesn't capture the `self` lifetime from `next`; with this, we can store `item` in the Vec without problems. This makes intuitive sense and it's good to see that *this* isn't a problem we have to work through.

#### `I: for<'a> LendingIterator<Item<'a> = A>`

Let's first look at the line labeled `(1)` above. What is this `for<'a>` business? Well, we have to have *some* way to denote the where clause "the item being iterated over is `A`, regardless of the lifetime passed." In other words, the type being iterated over can't name the lifetime passed. Therefore, it acts exactly like a non-generic associated type.

Now, let's remember that we want to replace the definition of `Iterator` with the definition of `LendingIterator`. So, we need to make `I: Iterator<Item = A>` "desugar" into `I: for<'a> Iterator<Item<'a> = A>`. Semantically, this is actually fairly straightforward.

#### `Iterator::Item`

Now let's look at the line labeled `(2)`.

In the `Iterator` example, we can name the type of the item being iterated by doing `I::Item`. With `LendingIterator` though, we can't do this, because `I::Item` isn't a type, it's a *type constructor*. To get an actual type, we have to use a lifetime, like `I::Item<'static>`. This analogous in that you can't just write `let x: Vec;`, you have to write something like `let x: Vec<()>;`.

Now, you might say "oh, but you could just write `Vec<A>` or `Vec<I::Item<'static>>` and it will be the same" and you would be correct: That would be fine. But if only it was so simple.

Again, let's remember the goal of converting `Iterator`. We can't suddently just break all the code out there. So, we have to figure out how to make `v: Vec<I::Item>` work, even if `Iterator::Item` has a lifetime parameter. For the example above, we could just handwave and see that we know that it doesn't matter what lifetime we pass to `I::Item`, since `A` can't bind it. So, we could "desugar" this to either `I::Item<'static>`, some higher-ranked `for<'a> I::Item<'a>`, or maybe some "empty" lifetime `I::Item<'empty>`.

#### Complicating matters a bit

Let's look at this example:

```rust
fn from_iter<I>(mut iter: I) -> Vec<I::Item>
where
    I: LendingIterator,
{
    ...
}
```

Well, this is more complicated. The code as written doesn't make much sense. Of course, if this were just a "clean" `LendingIterator` trait, then we just maybe want to return `Vec<I::Item<'static>>` and all is well and good (we have to pass *some* lifetime to the GAT). But, if we imagine that we just took `Iterator` and added a lifetime parameter, it becomes clear that we need some sort of *implicit* `I: for<'a> Iterator<Item<'a> = A>` for `Iterator`.

But now if `Iterator::Item` implicitly *can't* capture a lifetime, we have to allow code to *opt-in* to that. What might the syntax for that look like? To be honest, I'm not really sure. To extend upon this, you likely don't want *all* code using traits with GATs to have to opt-in to being able to allow the GATs to name a lifetime. So, `Iterator` would probably be special in some way.

I would love to hear thoughts on how *you* would like to interact with a GATified `Iterator`.

#### *Can* they be separate traits?

As I was writing this blog post, I played around with an idea that was brought up at one point or another. I couldn't get it to work then, but was able to get *something* to work, at least a little bit this time.

So, I'm going to spitball this potentially crazy idea, if for no other reason than to let people go "what the heck is he thinking" and pick it apart.

So bear with me here. What if we *do* have two traits, `Iterator` and `LendingIterator`, but they look something like this:

```rust
pub trait LendingIterator {
    type Item<'a> where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
impl<'i, I: Iterator + 'i> LendingIterator for I {
    type Item<'a> = I::Item where I: 'a;

    fn next(&mut self) -> Option<Self::Item<'i>> {
        Iterator::next(self)
    }
}
```

In words, every implementation of `Iterator` automatically implements `LendingIterator` with the items equal.

I *think* this *mostly* works.

Here are couple examples of how you might use these:

```rust
fn print_items<I>(mut iter: I)
where
    I: LendingIterator,
    for<'a> I::Item<'a>: std::fmt::Debug,
{
    while let Some(item) = iter.next() {
        println!("{item:?}");
    }
}

fn collect_items<I>(mut iter: I) -> Vec<I::Item>
where
    I: Iterator,
{
    let mut v = vec![];
    while let Some(item) = iter.next() {
        v.push(item);
    }
    v
}
```

Importantly, you could change the top function to use `Iterator` and it works (though it accepts *fewer* types, since we accepted all `Iterator` types already because of the blanket impl). You *cannot* change the bottom function to use `LendingIterator`, since that would end up with overlapping mutable borrows.

There are couple pain points here that I've found as I played with it a bit; let me go through them.

First, you can't actually call `next()` on a type that implements `Iterator` without disambiguating the `next` function of `Iterator` and `LendingIterator`. The "simple" solution is to rename `LendingIterator::next` to something like `lend_next`. I've tried making the defintion of `Iterator` be `Iterator: LendingIterator`, but haven't gotten it to work.

The other pain point (which might actually be *good*), is that all (already existing) functions don't *automatically* get to take a `LendingIterator`. I *think* changing from `I: Iterator` to `I: LendingIterator` is a backwards compatible change, but haven't worked through it to be sure. Related to this, if your impl of `LendingIterator` *just so happens* to not name the lifetime parameter on `Item`, you can't use a function that takes `Iterator`. But again, maybe this is the right thing.

And, of course, this doesn't generalize to other traits with GATs.

So, yeah. This was my crazy revelation while writing this. Here's a playground link where I've added some other examples, if you're interested in picking it apart: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=7647fd487f1733fe97fa55c38e4071ab

## The `for<'a> Self: 'a` problem

It's interesting, just earlier this week an absolutely beautiful blog post was published by Sabrina Jewson ([link](https://sabrinajewson.org/blog/the-better-alternative-to-lifetime-gats)) that mostly covered this problem, and gave some good and interesting solutions.

I'll briefly restate the problem, but if you're interested in a more "user-friendly" explantation defintion go check out Sabrina's post; it's great. I'll use the same `LendingIterator` example Sabrina uses, the problem definitely pops up in [even](https://github.com/rust-lang/rust/issues/90696) [more](https://github.com/rust-lang/rust/issues/91693) [subtle](https://github.com/rust-lang/rust/issues/92096) [places](https://github.com/rust-lang/rust/issues/95268).

So, the problem in the `LendingIterator` comes from the interaction of two pieces of code.

First, there is the `Self: 'this` bound on the `Item` associated type:

```rust
pub trait LendingIterator {
	type Item<'this> where Self: 'this;
                        // ^^^^^^^^^^^ this one
    ...
}
```

Second, we encounter a piece of code where we reference an GAT using a lifetime from a HRTB:

```rust
fn print_items<I>(mut iter: I)
where
	I: LendingIterator,
	for<'a> I::Item<'a>: Debug, // here
{
	while let Some(item) = iter.next() {
		println!("{item:?}");
	}
}
```

As noted in Sabrina's post, the problem comes because we try to prove that `for<'a> I: 'a`, meaning that we require for any lifetime that we pick, `I` must outlive it. This isn't really right, because ultimately, we don't need to prove that I can outlive *any* lifetime in this context. We only need to prove that `I` outlives any lifetime that we *could* have provided *if the GAT was valid* (in this case, this is only true if `I: 'a`, which makes `for<'a> I: 'a` trivially provable).

This is a bug. And one that is fixable in an almost certainly backwards-compatible manner. But it's also tough to fix. So it's not something we want to block stabilization on.

To give a *brief* teaser to what a fix *might* look like, you might want to check out Niko Matsakis [a-mir-formality](https://github.com/nikomatsakis/a-mir-formality). The key part of this formalism of the Rust type system (that differs from the current rustc and Chalk solvers), is the introduction of the "ensures" clause and the separation from an "implication" clause. If I understand the model correctly, I think this is crucial to being able to model and solve this problem.

## Object safety

Right now, you can't use GATs with `dyn`. In other words, the following doesn't work:

```rust
fn print_items(items: &mut dyn for<'a> LendingIterator<Item<'a> = &'a str>) {
    for item in items {
        println!("{item}");
    }
}
```

We expect this to work just fine one day, but there are still some implementation design work to be done.

## Wrapping up

Hopefully this post provides a little bit of a glimmer of a shiny future with GATs. I wanted to make this post more "glamorous" but really just fell short on time. I might publish a part 2 of this at some point in the coming months; we'll see.

Ultimately though, I don't think I can do justice to the many different use cases of GATs and the power they provide. If you'd like to get a sense of the many projects that are "waiting" for GATs, just take a scroll through the [tracking issue](https://github.com/rust-lang/rust/issues/44265) and see the numerous issues from external projects linking to it. I can't wait to see the types of projects people build with them.

If you feel like I've missed anything, please feel to reach out.
