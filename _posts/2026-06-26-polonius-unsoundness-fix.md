---
layout: post
title:  "Fixing an unsoundness with the Polonius borrow checker"
date:   2026-06-26 9:00 -0400
categories: rust
---

If you've used Rust enough, you might have run into one or two limitations in the borrow checker. Maybe you tried to write a function like this, and got an error:

```rust
fn get_default<K, V>(
    map: &mut HashMap<K,V>,
    key: K,
) -> &mut V {
    match map.get_mut(&key) {
        Some(value) => value,
        None => {
            map.insert(key, V::default());
            map.get_mut(&key).unwrap()
        }
    }
}
```

I won't go into details about this particular example, but this is a well-known problem referred to as ["problem case 3"][problem-case-3].

You may know that there are ongoing efforts (for many years now!) to fix the borrow checker to "just work" for this example and others. That work has been dubbed "Polonius" and talked about in great detail by Niko Matsakis and others. Niko has [a series of posts][niko-nll-polonius] talking about Polonius and its predecessor NLL, as well as a [couple more recent posts][niko-polonius-revisited] on the most recent approach to implementing Polonius-style borrow checking.

We're getting [very close to stabilizing][polonius-project-goal] the new iteration of the borrow checker, which has been coined "Polonius Alpha" (thanks in large part to [lqd]), and I want to talk today about one little hiccup that was discovered where Polonius Alpha is unsound when using `impl Trait` (and associated types, but we'll get there). I want to walk through the initial problem that was found, how it is more comprehensive than originally thought, and then the solution.

I won't go into *too many* details of how Polonius works under the hood; for that, you should go read Niko's latest couple blog posts, since they're very good. Instead, I'm going to try to give only the details that matter to explain the problem and the solution.

[problem-case-3]: https://rust-lang.github.io/rfcs/2094-nll.html#problem-case-3-conditional-control-flow-across-functions
[niko-nll-polonius]: https://smallcultfollowing.com/babysteps/categories/nll/
[niko-polonius-revisited]: https://smallcultfollowing.com/babysteps/series/polonius-revisited/
[polonius-project-goal]: https://rust-lang.github.io/rust-project-goals/2026/polonius.html
[lqd]: https://github.com/lqd

## Liveness & opaque types & Polonius, oh my

### What is liveness, anyways?

The first thing to talk about is *liveness*. Put simply, a variable is *live* if it could be used again:

```rust
fn foo() {
    let x = String::from("test"); //
    let y = &mut x;              // -- x is live
    call_a();                    //  |  -- y is live
    call_b();                    //  |   |
    drop(y);                     //  |  --
    call_c();                    //  |
}                                // -- (x is implicitly dropped)
```

Of course, it's a bit more complicated than that: we don't just think about live *variables*, but live *regions* (which are sort of the internal representation of a "lifetime"). For the above example, while `y` is live, so is a mutable *borrow* of `x`. So, because `y` is live at `call_b()`, we could not, for example, use `x` because `y` is later used.

This (and more) is of course available on stable today with NLL: when program flow is linear, borrow checking is "easy". However, with Polonius Alpha, liveness calculation is extended to take into account branches within a function, too. And further, liveness is *combined* with known lifetime-outlives relationships to reduce false-positive borrow-check errors. However, incorrect liveness information can result in Polonius disregarding needed outlives requirements, leading to missed borrow-check errors and unsound programs being accepted. To read into the specifics of how liveness is used, Niko shares much more detail in his blog posts.

So, how is liveness calculated anyways? Let's start with two simple examples:

```rust
let x: &'a u8;

let z: MyStruct::<'b>;
```

The first, we saw this above: as long as `x` is live, so is `'a`. The second is also pretty easy: as long as `z` is live, `'b` is too.

### Liveness for opaque types

What about opaque types? (You may know them as `impl Trait`.)

```rust
fn rpit_lt<'a, 'b>(x: &'a (), y: &'b mut ()) -> impl Sized + 'a + use<'a, 'b> {
    x
}
fn rpit_static<'a, 'b>(x: &'a (), y: &'b mut ()) -> impl Sized + 'static + use<'a, 'b> {
    ()
}
```

> **_NOTE:_**  I'm using an explicit [`use` bound][use-bound] to demonstrate the lifetime captures of the RPIT (return position impl trait) but starting in edition 2024, these captures are implicit.

The first signature basically says "the opaque type can capture any variable with either `'a` or `'b`, but it must outlive `'a`". The second says "the opaque type can capture any variable with `'a` or `'b`, but it must outlive `'static`". Also note that in neither example do we know if `'b` outlives `'a`.

Now, if this is confusing to you, then you're in good company. How can an opaque type be "allowed to capture a set of lifetimes" but simultaneously be required to outlive only one? Why is this useful to be able to say *anyways*?

For the question of "why do we need `use` bounds", I would direct you to read [this blog post][use-bound]. tl;dr we want you to be able to be precise in the lifetimes that can be captured in an `impl Trait`, particularly when the *default* is to capture everything.

For the first question, consider the following snippet:

```rust
let x = &();
let y = &mut ();
let a = rpit_lt(x, y);
let b = rpit_lt(x, y);
drop(a);
drop(b);
```

Now, think about this: by default, if we don't have the `'a` outlives bound, then the compiler must assume that both the lifetimes of `x` and `y` end up in the opaque type. So, from a compiler perspective, it's not much different than

```rust
let a = (x, y);
let b = (x, y);
drop(a);
drop(b);
```

Both `a` and `b` contain the same mutable reference and because you drop `a` while `b` is live, this results in a borrow-check error. The `'a` outlives bound makes the compiler treat it more like:

```rust
let a = (x,);
let b = (x,);
drop(a);
drop(b);
```

Only a shared reference is in both `a` and `b`, so the drop order doesn't matter and this compiles fine.

The `'static` bound says that the opaque type cannot capture *any* lifetimes, so the compiler treats it roughly like:

```rust
let a = ();
let b = ();
drop(a);
drop(b);
```

> Yes, you could achieve the same effect by changing the `use` bound to not name the lifetimes that can't be captured, but remember that this captures list is often *implicit*.

So, intuitively, let's think about liveness and opaque types:

```rust
let a = rpit_lt::<'x, 'y>(..);
let b = rpit_static::<'x, 'y>(..);
```

For `a`, because the opaque type outlives `'x`, then we *only* need to treat that as live. For `b`, because the opaque type outlives *neither* `'x` or `'y`, then we don't need to treat either as live.

### Problems

That's how it works on stable today (as of [#116733]). That's simple enough, right? Well, consider this variant:

```rust
fn rpit<'a, 'b>(x: &'a mut &'b ()) -> impl Sized + 'a + use<'a, 'b> {
    x
}
```

I want to point out that the opaque type captures both `'a` and `'b` because of the nested references. Because there is an implied `'b: 'a` outlives relationship, the outlives bound holds.

Let's look at a minimal example of how you can use (a slight variant of) this function to create an unsoundness with Polonius Alpha today:

```rust
trait Swap: Sized {
    fn swap(self, other: Self);
}

impl<T> Swap for &mut T {
    fn swap(self, other: Self) {
        std::mem::swap(self, other);
    }
}

fn hide_ref<'a, 'b, T: 'static>(x: &'a mut &'b T) -> impl Swap + 'a {
    x
}

fn dangle_ref() -> &'static [i32; 3] {
    let mut res = &[4, 5, 6];
    let x = [1, 2, 3];
    hide_ref(&mut res).swap(hide_ref(&mut &x));
    res
}
```

Let me point out the problem: in `dangle_ref`, we return a `'static` reference to a local value, which can then be read for a use-after-free.

On stable, [this fails][hidden-lifetimes-example] as expected: we don't treat `'b` as live, but it doesn't matter because NLL doesn't combine liveness with outlives relationships to perform borrow-checking (at least, not in a way that matters for this example). But *Polonius does*.

For the details on *why* Polonius needs liveness, I'd suggest that you dig into Niko's blog posts. In short, Polonius requires that for an outlives bound to be meaningful, variables *must be live* when the outlives bounds are required.

So, the fix here is to treat `'b` as live too. I'll talk about how we actually do that later in the post, and the rules that we use. But, in short: any lifetime must be considered live if it could be captured; and, we can identify that precise set by considering both the *item definition* and also *where clauses* in scope.

[use-bound]: https://blog.rust-lang.org/2024/09/05/impl-trait-capture-rules/#impl-traits-can-include-a-use-bound-to-specify-precisely-which-generic-types-and-lifetimes-they-use
[#116733]: https://github.com/rust-lang/rust/pull/116733
[hidden-lifetimes-example]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=f89c61363b55034aafe53627b88233d7

## That's a simple example, surely this isn't hard?

Oh, if only it was that simple. It turns out we also have to handle: associated types, rustc's odd opaque representation, RPITITs, and extra bounds from where clauses.

### Associated types!

So, `impl Trait` has this problem because they can have an outlives bound. Actually, associated types have a similar ability:

```rust
trait MyTrait {
    type Assoc<'a, 'b: 'a>: 'a
    where
        Self: 'a;
    fn changes<'a, 'b>(&'a self, x: &'b mut u8) -> Self::Assoc<'a, 'b>;
}
```

Surely you can see the parallel:

```rust
let t = make_something_that_impls_mytrait();
let x = &mut ();
let a = t.changes(x);
let b = t.changes(x);
drop(a);
drop(b);
```

We want this to compile, because of the `'a` outlives bound on `MyTrait::Assoc`. We know that the type cannot contain `x` (because it has `'b` which doesn't outlive `'a`); so in the example, there are no overlapping mutable references.


### Lifetime capture of `impl Trait` in rustc is *weird*

Another piece of complexity in this problem stems from how `impl Trait` is represented in rustc.

If you have:

```rust
fn foo<'a, 'b, T>(x: &'a mut &'b T) -> impl Sized + 'a + use<'a>;
```

That really looks something like:

```rust
opaque foo_opaque<'a1>: Sized + 'a1;
fn foo<'a, 'b, T>(x: &'a mut &'b T) -> foo::<'a, 'b, T>::foo_opaque<'a>;
```

There are two important considerations about the new "opaque" definition:
1) It gets the parent lifetimes copied to itself (but not type or const params).
2) All the information available to the parent (i.e. where clauses and the function signature) *don't* get copied.

The latter is *particularly* important, because the inputs and output of a function can provide *implied bounds* because those types must be *well-formed*. For the above, `&'a mut &'b T` must have been constructed, so callers of `foo` must prove that `'b: 'a` (also that `T: 'b`) and `foo` can just *assume* that holds.

### RPITITs (return position impl trait in traits)

So far when we've discussed RPITs, they've been present in freestanding functions. Turns out, RPITs in traits have a quirk: they are represented internally as (generic) associated types. Imagine you have a definition:

```rust
trait MyTrait {
    fn foo<'a>() -> impl OtherTrait + 'a;
}
```

That is represented as:

```rust
trait MyTrait {
    type FooRet<'a>: OtherTrait + 'a;
    fn foo<'a>() -> Self::FooRet<'a>;
}
```

Now, since we already need to apply a fix for associated types (see above), that seems fine. Right? Of course it's not that simple! But we'll get there later.

### Extra information from `where clauses`

Let's start with another example:

```rust
fn rpit<'a>(x: &'a mut ()) -> impl Sized + use<'a> {
    ()
}
```

This is an error today (and with Polonius on nightly, to be fair):

```rust
let x = &mut ();
let a = rpit(x);
let b = rpit(x);
drop(a);
drop(b);
```

This is an error for the same reasons we talked about before: the opaque type could contain the mutable reference. However, there is a nightly feature that allows you to name opaque types to use them in where clauses (return type notation, RTN). With that, you can add additional information about the opaque type:

```rust
fn foo() where rpit(..): 'static {
    let x = &mut ();
    let a = rpit(x);
    let b = rpit(x);
    drop(a);
    drop(b);
}
```

> **Note:** The RTN feature is *most useful* when used for *trait* methods, but the idea translates the same here.

We ideally would let this compile as-if the `'static` outlives bound was on the `impl Trait` itself.

Of course, you don't need RTN to add additional bounds for associated types, so this is something we need to consider even on stable today:

```rust
trait MyTrait {
    type Assoc<'a>
    where
        Self: 'a;
    fn foo<'a>(&'a mut self) -> Self::Assoc<'a>;
}

fn bar<T: MyTrait>(mut t: T)
where
    for<'a> T::Assoc<'a>: 'static,
{
    let a = t.foo();
    let b = t.foo();
}
```

## Though this be madness, yet there is method in’t

We've gone through what the source of the unsoundness is and the key considerations for what we need to keep in mind when fixing it. So, let's talk about the solution.

Remember, given an opaque type, we want to know what lifetimes can be contained inside it and we need to consider those *live* for borrowck (and elsewhere).

Let's go back to this example:

```rust
fn rpit<'a, 'b>(x: &'a mut &'b ()) -> impl Sized + 'a + use<'a, 'b> {
    x
}
```

Starting with the `'a` outlives bound: this tells us that the opaque type can *only* contain lifetimes that we *know* outlive `'a`. Or, put differently, any params that outlive `'a` *can* be contained within the opaque type (with a caveat below). And so, we must mark these lifetimes as *live*. This is the root of the unsoundness: we don't do that properly today.

> **Note:** It's not entirely true that we have to know *all* the params that outlive `'a` to be sound; we only need to know all the params that the *function* knows to outlive `'a` (otherwise the function would not allow that to be contained in the opaque type).

Now that caveat: `use<'a, 'b>` is the explicit precise capture list, but you can imagine that more often this is *implicitly* calculated. In either case, this gives us an *upper bound* on the params that we need to consider live; we don't need to consider *all* params that outlive `'a`, only those that can be captured.

So, how do we identify params that can be captured and outlive `'a`?

This is where `&'a mut &'b ()` comes in. Or, really, the *entire* function signature and its where clauses. As mentioned previously, this function's input argument tells us that `'b: 'a`. We combine these *implied* bounds from the function's signature with the *explicit* where clauses. Fortunately, there is existing code within the compiler to get all the explicit and implied bounds for a given item (in this case, for a given function).

Of course, there is a major consideration here: the opaque type has that weird representation where lifetime params are duplicated and where clauses are not copied. The fix is relatively simple *in theory*: we map the opaque's parameters back to the parent function, find the known outlives relationship *there*, and then map back.

So, to know what params to consider live on `rpit`, we do the following:

1) See that there is an outlives bound on `'a`.
2) Map that lifetime to the parent function.
3) For each param on the function (here `'a` and `'b`), use the where clauses and implied bounds to test if we know that it outlives `'a`. In this case, *both* do.
4) Map this larger set back to *captured* params on opaque (in this case, both `'a` and `'b` are captured).
5) Substitute the *actual* used values for the params.
6) Mark all lifetimes in these values as live.

Let's go through a much more complicated example. I'm adding a bunch of complexity here not just to the example, but to the explanation too. This much more closely matches the actual representation and implementation, but also means there are concepts I haven't otherwise covered. I'll do my best to explain:

```rust
fn rpit2<'a, 'b, 'c, 'd: 'b, U: 'a>(
    x: &'a &'b u8,
    y: &'c u8,
    z: U,
) -> impl Sized + 'a + use<'a, 'b, 'c, U> {
    (x, z)
}
```

Which gets represented and used like:

```rust
opaque rpit2_opaque<'a1, 'b1, 'c1>: Sized + 'a1;
fn rpit2_lowered<'a, 'b, 'c, 'd: 'b, U: 'a>(
    x: &'a &'b u8,
    y: &'c u8,
    z: U,
) -> rpit2::<'a, 'b, 'c, 'd, U>::rpit2_opaque<'a, 'b, 'c> {
    (x, z)
}

let s: OtherStruct<'_> = ...;
let out = rpit2(&&0, &0, s); // out has a type of:
//    `rpit2::<'?1, '?2, '?3, '?4, OtherStruct::<'?5>>::rpit2_opaque<'?6, '?7, '?8>`
// `'?X` is a *lifetime variable* and `?Y` is a *type variable*
```

1) We see that there is a `'a1` outlives bound on `rpit2_opaque`.
2) Map this back to `rpit2`: `'a1` matches `'a`.
3) Check outlives relationships:
    - `'a: 'a` trivially holds
    - `'b: 'a` holds because of `&'a &'b u8`
    - `'c: 'a` does not hold
    - `'d: 'a` holds because of the `'d: 'b` bound and `'b: 'a`
    - `U: 'a` holds because of the `U: 'a` bound
4) Map back to the captured opaque params:
    - `'a` => `'a1`
    - `'b` => `'b1`
    - `'d` => not captured
    - `U` => `U`
5) Substitute params with actual values:
    - `'a1` => `'?6`
    - `'b1` => `'?7`
    - `U` => `OtherStruct::<'?5>`
6) Mark lifetimes `'?6`, `'?7`, `'?5` as live

Before this change, we only would have marked `'?6` as live.

I also want to point out that the above can be generalized a bit: given *some* lifetime on the opaque, we can find the params that *must* be considered live (steps 2-4). This will be important later, but for now let's assume *this* is the generalization we want and work to apply it to other places.

> **Note:** Although RPITITs are internally represented as associated types, the information to calculate all the outlives relationships (namely the where clauses and the well-formed function signature) is not available. Therefore, the algorithm for calculating live params for RPITITs is the same as for RPITs.

### Associated types

Implementing this logic for associated types is easy.

>Fortunately, there is existing code within the compiler to get all the explicit and implied bounds for a given item.

This code *just works* for them.

```rust
trait MyTrait {
    type Assoc<'a, 'b: 'a, T: 'a>;
}
```

We can ask "what params outlive `'a`" and get back `'a`, `'b`, and `T`. We don't need to do any lifetime mapping tricks or filtering for captured params; those only matter for RPITs.

### Outlives bounds in where clauses

> **Warning:** this section is as complicated (if not more) as finding live params on the opaque or associated type definition.

Do you remember how you write today:

```rust
fn bar<T: MyTrait>(mut t: T)
where
    for<'a> T::Assoc<'a>: 'static,
{
    ...
}
```

And expect that `T::Assoc<'a>` can't *actually* contain anything that uses the lifetime param. However, here are *many* other examples to consider:

```rust
trait MyTrait { type Assoc<'a, 'b: 'a, 'c>; }

fn fn1<'x, 'y, 'z, T: MyTrait>() where
    T::Assoc<'x, 'y, 'z>: 'static,

fn fn2<'x, 'y, 'z, T: MyTrait>() where
    T::Assoc<'x, 'y, 'z>: 'x,

fn fn3<'x, 'y, 'z, T: MyTrait>() where
    T::Assoc<'x, 'y, 'z>: 'z,

fn fn4<'x, T: MyTrait>() where
    T::Assoc<'x, 'x, 'x>: 'x,

fn fn5<'x, 'y, T: MyTrait>() where
    for<'h> T::Assoc<'x, 'y, 'h>: 'h,

fn fn6<'x, T: MyTrait>() where
    for<'h> T::Assoc<'h, 'x, 'x>: 'x,

fn fn7<'x, T: MyTrait>() where
    for<'h> T::Assoc<'h, 'h, 'h>: 'x,

fn fn8<T: MyTrait>() where
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h,

fn fn9<T: MyTrait>() where
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h,
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'j,

fn fn10<T: MyTrait>() where
    for<'h> T::Assoc<'h, 'h, 'h>: 'h,

fn fn11<T: MyTrait>() where
    for<'h> T::Assoc<'h, 'h, 'h>: 'static,

fn fn12<T: MyTrait>() where
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'static,
```

That's a lot! But generally, there are a couple different questions to consider here:
  - Are the lifetimes in the associated type from the *function* or from a `for<...>`?
  - Is the outlived lifetime from the *function* or from a `for<...>` (or `'static`)?
  - What happens with multiple where clauses?
  - When the same lifetime is repeated in the associated type, what does that mean?

So, let's work through these, and see if we can develop an intuition about the general rule here. We'll start with actually backing up a bit, and thinking about what the implementation of the associated type could look like. All of these are "valid":

```rust
impl MyTrait for Foo {
    type Assoc<'a, 'b, 'c> = (); // No lifetimes used
    type Assoc<'a, 'b, 'c> = (&'a &'b (), &'c ()); // All used
    type Assoc<'a, 'b, 'c> = &'a &'b (); // Only some
    type Assoc<'a, 'b, 'c> = &'c (); // Only one
}
```

If you have a bound like `for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h`, that rules out the second and fourth definitions, because `'c: 'a` doesn't hold. It does not rule out the third, because we know that `'b: 'a` holds because of the trait definition.

Now, what about something like `fn2` above with the bound `T::Assoc<'x, 'y, 'z>: 'x`? Well, *free* lifetimes (i.e. defined on the function, not *within* the function) are *always* live. So, when all the lifetimes contained are *free*, then they are all *live*. What about a bound like `for<'h> T::Assoc<'x, 'h, 'x>: 'x`? Let's think about what that could mean if the impl definition looks like the third above:

```rust
fn foo<'x>()
where
    for<'h> T::Assoc<'x, 'h, 'x>: 'x,
{
    let val: T::Assoc<'x, '_, 'x> = &&(); // &'?1 &'?2 ();
}
```

Here, `'?2` must outlive `'?1` because of the well-formedness of the right side. *However*, `'?1` must *equal* `'x` because of the impl. Therefore, `'?2` must outlive `'x` and *also* be live.

What about this example, with the fourth impl definition:

```rust
fn bar<'a, 'b>(a: &'a &'a (), b: &'b ()) -> <Foo as MyTrait>::Assoc<'a, 'a, 'b> {
    b
}

fn foo<'x>()
where
    for<'h, 'i> <Foo as MyTrait>::Assoc<'h, 'i, 'x>: 'x,
{
    let a = &mut &();
    let b: &'x () = &();
    let val2 = bar(a, b);
    let val3 = bar(a, b);
    drop(val2);
    drop(val3);
}
```

In this case, because of the outlives bound we know that neither `'h` and `'i` can be in the associated type. So, we should be safe to only treat `'x` as live (which it already *must* be because it is *free*). *However*, we don't do that *today* (on stable); we treat everything as live. So, that isn't changed: when the outlives bound is a *free variable*, we don't restrict the things that *could* be live.

So, next example. We'll look at `fn10` from above:

```rust
fn fn10<T: MyTrait>() where
    for<'h> T::Assoc<'h, 'h, 'h>: 'h,
```

This means that *all* three lifetimes may appear in the associated type, and therefore must be considered live.

Of course, if we have `for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h`, then `'h` is obviously allowed to be contained in the associated type; but so is `'i`, because we know from the trait that the second lifetime outlives the first.

> **Note:** This doesn't *quite* work today, because you will get an error that we don't know that `'i: 'h`. But, we eventually want to make this work.

So, let's work through a general algorithm here, for the `for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h` where clause:
1) Identify the outlived lifetime. If it is `'static`, there are no live lifetimes. If it is free, all lifetimes are live.
    - The outlived lifetime is `'h`.
2) For each param in the where clause for the associated type, if it matches the outlived lifetime then map it to the lifetime on the associated type.
    - `'h` in `T::Assoc<'h, 'i, 'j>` corresponds with `'a` on the associated type.
3) For each lifetime on the associated type that may be live, find the transitive params that could *also* be live.
    - This uses the generalized algorithm from steps 2-4 earlier.
    - Here, that expanded set is `'a` and `'b`.
4) Substitute the actual values.
    - For `T::Assoc<'?1, '?2, '?3>`, map `'a` => `'?1` and `'b` => `'?2`.

Simple enough, right?

What about when there are multiple where clauses?

```rust
fn fn9<T: MyTrait>() where
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'h,
    for<'h, 'i, 'j> T::Assoc<'h, 'i, 'j>: 'j,
```

This answer *is* actually simple: each where clause *independently* restricts the set of live regions.

### Putting it all together

To reiterate and summarize the solution:
  - By default, all params substituted into an alias (either an associated type or an opaque) are treated as live
  - Outlives bounds on either the definition or in each in-scope where clause each *independently* restrict the set of params that are treated as live
  - The set of params treated as live is equal to the set of params that can outlive the outlives bound
  - Outlives relationships for opaques are calculated using the parent function's where clauses and well-formed inputs and output
  - Outlives relationships for associated types are calculated using in-scope bounds

## Conclusions

I'm hoping I didn't lose you. This is complicated, and there are some nuances that I didn't talk about (for example, free standing type aliases are *also* affected, albeit similar to associated types). [The PR](https://github.com/rust-lang/rust/pull/156027) with this fix is up, if you're curious to take a look. You might be interested to see some of the 8 new tests added that demonstrate the complexity (on top of the three existing ones), or read through the actual implementation (it's hopefully commented well enough).

As a final point: calculating liveness correctly for associated types and opaque types does have positive consequences outside the borrow-checker. For example, there are [open issues](https://github.com/rust-lang/rust/issues/42940) for where we incorrectly over-approximate live variables, because [attempting to be more correct](https://github.com/rust-lang/rust/pull/116040) was clearly unsound. The fix presented here allows us to *properly* calculate liveness and allow *more* code to compile (correctly).

With all hope, this will be landed soon and Polonius Alpha will be on its way to stabilization!
