---
title: "Soak: a struct-of-arrays library in Rust"
---

Arrays are a natural way to work with collections of objects. They have relatively effective cache utilization, they make iteration and random access easy, and they tend to have good language support. In Rust, arrays lay out each individual object's fields together in memory, as an "array of structs:"

```rust
struct GameObject {
    position: Vec2,
    velocity: Vec2,
    health: u32
}

let vec: &[GameObject] = /* ... */;
```

```
[ ptr | len ]
   |
   v
[ position | velocity | health | position | velocity | health | ... ]
```

Often, you want to lay things out differently, as a "struct of arrays:"

```
[ ptr | ptr | ptr | len ]
   |     |     |
   |     |     +------------------------------------------------+
   |     |                                                      |
   |     +-------------------------+                            |
   v                               v                            v
[ position | position | ... ] [ velocity | velocity | ... ] [ health | health | ... ]
```

There are two major reasons you might do this:

* The CPU cache works in units of fixed-size cache lines. When the program iterates through an array but only needs some of each object's fields, the other fields waste memory bandwidth and cache space. This layout fills up whole cache lines with only the fields that are needed.
* [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions perform the same operation on multiple values in parallel. This layout allows the CPU to handle data from multiple objects at once when transferring it into and out of SIMD registers.

### Struct-of-arrays in Rust

It's straightforward enough to do this in Rust by allocating a separate array for each field:

```rust
struct GameObject {
    positions: Vec<Vec2>,
    velocities: Vec<Vec2>,
    healths: Vec<u32>,
}
```

However, this has some downsides. Each array becomes a separate allocation, with a separate length and capacity that can get out of sync with the others. Instead, in typical implementations (at least in games, from where I'm familiar with the technique), the arrays are combined into a single allocation:

```rust
struct GameObjects {
    positions: *mut Vec2,
    velocities: *mut Vec2,
    healths: *mut u32,
    len: usize,
}

let position_size = align_to(len * mem::size_of::<Vec2>(), align);
let velocity_size = align_to(len * mem::size_of::<Vec2>(), align);
let health_size = align_to(len * mem::size_of::<u32>(), align);
let data = alloc(position_size + velocity_size + health_size, align);

let positions = data as *mut Vec2;
let velocities = data.add(position_size) as *mut Vec2;
let healths = data.add(position_size).add(velocity_size) as *mut u32;

GameObjects { positions, velocities, healths, len }
```

This is already quite a bit of boilerplate, and it's mostly pseudocode! It also feels like something the compiler should be able to do for us. So I put together a custom derive macro to extract this information, and created the [`soak`](https://crates.io/crates/soak) crate.

The first (and for now, only) type provided by `soak` is [`RawTable`](https://docs.rs/soak/0.1.0/soak/struct.RawTable.html). Much like `std`'s internal [`RawVec`](https://github.com/rust-lang/rust/blob/master/src/liballoc/raw_vec.rs), it manages an allocation without paying attention to its contents---it won't initialize or drop anything. It looks something like this:

```rust
#[derive(Columns)]
struct GameObject {
    position: Vec2,
    velocity: Vec2,
    health: f32,
}

unsafe fn move_objects(table: &mut RawTable<GameObject>) {
    let positions = table.ptr(GameObject::position);
    let velocities = table.ptr(GameObject::velocity);

    for i in 0..table.capacity() {
        *position.add(i) += *velocity.add(i);
    }
}
```

### The future

I am already making use of `RawTable` as it stands today, in situations where I have a special-purpose pattern for which entries are initialized. But clearly this is not the ideal API for most actual users of `GameObject`s. I would like to be able to provide a `Table` type, the `Vec` to `RawTable`'s `RawVec`, but it's unclear to me how to do so on stable Rust.

#### Split borrows

For example, to allow [split borrows](https://doc.rust-lang.org/nomicon/borrow-splitting.html) into a `Table<GameObject>`, I might provide a method that returns a `(&mut [Vec2], &mut [Vec2], &mut [u32])`. But where to define this type, when `Table` can only learn about `GameObject` via the `Columns` trait? An associated type can't express this, because the references' lifetimes need to be tied to `Table` methods' `&mut self`.

The solution here is probably [generic associated types](https://github.com/rust-lang/rust/issues/44265). In the meantime, perhaps a macro could generate the necessary code for each instantiation of `Table`, or `soak` could just live without split borrows.

#### Slices and mutability

This `(&mut [Vec2], &mut [Vec2], &mut [u32])` type has similar problems to the straightforward implementation of SoA above. The length is duplicated for each slice, and standard tools like iterators are not ergonomic to use. Ideally the `Columns` trait would expose the tuple of `(Vec2, Vec2, u32)` directly, and then `Table` would manipulate it to produce "table iterators," shared and mutable "table slices," and so forth.

This time the solution is probably variadic generics, which seem even further away than generic associated types. There is, though, also probably less need for workarounds. Some API duplication, especially in a macro, is not the worst thing in the world.

#### Driveyard

I intend `soak` to become part of a larger set of tools for tweaking memory layout in Rust, called [Driveyard](https://github.com/driveyard/driveyard). Things like LLVM's [`StringMap`](https://github.com/llvm/llvm-project/blob/2946cd7/llvm/include/llvm/ADT/StringMap.h), a hash table with slices for keys, useful for string interning. Or like [Unity's ECS library](https://unity.com/unity/features/job-system-ECS), which solves several problems with existing Rust implementations of the concept. Or perhaps, as I recently [ranted about on Twitter](https://twitter.com/rpjohnst/status/1095765056308994048), tools for implementing pointer-based data structures.
