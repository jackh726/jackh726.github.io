---
layout: post
title:  "The Rust borrow checker just got (a little bit) smarter"
date:   2022-06-10 11:00 -0400
categories: rust
---

Well, at least, the NLL borrow checker finally got fully enabled by default - but that's not as catchy of a title, is it?

Let's start with the basics. What am I writing this post for? For a few reasons, actually. First, many lifetime errors have changed on nightly (and will hit stable in August), so I want to introduce them a bit, because I think they're really cool. Second, I want to give a little history of how we got here, because I think it's always important to look at things in retrospect. And third, along the way, I want to shout out all the work done by various contributors to get this enabled. I personally have done very little of the work here; I'm mostly going to be telling a story written by others, and they deserve all the credit.

Also note, while there are a handful of things that will now compile that didn't before, I won't really talk about them here. The list is small and most people won't hit them. The [stabilization PR](https://github.com/rust-lang/rust/pull/95565) has a couple examples.

I'll start by just including a snippet of the stabilization report, which gives a brief summary of the motivation of this change, which might help frame the current text.

>Over time, the Rust borrow checker has become "smarter" and thus allowed more programs to compile. There have been three different implementations: AST borrowck, MIR borrowck, and polonius (well, in progress). Additionally, there is the "lexical region resolver", which (roughly) solves the constraints generated through HIR typeck. It is not a full borrow checker, but does emit some errors.
>The AST borrowck was the original implementation of the borrow checker and was part of the initially stabilized Rust 1.0. In mid 2017, work began to implement the current MIR borrow checker and that effort ompleted by the end of 2017, for the most part. During 2018, efforts were made to migrate away from the AST borrow checker to the MIR borrow checker - eventually culminating into "migrate" mode - where HIR typeck with lexical region resolving following by MIR borrow checking - being active by default in the 2018 edition.
>In early 2019, migrate mode was turned on by default in the 2015 edition as well, but with MIR borrowck errors emitted as warnings. By late 2019, these warnings were upgraded to full errors. This was followed by the complete removal of the AST borrow checker.
>In the period since, various errors emitted by the MIR borrow checker have been improved to the point that they are mostly the same or better than those emitted by the lexical region resolver.
>While there do remain some degradations in errors (tracked under the NLL-diagnostics tag, those are sufficiently small and rare enough that increased flexibility of MIR borrow check-only is now a worthwhile tradeoff.

# Error changes

Likely the biggest difference that will be noticed after this change will be to the error messages you get regarding lifetime errors. To showcase some of the improvements, let me give a few examples. Ultimately, I won't be able to give a comprehensive overview (there were over 900 rustc test files changed!), but hopefully these examples can provide some representation. For these examples, I'll give a code snippet, the output on stable 1.61 and on the current nightly.

## Lifetime may not live long enough

The first error change highlights nicely the difference in how the compiler processes lifetime errors before and after this change.

Given the following snippet:

```rust=
fn transmute_lifetime<'a, 'b, T>(t: &'a (T,)) -> &'b T {
    match (&t,) {
        ((u,),) => u,
    }
}
fn main() {
  let y = Box::new((42,));
  let x = transmute_lifetime(&y);
}
```

Stable emits the following error:

```rust
error[E0495]: cannot infer an appropriate lifetime due to conflicting requirements
 --> src/main.rs:2:11
  |
2 |     match (&t,) {
  |           ^^^^^
  |
note: first, the lifetime cannot outlive the lifetime `'a` as defined here...
 --> src/main.rs:1:23
  |
1 | fn transmute_lifetime<'a, 'b, T>(t: &'a (T,)) -> &'b T {
  |                       ^^
note: ...so that the types are compatible
 --> src/main.rs:2:11
  |
2 |     match (&t,) {
  |           ^^^^^
  = note: expected `(&&(T,),)`
             found `(&&'a (T,),)`
note: but, the lifetime must be valid for the lifetime `'b` as defined here...
 --> src/main.rs:1:27
  |
1 | fn transmute_lifetime<'a, 'b, T>(t: &'a (T,)) -> &'b T {
  |                           ^^
note: ...so that reference does not outlive borrowed content
 --> src/main.rs:3:20
  |
3 |         ((u,),) => u,
  |                    ^

For more information about this error, try `rustc --explain E0495`.
```

Note that the primary location of the error points at the scrutinee (`(&t,)`) of the match. Also note that this error is pretty verbose and takes up a whopping 30 lines. On nightly:

```rust
error: lifetime may not live long enough
 --> src/main.rs:3:20
  |
1 | fn transmute_lifetime<'a, 'b, T>(t: &'a (T,)) -> &'b T {
  |                       --  -- lifetime `'b` defined here
  |                       |
  |                       lifetime `'a` defined here
2 |     match (&t,) {
3 |         ((u,),) => u,
  |                    ^ function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
  |
  = help: consider adding the following bound: `'a: 'b`
```

This is quite different. First, it's much more terse in output; only taking up 12 lines. Second, the error points primarily at `u`, which actually is what get returned by the match expression. Finally, we get an actual helpful suggestion to add the `'a: 'b` bound, which actually does fix this code. On the other hand, we do lose the "expected XX found XX" note, but it's not super helpful in this case anyways. Also note that we lose the error code (E0495), which is unfortunate as this might be able to point the user to a more detained explanation.

Here's a slightly different code snippet:

```rust=
pub fn opt_str<'a>(maybestr: &'a Option<String>) -> &'static str {
    if maybestr.is_none() {
        "(none)"
    } else {
        let s: &'a str = maybestr.as_ref().unwrap();
        s
    }
}
```

The error on stable isn't too bad and fairly straightforward:

```rust
error[E0312]: lifetime of reference outlives lifetime of borrowed content...
 --> src/lib.rs:6:9
  |
6 |         s
  |         ^
  |
  = note: ...the reference is valid for the static lifetime...
note: ...but the borrowed content is only valid for the lifetime `'a` as defined here
 --> src/lib.rs:1:16
  |
1 | pub fn opt_str<'a>(maybestr: &'a Option<String>) -> &'static str {
  |                ^^

For more information about this error, try `rustc --explain E0312`.
```

And on nightly:

```rust
error: lifetime may not live long enough
 --> src/lib.rs:6:9
  |
1 | pub fn opt_str<'a>(maybestr: &'a Option<String>) -> &'static str {
  |                -- lifetime `'a` defined here
...
6 |         s
  |         ^ returning this value requires that `'a` must outlive `'static`
```

We give all the same infomration in a slightly shorter error. We also lose the error code (E0312), which again is unfortunate. As you might notice though, what were two very different error outputs before look a bit more similar now: we do slightly less arbibtrary grouping.

## Borrowed data escapes function

This next error is a little less straightforward. In fact, this is a case we actually regress a bit, but not *too* bad.

```rust=
use std::sync::Mutex;
struct MyString<'a> {
    data: &'a str,
}
fn i_want_static_closure<F>(a: F)
where
    F: Fn() + 'static,
{
}
fn print_string<'a>(s: Mutex<MyString<'a>>) {
    i_want_static_closure(move || {
        println!("{}", s.lock().unwrap().data);
    });
}
```

Before I show the errors, let me just say: there's a lot going on here. However, the crux of the error is the `'static'` bound on `F` on line 7. So...stable:

```rust
error[E0477]: the type `[closure@src/lib.rs:11:27: 13:6]` does not fulfill the required lifetime
  --> src/lib.rs:11:5
   |
11 |     i_want_static_closure(move || {
   |     ^^^^^^^^^^^^^^^^^^^^^
   |
note: type must satisfy the static lifetime as required by this binding
  --> src/lib.rs:7:15
   |
7  |     F: Fn() + 'static,
   |               ^^^^^^^

For more information about this error, try `rustc --explain E0477`.
```

Okay, great. This is exactly what we want: The closure is a problem, and it's because we don't satisfy that `'static'` bound. Nightly:

```rust
error[E0521]: borrowed data escapes outside of function
  --> src/lib.rs:11:5
   |
10 |   fn print_string<'a>(s: Mutex<MyString<'a>>) {
   |                   --  - `s` is a reference that is only valid in the function body
   |                   |
   |                   lifetime `'a` defined here
11 | /     i_want_static_closure(move || {
12 | |         println!("{}", s.lock().unwrap().data);
13 | |     });
   | |      ^
   | |      |
   | |______`s` escapes the function body here
   |        argument requires that `'a` must outlive `'static`
   |
   = note: requirement occurs because of the type `Mutex<MyString<'_>>`, which makes the generic argument `MyString<'_>` invariant
   = note: the struct `Mutex<T>` is invariant over the parameter `T`
   = help: see <https://doc.rust-lang.org/nomicon/subtyping.html> for more information about variance

For more information about this error, try `rustc --explain E0521`.
```

Well...this isn't great. We at least get the "`'a` must outlive `'static`", which might give some clue to the problem. But, we also get a couple notes about variance. This is not really helpful here - it doesn't have any effect on the error (even though it *is* true).

## Helpful variance note

Okay, let's pick out an error that actually does stem from variance and how it has changed. We'll have this be the last one.

```rust=
struct SomeStruct<T>(*mut T);

fn foo<'min, 'max: 'min>(v: SomeStruct<&'max ()>) -> SomeStruct<&'min ()> {
    v
}
```

Just quickly, I want to point out the `*mut T` in `SomeStruct`. If this was only `T`, this snippet wouldn't have any problem. See if you can find out why from the errors (I did give a hint above). On stable:

```rust
error[E0308]: mismatched types
 --> src/lib.rs:4:5
  |
4 |     v
  |     ^ lifetime mismatch
  |
  = note: expected struct `SomeStruct<&'min ()>`
             found struct `SomeStruct<&'max ()>`
note: the lifetime `'min` as defined here...
 --> src/lib.rs:3:8
  |
3 | fn foo<'min, 'max: 'min>(v: SomeStruct<&'max ()>) -> SomeStruct<&'min ()> {
  |        ^^^^
note: ...does not necessarily outlive the lifetime `'max` as defined here
 --> src/lib.rs:3:14
  |
3 | fn foo<'min, 'max: 'min>(v: SomeStruct<&'max ()>) -> SomeStruct<&'min ()> {
  |              ^^^^

For more information about this error, try `rustc --explain E0308`.
```

So...this error is *okay*. We get told that we're expecting `SomeStruct<&'min ()>` and that `'min` doesn't outlive `'max`. But we don't really know *why*. There's an error code, but it's pretty generic. Let's look at nightly:

```rust
error: lifetime may not live long enough
 --> src/lib.rs:4:5
  |
3 | fn foo<'min, 'max: 'min>(v: SomeStruct<&'max ()>) -> SomeStruct<&'min ()> {
  |        ----  ---- lifetime `'max` defined here
  |        |
  |        lifetime `'min` defined here
4 |     v
  |     ^ function was supposed to return data with lifetime `'max` but it is returning data with lifetime `'min`
  |
  = help: consider adding the following bound: `'min: 'max`
  = note: requirement occurs because of the type `SomeStruct<&()>`, which makes the generic argument `&()` invariant
  = note: the struct `SomeStruct<T>` is invariant over the parameter `T`
  = help: see <https://doc.rust-lang.org/nomicon/subtyping.html> for more information about variance
```

Wow, that has a lot more information (and it's also a bit more terse). And we get a suggestion (that works)! Does this help you understand why the code snippet is problematic? We can't substitute `SomeStruct<&'max ()>` for `SomeStruct<&'min ()>` in the return type, because `T` is invariant. If `T` was covariant, we could.


## Recap

I'm not sure if this was too much - or too little. It's also most just pasting a few code snippets and their errors, with a bit of commentary. Hopefully it at least scratches the surface of the types of lifetime error differences you might see on nightly compared to stable. If you're interested in checking out how lifetime errors in rustc test suite have changed, feel free to check out the [PR](https://github.com/rust-lang/rust/pull/95565/files) or take a look at the short list of [diagnostic regressions](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3ANLL-diagnostics).

# How did we get here?

So, for this section, I'm going to work backwards. This is primarily because it's just easier to recall recent work than older work. I want to just briefly point out some of the major stepping stones towards this stabilization and shout out the various contributors that pushed to make this happen.

## Triaging diagnostic changes and preparing for stabilization

The [stabilization PR](https://github.com/rust-lang/rust/pull/95565) had 985 changed files. Most of those were test-related. It's impractical to expect a reviewer to go through all of those changes and "sign off" on them all. Adding to that, with the way the rustc test suite is set up, changing test error output requires some manual intervention, so it would be helpful to make such a *big* PR be as mechanical as possible.

Over the past couple months [@marmeladema](https://github.com/marmeladema) and I have worked to [go](https://github.com/rust-lang/rust/pull/95591) [through](https://github.com/rust-lang/rust/pull/96148) [all](https://github.com/rust-lang/rust/pull/96212) [the](https://github.com/rust-lang/rust/pull/97258) test error differences, file [issues](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3ANLL-diagnostics) for those where diagnostics have regressed (subjectively), and prepare the tests for a purely mechanical stabilization PR.

[@m-ou-se](https://github.com/m-ou-se) also triaged the crater run for the stabilization PR to give the stabilization a clean bill of health.

## Tying up loose ends

There are three parts to this.

First, [@marmeladema](https://github.com/marmeladema) and [@Aaron1011](https://github.com/Aaron1011) made some great PRs over the [last](https://github.com/rust-lang/rust/pull/96409) [couple](https://github.com/rust-lang/rust/pull/96385) [months](https://github.com/rust-lang/rust/pull/96352) [to](https://github.com/rust-lang/rust/pull/96236) reduce the number of diagnostic regressions.

Second, one of the last "big questions" prior to this stabilization was regarding to [`mutable_borrow_reservation_conflict` lint](https://github.com/rust-lang/rust/issues/59159) (this actually came up in the crater run mentioned above). Without going into details, there were concerns that allowing the linted pattern would disallow some optimizations. It accidentally got stabilized in Rust 2018 and the thought was maybe to disallow it after full NLL stabilization. The lang team decided against this and the lint was removed.

Finally, the last bit of known behavior difference with the full stabilization concerned coercion with match statements and the [order of match expressions](https://github.com/rust-lang/rust/issues/73154). Actually, in migrate mode, the coercion here fails. Fully enabling NLL allows the pattern to compile, but only in one ordering. While we do want the coercion to compile eventually, we chose for now to just fallback to migrate mode behavior of [disallowing the coercion in either ordering](https://github.com/rust-lang/rust/pull/97206).

## Huge diagnostic overhaul

Up until mid last year, progress on enabling NLL had mostly stalled. The biggest and most daunting blocker being a slew of hideus and generic "higher-ranked subtype error" for a large number of lifetime errors. [@matthewjasper](https://github.com/matthewjasper) did some fantastic work in a branch that [@lqd](https://github.com/lqd) rebased and made a [PR](https://github.com/rust-lang/rust/pull/86700) for. It didn't solve all the errors, but it provided a framework to solve most of the remaining problems. [@lqd](https://github.com/lqd) followed up with another [PR](https://github.com/rust-lang/rust/pull/88270) which fixed more diagnostics.

Then, [@Aaron1011](https://github.com/Aaron1011) [made](https://github.com/rust-lang/rust/pull/88708) [a](https://github.com/rust-lang/rust/pull/89028) [long](https://github.com/rust-lang/rust/pull/89110) [series](https://github.com/rust-lang/rust/pull/89250) [of](https://github.com/rust-lang/rust/pull/89249) [PRs](https://github.com/rust-lang/rust/pull/89336) [addressing](https://github.com/rust-lang/rust/pull/89504) [various](https://github.com/rust-lang/rust/pull/89501) [diagnostics](https://github.com/rust-lang/rust/pull/92306). At this point, NLL was very close to being "ready" and just needed a few loose ends tied up.

## Enabling migrate mode and removing AST borrowck

In 2018, migrate mode was [added and enabled](https://github.com/rust-lang/rust/pull/52681) for the Rust 2018 edition by [@pnkfelix](https://github.com/pnkfelix). However, if code was rejected by NLL, but allowed by the AST borrowck, issues were only emitted as warnings. [@spastorino](https://github.com/spastorino) also [made](https://github.com/rust-lang/rust/pull/52083) the AST borrowck not run at all under `feature(nll)`.

Over the period of 2019, lots of shifting happening in the migration from the AST borrowck to the MIR borrowck (NLL). In March, [@matthewjasper](https://github.com/matthewjasper) [enabled](https://github.com/rust-lang/rust/pull/59114) migrate mode in Rust 2015, but with warning emitted instead of errors. [@chrisvittal](https://github.com/chrisvittal) [removed](https://github.com/rust-lang/rust/pull/60513) the mostly unused `-Zborrowck=compare` flag. Then [@Centril](https://github.com/Centril) made a [series](https://github.com/rust-lang/rust/pull/63565) [of](https://github.com/rust-lang/rust/pull/64221) [PRs](https://github.com/rust-lang/rust/pull/64790) that upgraded migrate mode NLL warnings to errors in Rust 2018 and Rust 2015, then finally removed the unused AST borrowck.

## NLL implementation

In the latter half of 2017 and into 2018, work was done to implement the MIR borrowck and bring it to feature and performance parity. [@nikomatsakis](https://github.com/nikomatsakis) [envisioned](https://github.com/rust-lang/rfcs/pull/2094) most of the borrow checker and did a large portion of the [implementation](https://github.com/rust-lang/rfcs/pull/2094) [work](https://github.com/rust-lang/rust/pull/46862). However [@Nashenas88](https://github.com/Nashenas88) made [a](https://github.com/rust-lang/rust/pull/43271) [few](https://github.com/rust-lang/rust/pull/43324) [PRs](https://github.com/rust-lang/rust/pull/43559) to prepare the compiler for Niko's branch. [@pnkfelix](https://github.com/pnkfelix) and [@shepmaster](https://github.com/shepmaster) did great work tracking progress. [@nnethercote](https://github.com/nnethercote) did some amazing work [speeding up the implementation](https://blog.mozilla.org/nnethercote/2018/11/06/how-to-speed-up-the-rust-compiler-in-2018-nll-edition/). It's very possible I've missed more, so if you see someone missing from this list, please DM me and I'll correct it.

# Summary

That basically concludes this post. As always, this turned out a bit different from how I imagined it in my mind. But I do think it's important to recount the history of the implementation and give credit to those that did the work to get where we are today. As a side effect, maybe a few people might find it interesting or get some use from going over a couple lifetime error differences.

As a side note, I do welcome feedback on these. I tend to make my blog posts more "informal" and "prose-like", versus a more rigid structure. The downside of this is that they can be less "beginner-friendly", in a sense. I would love to know if people enjoy these or not.
