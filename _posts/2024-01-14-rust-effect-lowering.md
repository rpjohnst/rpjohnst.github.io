---
title: A universal lowering strategy for control effects in Rust
---

The Rust language has incrementally grown a set of patterns to support *control-flow effects* including error handling, iteration, and asynchronous I/O. In [The registers of Rust][registers], boats lays out four aspects of this pattern shared by Rust's three effects. Today these effects are typically used in isolation, or at most combined in bespoke ways, but the Rust project has been working on ways to integrate them more deeply with each other, such as [`async gen` blocks][async-gen].

[registers]: https://without.boats/blog/the-registers-of-rust/
[async-gen]: https://github.com/rust-lang/rust/pull/118420

The theory of algebraic effects and handlers has explored this design space and offers answers to many of the questions that the Rust project has encountered during this work. This post will relate the patterns employed by Rust to the terminology and semantics of effects, to help build a shared vocabulary and understanding of the implications of combining multiple effects.

This connection suggests an API pattern that can be instantiated for arbitrary combinations of effects. It generalizes Rust's existing `Iterator` and `Future` traits in a way that supports existing Rust idioms while remaining a zero-cost abstraction by construction. (In other words, it works the same way as a specialized hand-written version, with no extra heap allocation or indirection.) It is also forward-compatible with, but does not require, hypothetical features like `async Drop`, (un)pinning, lending, or effect polymorphism.

### Some definitions and how they apply to Rust

Consider the platonic concept of a "function," independent of programming language. At this level, the idea of a function is defined primarily by how it interacts with the rest of the program, rather than by any particular syntax or implementation strategy. That is, when we call something a function, we are talking about a sub-program that can be applied to some argument values, at which point its parameters are bound to those values, and it runs to produce a result.

Algebraic effects are a similarly abstract idea. Where functions are defined in terms of parameters and application, effects are defined in terms of effect operations and handlers. Performing one of these operations suspends execution and transfers control flow to a handler somewhere up the call stack. The handler receives a first-class value representing the suspended computation (a "continuation"), which can be used to transfer control flow back and resume execution after the operation.

> #### Type and effect systems
>
> Effects and functions are both purely dynamic constructs. They can exist entirely without types, as in dynamically typed languages. However, effectful languages with a type system do often extend it to a "type and effect system" that tracks the set of effect operations each sub-program may perform. One reason to do this is to ensure that there is always an appropriate handler available when an operation is performed. Another reason is to drive code generation, potentially producing more efficient output when the compiler knows precisely which interactions are possible. However, this post focuses on the dynamic aspect of effects.

An effect can be described with a signature that specifies how values are passed back and forth between the effectful computation and its handler. For example, we might invent a notation where effect operations look like functions:

```rust
effect my_effect {
    operation_x(A) -> B;
}
```

The effectful computation performs `operation_x` by supplying an `A`. The handler receives this `A`, and to resume the computation it supplies a `B`, which appears as the result of `operation_x`. In our notation, this process looks like this:

```rust
fn my_computation() -> C {
    let b: B = operation_x(A { ... });

    C { ... }
}

fn main() {
    handle my_computation() {
        // The computation has performed an operation:
        operation_x(resume, a) => { resume(B { ... }) }

        // The computation has completed:
        return(c) => {}
    }
}
```

An effectful computation like this is a running sub-program that performs a sequence of effect operations. Importantly, for each operation in this sequence, the handler is free to choose when to resume the computation, and even whether to resume it at all. This is what distinguishes effect handlers from first class functions that must always resume their caller (or diverge).

By desugaring Rust programs into our notation, we can map Rust's concrete effects to these abstract definitions. This is not any kind of proposal for Rust itself---merely a concise way to describe Rust's behavior, before we consider its implementation.

Here is the signature of Rust's iteration effect:

```rust
effect gen<Item> {
    yield(Item) -> ();
}
```

The proposed `yield` operator is its one operation. It passes an `Item` to the handler, which resumes it with a `()`. The `for` loop is Rust's dedicated syntax for handling this effect. Here is its desugaring:

```rust
for item in some_iter() {
    loop_body(item);
}
```

```rust
handle some_iter() {
    yield(resume, item) => {
        loop_body(item);
        resume(())
    }
    return(()) => {}
}
```

The handler may choose not to resume the computation by `break`ing out of the loop. It may also choose to resume a different computation first, as with the `zip` combinator. Rust's choice of external iteration is equivalent to the first-class control flow of algebraic effects.

Here is the signature of Rust's failure effect:

```rust
effect try<E> {
    throw(E) -> !;
}
```

The computation passes an error `E` to the handler, which never resumes the computation. Rust has no dedicated syntax just for handling this effect, but it does have the `?` operator.

At the abstract level, the reason for this comes from how Rust chooses the boundary of a suspendable computation. Typical languages with effects treat the whole chain of stack frames between handler and operation as a single computation to be suspended and resumed all at once---for example, this is how exceptions (which share this effect signature) work in other languages. Rust instead treats individual function bodies and blocks as distinct computations to be handled individually.

To make this convenient, the `?` operator is used to *propagate* this effect across a chain of computations, until it reaches an actual handler. It behaves as a sort of no-op handler that immediately re-performs the effect:

```rust
try_work()?
```

```rust
handle try_work() {
    // Propagate the `try` effect by forwarding `throw` operations in `try_work`.
    throw(resume, e) => { throw(e) }
    return(x) => { x }
}
```

The signature of Rust's `async` effect is slightly more obscured, because of its flexibility for interacting with arbitrary external events. One possibility would be to give it multiple operations, like this:

```rust
effect async {
    read(File, &mut [u8]);
    write(File, &[u8]);
    // ...
}
```

However, this makes the handler responsible for implementing all these operations, while in Rust the set is open for extension. (It also ignores its interaction with the failure effect, addressed below.) A more precise signature, adapted from [Structured Asynchrony with Algebraic Effects][async-effect], looks like this:

[async-effect]: https://www.microsoft.com/en-us/research/wp-content/uploads/2017/05/asynceffects-msr-tr-2017-21.pdf

```rust
effect async {
    sleep(impl FnOnce(&Waker)) -> ();
}
```

Now the computation supplies the handler with a callback that arranges for the `Waker` to be signaled when the I/O completes. The `.await` operator is, like `?`, used for propagation:

```rust
async_work().await
```

```rust
handle async_work() {
    // Propagate the `async` effect by forwarding `sleep` operations in `async_work`.
    sleep(resume, schedule) => { resume(sleep(schedule)) }
    return(x) => { x }
}
```

### Combining effects

So far we have only considered computations with a single kind of effect, but many computations can perform multiple kinds of effects. For example, an iterator may perform some asynchronous I/O to produce its output, or an asynchronous computation may fail. The abstract definition of effects accommodates this without any trouble---operations from different effects simply transfer control to different handlers.

As a result, the combined effects of a computation form a flat, unordered set. This is an extremely useful property, because it means programs can deal with each effect uniformly and in isolation, regardless of how they are composed. Adding a new effect to a computation need not change how the rest of the computation is written, nor how its other effects are handled. This kind of portability is one of the Rust project's high-level goals for `async` (i.e. it is a generalization of portability between sync and async) making algebraic effects an attractive way to think about this language feature.

Consider the combination of `async` and `gen`. The first effect is handled by an executor while the second is handled by a `for` loop. The two handlers can be specified in either order, without changing anything about the computation, which freely interleaves `.await`s and `yield`s. For example, here is a desugaring for the proposed `for await` loop, handling the `gen` effect and propagating the `async` effect:

```rust
for await item in some_async_iter() {
    loop_body(item);
}
```

```rust
handle (handle some_async_iter() {
    yield(resume, item) => {
        loop_body(item);
        resume(())
    }
    return(()) => {}
}) {
    sleep(resume, schedule) => { resume(sleep(schedule)) }
    return(()) => {}
}
```

Conversely, here is a combinator that converts an asynchronous iterator to a synchronous one by handling the `async` effect and propagating the `gen` one:

```rust
handle (handle some_async_iter() {
    sleep(resume, schedule) => {
        /* ... block_on ... */
        resume(())
    }
    return(()) => {}
}) {
    yield(resume, item) => { resume(yield(item)) }
    return(()) => {}
}
```

Both of these can also be written with the opposite handler nesting, with the propagating handler on the inside and the non-propagating handler on the outside, without changing behavior. Handling one effect merely produces a larger computation with that effect removed from its set.

This commutativity has an important implication for how these imaginary `handle` expressions are evaluated. A good place to see this is the combination of `async` and `try`. An API like Tokio's [`AsyncReadExt::read`][async-read] may perform either of the `sleep` or `throw` operations. It is tempting to think in terms of an inherent layering, with an outer `Future` driven to completion to reach an inner `Result`, and indeed this is the only way to propagate these effects today: `f.read(buf).await?`. But this restriction is merely an artifact of how Rust lowers these effects, not something dictated by the effects themselves. For example, here is a desugaring for `f.read(buf)?.await`:

[async-read]: https://docs.rs/tokio/latest/tokio/io/trait.AsyncReadExt.html#method.read

```rust
handle (handle f.read(buf) {
    throw(resume, e) => { resume(throw(e)) }
    return(len) => { len }
}) {
    sleep(resume, schedule) => { resume(sleep(schedule)) }
    return(len) => { len }
}
```

The key here is that the scrutinee of a `handle` expression is not fully evaluated before the `handle` expression itself, the way a `match` scrutinee would be. Instead, the scrutinee and the `handle` arms are evaluated together, switching back and forth for effect operations. The `sleep` operation performed by `read` crosses the inner `handle` expression to reach the outer one in the same way that a `break` crosses any number of containing expressions to reach a loop, or a `return` crosses its containing expressions to reach a `fn`.

In most contexts, effectful computations in Rust "decay" to their lowering, such that expressions like `async { 42 }` or a call to an `async fn() -> i32` have type `impl Future<Output = i32>` rather than `i32`. But as the scrutinee of a `handler` for the `async` effect, these expressions remain un-decayed, and the outermost handler expression that removes the last effect becomes a non-effectful, non-decaying computation. Properly handling multiple effects means deferring this decay until the point that the program actually demands it, for a containing expression that is not a handler.

> If this deferred behavior feels strange or magical, consider the behavior of place and value expressions (also called lvalues and rvalues). A place expression like `obj.field`, `*expr`, or `slice[i]` does not immediately do anything beyond computing a location. The choice of what to do with that location is deferred until the containing expression demands a specific behavior: an address-of operator `&expr` will materialize the location as a reference; a call `f(expr)` will move from the location; an assignment `expr = v` will write to the location. Like reading vs writing, the choice of running vs lowering is a property of how the expression is used.

### How Rust implements effects

Rust implements effectful computations by lowering them to stackless coroutines. (More precisely, this is Rust's most general strategy; some effects use a subset or specialization of this approach.) The compiler transforms each computation into an object to hold its state while suspended, with a method to resume the computation. In Rust this object is often called a "state machine," but to avoid ambiguity I will use the more specific term *coroutine frame.*

In [Why async rust?][why-async], boats covers the history of and motivation for this choice: while typical implementations of effects use continuation passing or stackful coroutines, these implementations also introduce allocation and indirection that Rust cannot afford. Rust's coroutine frames exhibit two key characteristics that satisfy these requirements:
* They are ordinary value types, with a statically-knowable layout and statically-dispatchable resumption method, that can be composed into into larger computations that are themselves ordinary value types. The entire chain of frames, from handler to effect operation, uses only a single allocation which may come from anywhere---the heap, the stack, a global, a struct field, etc.
* They are mutated in-place, giving a stable address to the local variables of the computation. If the coroutine frame were re-constructed or moved at each suspension, then live references to locals would force those locals to live in a separate allocation despite the previous point.

[why-async]: https://without.boats/blog/why-async-rust/

Rust's lowering gives each effect a trait method corresponding to its signature. Effectful computations (encoded as coroutine frames) implement the trait; handlers invoke the method repeatedly and match on the result. Thus, an effect signature with operation `e(A) -> B` becomes a trait method `fn m(Pin<&mut Self>, B) -> Either<A, Output>` to be invoked until the computation completes by returning `Output`. This picture is complicated by three factors:
* This method is also used to kick off the computation before it has performed any effects. The `B` parameter does not make sense for this initial call. Because of this, Rust iterators and futures are said to be "lazy," while failure (with `B = !`) cannot use this method for its initial call and is "eager."
* Computations with the `async` effect do not literally pass an `impl FnOnce(&Waker)` to the handler. Instead, the handler preemptively supplies a `&Waker` as an additional parameter to the resumption method, and computations are expected to use it before transferring control to the handler. This relies on the point above---the initial part of the computation needs a `&Waker` just as later parts do.
* The trait for the iteration effect was designed long before the language added support for self-referential coroutine frames. The `Pin<&mut Self>` parameter becomes `&mut self` in this case.

The `Either<A, Output>` return type is also instantiated to a distinct `enum` for each effect, giving us the current form of Rust's three effect lowerings as specializations of this general recipe:

```rust
trait Iterator {
    type Item;

    // isomorphic to Either<Item, ()>
    fn next(&mut self) -> Option<Self::Item>;
}
```

```rust
trait Future {
    type Output;

    // isomorphic to Either<impl FnOnce(&Waker), Output>
    fn poll(Pin<&mut Self>, &mut Context<'_>) -> Poll<Self::Output>;
}
```

```rust
// isomorphic to Either<E, Output>
fn /* ... */ -> Result<Output, E>;
```

> #### An alternate history: zero-cost continuation passing
>
> With deeper language and compiler integration, a language could bring the efficiency of stackless coroutines to continuation passing style. The relevant insight is that the traditional call stack is already a form of continuation passing, restricted with linearity to enable contiguous allocation and in-place mutation. Restricting continuations further to a static depth makes their layout statically-knowable, in the same way as a chain of coroutine frames.
>
> Each suspension point can be lowered to a distinct closure-like anonymous struct holding live values, with a dynamically-sized tail representing its chain of callers. Each struct's `resume` method ends with an adjustment to the continuation followed by a tail-call to a callee, a caller, or a handler. The handler supplies a fixed-size space from which to allocate these objects, and each leaf suspension point lends its object back to the handler.
>
> This approach has several nice properties:
> * Resumption bypasses the chain of callers, jumping directly to the leaf frame.
> * Propagation operators like `?` or `.await` do not generate any glue code.
> * `Waker`s are not constructed or passed around until an effect is performed.
> * The type system prevents handlers from resuming completed computations.
>
> Rust's coroutine frames are essentially an emulation of this approach, necessitated by the language's lack of tail calls. The complications above are part of this emulation, not essential to zero-cost effects.

### A detour: combining effects by layering

Because these lowerings are so specialized, it is not always obvious how they should be combined. Haskell lowers effects in a way that is related to Rust: effectful computations are defined in terms of an ordinary in-language interface, making the lowering observable to other parts of the program. This interface, the `Monad` typeclass, is implemented by the handler rather than the computation, and has what Rust would call an `and_then` method.

> One other interesting detail that is not directly relevant here: while Rust's effectful computations can be reused with any handler, monadic programs essentially bake in their choice of handler by default. It is possible to write them in a way that does not do this, e.g. by using a particular implementation of the `Monad` typeclass (known as the "free monad") that saves all the invocations of `and_then` in an AST-like structure to be interpreted later by an arbitrary handler.

For computations with multiple effects, Haskell makes this `and_then` method generic over a "wrapper" monad representing an outer handler. These generics, or "monad transformers," can then be nested in arbitrary ways. Unfortunately, this introduces an arbitrary ordering to the set of effects, which infects both computations and handlers. Handlers must implement a `lift` method alongside their generic `and_then`, and computations must use the appropriate number of `lift` calls when performing an outer effect operation.

Notably, this layering approach is very similar to the one proposed by Rust's keyword generics initiative. There, traits like `Iterator` would become generic over any additional effects their method might perform, supporting instantiations like `async Iterator`. This has a similar unfortunate implication for handlers, which must use an appropriate number of nested `map` calls to get at the representation of the effect they care about.

This layering also interferes with Rust's lowering of coroutine frames. While Haskell's monadic computations pass a new closure to the handler for each place they can resume, Rust relies on these continuations (represented as coroutine frames) being used linearly so it can mutate them in-place. When the computation is lowered one effect at a time, each effect is forced to produce a separate coroutine frame to satisfy the API. Each frame must borrow from the next, and must be reconstructed after any of the frames it borrows from is resumed.

When a coroutine frame is used as a trait object, things get even worse. An effectful resumption method returns an associated type for the next frame in the chain---where should it be allocated? The number of resumption calls increases from one per effect operation to, at worst, the total number of effects *times* the number of effect operations---are these all dynamically dispatched? It is possible to mitigate these costs by reserving enough space for the whole chain of frames up front, and simultaneously monomorphizing a new method that dispatches the chain of resumptions and reconstructions statically, but this is not something we can ever expect the compiler to do automatically, because it changes observable behavior and the type of the trait object.

Do we at least get any benefits in exchange for dealing with this complexity everywhere? No---neither the computation nor the handler can really do anything with the layering, because it carries no semantic meaning and is purely an artifact of the lowering. Conversely, if we want it for any reason, we can always recover this layering from an un-layered lowering, by exploiting the ability to handle algebraic effects in any order we choose! For example, we can produce an `async fn next` from a lowered `poll_next`, and vice versa produce a `poll_next` from a hand-written `async fn next`, with just a few lines of code.

> This is not to say that effectful traits are necessarily a bad idea in general. Traits like `Read` and `Write` do not represent coroutine frames, and thus do not run into these problems in quite the same way. But this post is about how effects are lowered, not general API design.

Precisely because of this sort of type tetris, the Haskell ecosystem has grown several libraries designed to represent effects without this layering, and the Haskell compiler recently [gained support][delim] for delimited continuations to make these kinds of libraries more efficient. Rust should learn from this work and skip the layering.

[delim]: https://gitlab.haskell.org/ghc/ghc/-/merge_requests/7942

### Lowering combined effects

A running computation produces a sequence of effects drawn from a set. Its handler (or equivalently, set of handlers) must be prepared to accept any of them. Rust's single-effect lowering already uses an `enum` with one variant for the effect operation and another to signal that the operation has completed. This naturally extends to a larger `enum` with one variant for each effect, plus one for the initial call and final return.

Given a set of effect signatures, their combined lowering looks like this:

```rust
effect eff1 { op1(A) -> B; }
effect eff2 { op2(C) -> D; }
// ...

trait /* ... */ {
    type Output;
    fn resume(Pin<&mut Self>, Either<(), B, D>) -> Either<A, C, Self::Output>;
}
```

> This does lean further into the design choice not to enforce appropriate use of this protocol at the type level. Just as it is possible to call `Iterator::next` after it has returned `None`, it is possible to pass the wrong variant to `resume`. It is, of course, possible to prevent this with increasingly-elaborate type system machinery like ZST tokens and existential or generative lifetimes, but that is really beside the point here.

The `AsyncIterator` (formerly known as `Stream`) trait which combines the `async` and `gen` effects is, once again, a specialization of this pattern:

```rust
trait AsyncIterator {
    type Item;
    fn poll_next(Pin<&mut Self>, &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

It can be tempting to read a return type like `Poll<Option<Item>>` as more nesting, especially when Rust programmers are so used to simply wrapping `Output` in a `Result` when writing the `try` lowering by hand, but this is a red herring. `Poll<Option<Item>>` is isomorphic to an enum with the three variants `Pending` (the `sleep` operation), `Some(Item)` (the `yield` operation), and `None` (a completed computation). The general recipe is to add more variants, not to add more layers.

Similarly, `impl Future<Output = Result>` has a return type `Poll<Result<Output, E>>` which is isomorphic to the three variants `Pending` (`sleep`), `Err(E)` (`throw`), and `Ok(Output)` (completion). Even `impl Iterator<Item = Result>`'s return type `Option<Result<Item, E>>` is isomorphic to the three variants `Ok(Item)` (`yield`), `Err(E)` (`throw`), and `None` (completion), though this one collides with the type of an iterator yielding `Result`s.

This recipe is not merely a "god trait" that indiscriminately accumulates everything we need in one place. It is a more faithful representation of the commutative semantics of effects and handlers, which avoids the complexities that come from layering and restores the flexibility to handle effects without regard for where they sit in an irrelevant ordering.

Other variations we might want, like (un)pinning or lending, are not themselves effects, but aspects or modes of effects that involve the coroutine frame. If we extend our notation to include an explicit continuation type (analogous to an explicit receiver type like `&self`), we can express these variations as part of an effect's signature.

For example, we might write the un-pinned iteration effect like this:

```rust
effect gen<Item> {
    yield(Item) -> (&mut resume, ());
}
```

Similarly, we might write a pinned, lending iteration effect like this:

```rust
effect gen<Item<'a>> {
    yield<'a>(Item<'a>) -> (Pin<&'a mut resume>, ());
}
```

Combining these continuation types calls for a different approach than adding variants. Because `Pin` guarantees that its target stays pinned, we can't use an `enum` with `&mut self` and `Pin<&mut Self>` variants (though this might work in another language). Instead, we need a single receiver type that combines the restrictions of all the effects. If any of them are pinned, use `Pin`; if any of them are lending, add a lifetime. For example, this justifies the use of `Pin<&mut Self>` for async iterators---the iteration effect signature does not use pinning but the async one does.

Finally, the use of a single coroutine frame also simplifies the design of features like `async Drop` or `poll_progress`, which do interesting things to the effectful computation's control flow graph. Where a chain of layered frames would require the cooperation of the handler to propagate these signals across the layers, the states of a single combined coroutine frame represent the whole control flow graph in one place.

### Future possibilities

One of the motivations for the keyword generics initiative to consider the layered approach is a desire to avoid a combinatorial explosion of traits. The un-layered lowering already reduces this number because it does not differentiate between orderings. But if necessary, it can go even further than the `async Iterator` approach by using more modest and more widely-applicable extensions to the language, many of which are already implemented on nightly or have been repeatedly proposed. Allow me to engage in some wild speculation and play fast and loose with syntax.

As Rust's existing effect traits are already instantiations of the pattern described above, they could be given bridge `impl`s to or from a generic trait along the lines of the unstable `Coroutine`. Going further, they could be made into aliases of that trait, using something along the lines of the unstable trait alias feature. Doing this compatibly would additionally require the ability to `impl` a trait alias, as well as the ability to remap associated types and method names the way same aliases can remap generic parameters.

Is replacing a combinatorial explosion of traits with a combinatorial explosion of aliases good enough? Maybe! But the cosmetics of these aliases could be improved in a similar way to the `Fn` trait bound sugar. Instead of using names like `AsyncIterator`, the appropriate instantiation of the general trait could be referred to using its effect keywords. For example, `async gen<I> ()` would mean `AsyncIterator<Item = I>` or equivalently `Coroutine<Yield = Pending + Some(Item) + None>`, while `try<E> async T` would mean `Future<Output = Result<T, E>>` or equivalently `Coroutine<Yield = Pending + Err(E) + Ok(T)>`. This treats all the effects uniformly, unlike `async Iterator` which singles one out as the base trait.

What about sharing combinators across multiple instantiations? This would involve being generic over some of the variants while specifying others concretely. We can do this with another language extension called open `enum`s, sometimes known as open polymorphic variants, and related to the existing `#[non_exhaustive]` feature. For example, here is `Iterator::map` generalized to any computation with the `yield` effect, using the notation `Ts...` to declare an open `enum` generic parameter, and `A + Bs` to add a variant to one:

```rust
trait Coroutine {
    type Resume...;
    type Yield...;
    type Output;
    fn resume(Pin<&mut Self>, () + Self::Resume) -> Self::Yield + Self::Output;
}

impl<I: Coroutine, F, B> Coroutine for Map<I, F> where
    F: FnMut(I::Item) -> B,
{
    type Resume = I::Resume;
    type Yield = Option<B>::Some + I::Yield;
    type Output = I::Output;

    fn resume(self: Pin<&mut Self>, r: () + Self::Resume) -> Self::Yield + Self::Output {
        match self.project().iter.resume(r) {
            Some(item) => { Some((self.f)(item)) }
            y => { y }
        }
    }
}
```

Another sense in which a function can be polymorphic over effects is to support effectful closures. Here is `Iterator::map` generalized in this way instead:

```rust
impl<I: Iterator, F, B: Coroutine> Coroutine for Map<I, F> where
    F: FnMut(I::Item) -> B,
{
    type Resume = () + B::Resume;
    type Yield = Option<B::Output>::Some + B::Yield;
    type Output = Option<B::Output>::None;

    fn resume(self: Pin<&mut Self>, r: () + Self::Resume) -> Self::Yield + Self::Output {
        let state = self.project().state;
        match state {
            State::Done => { panic!() }
            State::Iter => match self.project().iter.next() {
                None => { *state = State::Done; None }
                Some(item) => { *state = State::Call(f(item)); self.resume(()) }
            }
            State::Call(c) => match c.resume(r) {
                item: B::Output => { *state = State::Iter; Some(item) }
                y => { y }
            }
        }
    }
}
```

Of course, there is little reason to write either of these by hand when we have the ability to handle effects selectively, so here is a version generalized in both ways simultaneously, using one final language extension in the form of a `do` operator for propagating unknown effects:

```rust
gen do {
    for do item in /* ... */ {
        yield f(item).do;
    }
}
```

For good measure, here it is again using `handle` expressions:

```rust
handle /* ... */ {
    yield(resume, item) => handle f(item) {
        eff(resume, x) => { resume(eff(x)) }
        return(x) => { resume(yield(x)) }
    }
    eff(resume, x) => { resume(eff(x)) }
    return(x) => { x }
}
```

At this point, I am still unconvinced that this sort of generality is actually important for Rust. My point is rather that the un-layered effect lowering strategy (represented by `poll_next`) is forward-compatible with mechanisms for reducing trait proliferation, and indeed that it does this more directly than the layered strategy. This is what algebraic effects were designed for, after all.

### Conclusion

Algebraic effects are an attractive framework for designing these features of Rust, because they are more composable than approaches like monad transformers or effectful coroutine traits. Many of their affordances come from the way that effects are *commutative* and can be handled in any order. Languages like Haskell have already covered similar ground. We can learn from their experience to unblock features like `async gen` and `for await` now, while retaining a path toward greater flexibility.