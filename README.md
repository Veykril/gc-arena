[![crates.io](https://img.shields.io/crates/v/gc-arena)](https://crates.io/crates/gc-arena)
[![docs.rs](https://docs.rs/gc-arena/badge.svg)](https://docs.rs/gc-arena)
[![Build Status](https://img.shields.io/circleci/project/github/kyren/gc-arena.svg)](https://circleci.com/gh/kyren/gc-arena)

This repo is home to the `gc-arena` crate, which provide Rust with garbage
collected arenas and a means of safely interacting with them.

This crate is still fairly experimental and WIP, does work and provides
genuinely safe garbage collected pointers.

### gc-arena

The `gc-arena` crate, along with its helper crate `gc-arena-derive`, provides
safe allocation with cycle-detecting garbage collection within a closed "arena".
There are two techniques at play that make this system sound:

* Garbage collected objects are traced using the `Collect` trait, which must
  be implemented correctly to ensure that all reachable objects are found. This
  trait is therefore `unsafe`, but it *can* safely be implemented by procedural
  macro, and the `gc-arena-derive` provides such a safe procedural macro.

* In order for garbage collection to take place, the garbage collector must
  first have a list of "root" objects which are known to be reachable. In our
  case, the user of `gc-arena` chooses a single root object for the arena, but
  this is not sufficient for safe garbage collection. If garbage collection
  were to take place when there are garbage collected pointers anywhere on the
  Rust stack, such pointers would also need to be considered as "root" objects
  to prevent memory unsafety. `gc-arena` solves this by strictly limiting where
  garbage collected pointers can be stored, and when they can be alive. The
  arena can only be accessed through a single `mutate` method which takes a
  callback, and all garbage collected pointers inside this callback are branded
  with an invariant lifetime which is unique to that single callback call. Thus,
  when outside of this `mutate` method, the rust borrow checker ensures that
  it is not possible for garbage collected pointers to be alive anywhere on
  the stack, nor is it possible for them to have been smuggled outside of the
  arena's root object. Since all pointers can be proven to be reachable from the
  single root object, safe garbage collection can take place.
  
In other words, the `gc-arena` crate does *not* retrofit Rust with a globally
accessible garbage collector, rather it *only* allows for limited garbage
collection in isolated garbage collected arenas. All garbage collected pointers
must forever live inside only this arena, and pointers from different arenas are
prevented from being stored in the wrong arena.

## Use cases

This crate was developed primarily as a means of writing VMs for garbage
collected languages in safe Rust, but there are probably many more uses than
just this.

## Current status and TODOs

Currently, this crate is WIP and experimental, but is basically usable and
safe. Some notable current limitations:

* While this crate hopefully contains no *pathalogical* slowness, it is not
  highly optimized currently. The garbage collector in the `gc-arena` crate is a
  basic incremental mark-and-sweep collector which by itself is not *terrible*,
  but there is still a lot of unnecessary space overhead per-allocation.
  Additionally, there is not currently a way to allocate DSTs in a Gc pointer,
  instead requiring very slow double indirection for any non-`Sized` type. Both
  of these problems are very solvable, but this work hasn't been done yet.
  
* There is currently no system for object finalization. This is not terribly
  difficult to implement, depending on the system, but it would require picking
  a particular set of edge-case finalization behavior.
  
* A harder to solve limitation is that there is currently no system for multi-
  threaded allocation and collection. The basic lifetime and safety techniques
  here would still work in an arena supporting multi-threading, but this crate
  does not support this.
  
* Another limitiation is that the `Collect` trait does not provide a mechanism
  to move objects once they are allocated, so this limits the types of
  collectors that could be written.
  
* The crate is currently pretty light on documentation and examples.

## Prior Art

The ideas here are mostly not mine, much of the design is borrowed heavily from
[rust-gc](https://manishearth.github.io/blog/2015/09/01/designing-a-gc-in-rust/),
and the idea of using "generativity" comes from [You can't spell trust without
Rust](https://raw.githubusercontent.com/Gankro/thesis/master/thesis.pdf).

## License ##

Everything in this repository is licensed under either of:

* MIT license [LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT
* Creative Commons CC0 1.0 Universal Public Domain Dedication
  [LICENSE-CC0](LICENSE-CC0) or
  https://creativecommons.org/publicdomain/zero/1.0/

at your option.
