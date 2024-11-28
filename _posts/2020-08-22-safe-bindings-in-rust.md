---
title: The problem of safe FFI bindings in Rust
---

Google has published a document on [Rust and C++ interoperability][chromium-interop] in the context of Chromium. It describes their criteria for the experience of calling C++ from Rust --- minimal use of `unsafe`, no boilerplate beyond existing C++ declarations, and broad support for existing Chromium types.

[chromium-interop]: https://www.chromium.org/Home/chromium-security/memory-safety/rust-and-c-interoperability

The response to this document has included a lot of confusion and misinformation. Really, this is a continuation of a larger ongoing discussion of how to use `unsafe` properly. Rust FFI and `unsafe` are complicated and often subtle topics, so here I am going to attempt to summarize the issues and suggest how we might address them.

## What is `unsafe`, exactly?

[Rust]'s core value proposition is to "guarantee memory-safety and thread-safety --- enabling you to eliminate many classes of bugs at compile time," without sacrificing the ability to target the same use cases as C++ (namely: performance-critical, embedded, and/or integrated with other languages).

[rust]: https://www.rust-lang.org/

This memory and thread-safety (henceforth just "safety") is normally enforced automatically by the compiler. However, Rust includes several features that are *not* checked in this way, including raw pointers, global mutable state, untagged unions, and (notably) the ability to call functions written in other languages. These features can only be used within an `unsafe` block:

```rust
// This function accepts a raw pointer, which may be null, misaligned, dangling, etc.
fn read(address: *const i32) {
    // The compiler rejects this:
    let data = *address;

    // The compiler accepts this:
    unsafe { let data = *address; }
}
```

Rust provides these operations with the intent that they will be used *only* as building blocks for safe APIs. In other words, while the compiler does not enforce them, there *are* a set of rules governing the safe use of these operations.

Thus, an `unsafe` block is more than a mere warning that "this program might crash or corrupt the heap." It is a marker that the program author has checked that they are following those rules! This is why `unsafe` is not "viral," and we can still have safe Rust programs even with `unsafe` in the standard library:

**The most important part of an `unsafe` block is the surrounding context that ensures its contents will never break those rules, no matter how it is used.** (The question of ["how much context?"][scope-of-unsafe] is an interesting one. It's a bit of a tangent to this post, but the answer is generally based on module privacy.)

[scope-of-unsafe]: https://www.ralfj.de/blog/2016/01/09/the-scope-of-unsafe.html

## How can calling C++ ever be safe?

When a Rust program calls a foreign function (that is, one written in another language like C++), there are several ways things can go wrong, breaking Rust's memory and thread-safety promise. We can split them into two failure modes, which the surrounding context will need to prevent:

* Each foreign function's type signature must be declared in the calling Rust program, because the compiler doesn't read any external formats like C++ header files.

  This means the signature may not match the actual function, in which case the compiler produces garbage, and the call will probably corrupt the stack or similar.

* Each foreign function must be called following certain rules (e.g. "only call this once," "only call this from one thread at a time," "don't pass a null pointer," "this array argument must have at least N elements").

  This means arguments or program state may not be valid, in which case the foreign function itself will probably do something to violate memory or thread-safety.

Writing matching type signatures by hand can be quite mechanical, tedious, and error-prone. When those signatures are simple enough, tools like [`bindgen`][bindgen] can instead automatically generate them from C or C++ headers. This eliminates a whole class of bugs!

[bindgen]: https://github.com/rust-lang/rust-bindgen

However, C++ type signatures encode more information than `bindgen` understands, including *ownership* information --- who is responsible for freeing the object, and when and how should they do so? The [`cxx`][cxx] binding generator understands several standard C++ "vocabulary" types like `std::vector` and `std::unique_ptr`, expanding the class of bugs we can eliminate.

[cxx]: https://github.com/dtolnay/cxx/

The caller of a foreign function may not need to worry about the first failure mode, if they can use a tool like `bindgen` or `cxx`, but the second failure mode is more fundamental. They must inspect the foreign function and its documentation, and then add static or dynamic checks that ensure safe Rust programs can't violate that function's safety rules.

At this point, **we can be sure that the Rust program is safe by inspecting these three things:**

* The foreign function: it must actually be safe to call when following the rules enforced on the Rust side.
* The Rust call site: it must actually enforce the rules required by the foreign function.
* The binding generator tool: it must correctly translate the foreign language's type signatures to Rust.

The whole *rest* of the Rust program no longer matters at all, because it's already checked by the Rust compiler.

## So what's the problem?

After `cxx`'s announcement, it immediately received an issue titled ["Calling from Rust to C++ is not safe"][cxx-issue], which cut directly to the core of this discussion: `cxx` generates wrapper functions that make the `unsafe` foreign function calls for you, so a Rust program can call C++ without the literal text `unsafe` anywhere in its source code.

[cxx-issue]: https://github.com/dtolnay/cxx/issues/1

It's important to approach this with the realization that this particular aspect of `cxx` was *not* a new development. From the start, the Rust ecosystem has relied on macros and build scripts that generate `unsafe` blocks. Some of these macros are always safe to use, containing the blocks' entire "safety context," while others expect their users to do some extra safety checking on their own.

When a Rust function requires its callers to do some extra safety checking, you can add the `unsafe` keyword to its declaration, which makes it into a new unsafe operation that can only be invoked from another `unsafe` block:

```rust
// This function can only be called from an `unsafe` block:
unsafe fn read(address: *const i32) { .. }

fn caller() {
    let x = 5;

    // This is only okay if `&x` is valid for whatever `read` does with `address`:
    unsafe { read(&x); }
}
```

In a sense, `unsafe` functions like this are the *exact opposite* of `unsafe` blocks. An `unsafe` *block* says "I've ensured that these operations are always safe." An `unsafe` *function* says "The caller is responsible for ensuring this operation is always safe." Ideally, both should carry comments justifying their correctness --- on blocks, pointing out the safety checks; on functions, describing the rules the caller must follow.

However, code generation happens before and outside of this whole system! There's no way to mark a macro or build script as an `unsafe` operation. This is the real question here: **Where do we put the comments justifying that generated, safe wrapper functions enforce the foreign functions' safety rules, and how would an audit know to look for them?**

## What are our options?

I begin with a couple of obligatory non-options:

> Why don't you want `unsafe` blocks at all your FFI call sites? Isn't the whole point of Rust to be safe by default, and mark all the exceptions to make debugging easier?

This is deeply unsatisfying, and flies in the face of Rust's usual approach to language design: "eat your cake and have it too!" We should be able to isolate most FFI-related unsafety to the FFI layer. To quote Google's first requirement: "[`unsafe`] should be restricted to patches of genuinely unsafe Rust code, and for C++ interoperability code where there's shared ownership or other complexities."

There's also the inverse, though I see it as a straw man more than a real argument:

> Our APIs are already pretty easy to use correctly, let's just mark them safe and move on.

This is also deeply unsatsifying. Even in C++, we should be aware of our APIs' rules and the ways people might accidentally break them. Preventing those mistakes is the *true* point of Rust's safety, so if you would rather debug recurring memory corruption than translate those rules to Rust, either the API needs improvement or Rust is failing.

A couple of more realistic options:

### Hand-written wrappers

You might take the time-honored approach of a typical "idiomatic Rust FFI wrapper" crate: use a binding generator however you like, but don't make its output public. Instead, hand-write the crate's public API from scratch.

This often works well! Plenty of libraries are small enough, the [`-sys` crate convention][sys] lets people work around the idiomatic wrapper if necessary, and there is an obvious place to deal with extra rules from foreign code. However, this duplication of API design is not always scalable.

[sys]: https://doc.rust-lang.org/cargo/reference/build-scripts.html#-sys-packages

It may also be the case that the foreign API is not conducive to being safely wrapped. Here it may be better to leave the bindings as `unsafe`, and build a safe API at a higher, and more application-specific, level.

### At last, generated wrappers

You might instead come up with a convention for "`unsafe` macros." Put "unsafe" in their name, or require users to pass the `unsafe` at an appropriate position in the macro's input. [This is likely to show up in `cxx`][cxx-unsafe] at some point.

[cxx-unsafe]: https://news.ycombinator.com/item?id=24244121

This way you can still "grep for `unsafe`" and find the binding generator invocation, like we wanted. It amounts to asserting that all the foreign functions you're binding to have no extra safety rules, which (believe it or not) *is* sometimes the case.

You might further tweak the binding generator to let you mark individual wrappers as `unsafe`, as an opt-out for those cases "where," as Google puts it, "there's shared ownership or other complexities."

Another approach comes from Microsoft, in the [`winrt-rs`][winrt-rs] crate. WinRT defines a whole new type system, designed to be easy to "project" into various languages, including C#, Javascript, C++, and now Rust. It also defines a file format to describe APIs in this type system. So long as it fits into the WinRT type system, a library written in one language can be used from any other WinRT language, with safe, automatically-generated bindings.

[winrt-rs]: https://github.com/microsoft/winrt-rs

## Where's the fire?

We have the technology to generate correctly-safe bindings for large C++ APIs, assuming most of those APIs limit their safety rules to things that can be inferred (by definition or by convention) from their type signatures. The remaining work, for the Chromium team behind their interoperability document, will be to validate that assumption for the APIs they call from Rust.

Does the presence of a tool like `cxx` (or, closer to their requirements, [`autocxx`][autocxx]) make this inspection more or less likely to happen? How do the defaults of such a tool play into this? Does it matter?

[autocxx]: https://github.com/google/autocxx

I believe that these kinds of tools can only help, by eliminating the mental overhead of more mechanical and tedious aspects of FFI. With the minor tweak of making themselves "`unsafe` greppable," I see no technical cause for alarm.

If someone is inclined to take the YOLO approach of throwing `cxx` at whole C++ APIs with no further thought, then no amount of obfuscatory requirement for "more `unsafe`!" is going to dissuade them --- though it may convince them to give up on Rust.

On the other hand, if someone is unaware of their obligation to make these checks, then we can hardly blame the tool's mere existence --- it is instead a failure of the onboarding process, documentation, error messages, and so forth.

Rust has always been built on safe wrappers around unsafe code; all that's new here is the scale of those wrappers. So if you find yourself wringing your hands about this whole direction and how it might corrupt the Rust ecosystem, I suggest instead putting some thought into how we might make that scale easier to manage.