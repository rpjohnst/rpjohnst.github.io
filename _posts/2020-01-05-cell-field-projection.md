---
title: "Safe Cell field projection in Rust"
---

I've just published the [`dioptre`][dioptre] crate, the newest addition to [Driveyard][driveyard]. Dioptre is a lightweight proc-macro for struct field reflection. The subject of this post is a trick built on top of this functionality---`Cell` field projection, going from `Cell<Struct>` to `Cell<Field>`.

[dioptre]: https://crates.io/crates/dioptre
[driveyard]: https://github.com/driveyard/driveyard

### Interior mutability

One reason people wind up "fighting the borrow checker" in Rust is its strict rules around pointer aliasing. The two built-in reference types, `&T` and `&mut T`, allow either a single mutable `&mut T`, or multiple immutable `&T`s. Rust uses these rules to preserve memory safety without requiring a garbage collector. Other languages typically also allow mutation of shared (or *aliased*) objects instead. (See [The Problem With Single-threaded Shared Mutability][shared-mutability] or [Rust: A unique perspective][unique-perspective] for a more detailed look at these rules.)

Sometimes this design is more strict than it is useful, so Rust also provides *interior mutability*, which relaxes the immutability of `&T` references in various ways, while still preserving memory safety. (If this feels like a Rust-specific complication just to reach other languages' baseline, [C++ has a similar feature][mutable]!)

[shared-mutability]: https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/
[unique-perspective]: https://limpet.net/mbrubeck/2019/02/07/rust-a-unique-perspective.html
[mutable]: https://en.cppreference.com/w/cpp/language/cv#mutable_specifier

The most recognizable example of interior mutability is [`Mutex`][mutex], which lets multiple threads communicate with each other by sharing access to a mutable object. [`OnceCell`][oncecell] provides one-time initialization of otherwise-immutable data. Rust beginners frustrated with the `&T`/`&mut T` rules are commonly directed to [`RefCell`][refcell] (or worse, the soup that is `Rc<RefCell<T>>`), a single-threaded variation on `Mutex` that lets them continue to use Rust's built-in `&mut T` with shared data.

[mutex]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[oncecell]: https://docs.rs/once_cell/1.2.0/once_cell/sync/struct.OnceCell.html

### Cell and its upgrades

When all you want are a few references to a mutable object, there is a simpler alternative. The [`Cell`](cell) type started its life providing a very simple form of interior mutability. You could wrap it around a small `Copy` type to give it `&T`-based `get` and `set` methods, in exchange for giving up direct access to the object itself. For example, [`Rc`][rc]'s internal reference count field must be mutable so it can be updated when references are created or dropped, but it is also inherently shared, so that field is represented as a `Cell<usize>`.

[cell]: https://doc.rust-lang.org/std/cell/struct.Cell.html
[rc]: https://doc.rust-lang.org/std/rc/struct.Rc.html

Over time `Cell` has grown a few new abilities. [RFC 1651][rfc-1651] extended it to work with non-`Copy` types---the only things relying on `Copy` were `Cell::get` and traits like `Eq`, but `Cell` can still provide methods like `set`, `replace`, `take`, and `into_inner`.

[rfc-1651]: https://github.com/rust-lang/rfcs/blob/master/text/1651-movecell.md

Even more interesting, [RFC 1789](rfc-1789) extended `Cell` to support *references to the interior of the wrapped object*. This was originally forbidden because writing to the outer `Cell` would overwrite the target of the reference... but if the target object is also wrapped in a `Cell`, that's just fine!  Specifically, RFC 1789 provides a conversion from `&Cell<[E]>` to `&[Cell<E>]`, by guaranteeing that `T` and `Cell<T>` have the same memory layout.

[rfc-1789]: https://github.com/rust-lang/rfcs/blob/master/text/1789-as-cell.md

### Field projection

RFC 1789's `Cell` indexing operations are a form of *projection*, extracting a piece of an object from the whole. The most familiar form of projection operates on `struct` types, using the `foo.bar` syntax. Can we extend this to `Cell`-wrapped objects? This would let us write our type definitions in an idiomatic style, and only bring in `Cell` in parts of the program that need it.

Here is an example of what we might want to do, based on an example from the RFC:

```rust
// Just a normal Rust type, no interior mutability in sight.
#[derive(Copy, Clone)]
struct Point {
    x: f32,
    y: f32,
}

// Update some points in-place.
fn process(data: &mut [Point]) {
    for i in data {
        for j in data {
            // Set some fields of i and j:
            j.x = foo(i, j);
            i.y = bar(i, j);
        }
    }
}
```

Without interior mutability, this will not compile:

```
error[E0382]: use of moved value: `data`
  --> src/lib.rs:11:18
   |
9  | fn process(data: &mut [Point]) {
   |            ---- move occurs because `data` has type `&mut [Point]`, which does not implement the `Copy` trait
10 |     for i in data {
   |              ---- value moved here
11 |         for j in data {
   |                  ^^^^ value moved here, in previous iteration of loop
```

We could work around this by switching from iterators to explicit indexing:

```rust
for i in 0..data.len() {
    for j in 0..data.len() {
        data[j].x = foo(&data[i], &data[j]);
        data[i].y = bar(&data[i], &data[j]);
    }
}
```

However, this is not ideal. It may (for a given program) make it harder to ensure array indices stay in-bounds, or it may make it harder for the compiler to optimize out bounds checks.

#### Cell fields

RFC 1789 enables this approach:

```rust
// Switch from a `&mut [Point]` to a `&Cell<[Point]>`,
// so we can have multiple references:
let data = Cell::from_mut(data);

for i in &data[..] {
    for j in &data[..] {
        // ...
    }
}
```

The type of `i` and `j` is now `&Cell<Point>`, for which the usual field projection syntax does not work. But with a little help from `dioptre`, we can write this instead:

```rust
use dioptre::{Fields, ext::CellExt};

// Derive `dioptre::Fields`:
#[derive(Copy, Clone, Fields)]
struct Point {
    x: f32,
    y: f32,
}

// Step into `i` and `j` to get at their fields as `Cell<f32>`s:
j.project(Point::x).set(...);
i.project(Point::y).set(...);
```

#### Split borrows

As an extra note, because we're allowed to have multiple shared references into a struct, we don't need any special support for obtaining references to disjoint fields, the way we would for `&mut T` references:

```rust
fn split(point: &Cell<Point>) -> (&Cell<f32>, &Cell<f32>) {
    (point.project(Point::x), point.project(Point::y))
}
```
