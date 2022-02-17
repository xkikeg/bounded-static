![ci](https://github.com/fujiapple852/bounded_static/actions/workflows/ci.yml/badge.svg)

# Bounded Static

An experimental crate that defines the `ToBoundedStatic` and `IntoBoundedStatic` traits and provides impls for common
types.  This crate has zero-dependencies, is `no_std` friendly & forbids `unsafe` code.

As described in
the [Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program):

> `T: 'static` should be read as _"`T` is bounded by a `'static` lifetime"_ not _"`T` has a `'static` lifetime"_.

The traits `ToBoundedStatic` and `IntoBoundedStatic` each define an associated type which is bounded by `'static` and 
provide a method to convert to that bounded type:

```rust
pub trait ToBoundedStatic {
    type Static: 'static;
    
    fn to_static(&self) -> Self::Static;
}

pub trait IntoBoundedStatic {
    type Static: 'static;

    fn into_static(self) -> Self::Static;
}
```

## Status

Experimental

## Features

Implementations of `ToBoundedStatic` and `IntoBoundedStatic` are provided for the following `core` types:

- [Primitives](https://doc.rust-lang.org/core/primitive/index.html) (no-op conversions)
- [Option](https://doc.rust-lang.org/core/option/enum.Option.html)

Additional implementations are available by enabling the following features:

- `alloc` for common types from the `alloc` crate:
  - [Cow](https://doc.rust-lang.org/alloc/borrow/enum.Cow.html)
  - [String](https://doc.rust-lang.org/alloc/string/struct.String.html)
  - [Vec](https://doc.rust-lang.org/alloc/vec/struct.Vec.html)
  - [Box](https://doc.rust-lang.org/alloc/boxed/struct.Box.html)
- `collections` for all collection types in the `alloc` crate:
  - [BinaryHeap](https://doc.rust-lang.org/alloc/collections/binary_heap/struct.BinaryHeap.html)
  - [BTreeMap](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html)
  - [BTreeSet](https://doc.rust-lang.org/alloc/collections/btree_set/struct.BTreeSet.html)
  - [LinkedList](https://doc.rust-lang.org/alloc/collections/linked_list/struct.LinkedList.html)
  - [VecDeque](https://doc.rust-lang.org/alloc/collections/vec_deque/struct.VecDeque.html)
- `std` for additional types from `std`:
  - [HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html)

Note that `collections`, `alloc` and `std` are enabled be default.

## Examples

### Converting a `Cow`

Converting a borrowed `Cow<str>` (`Cow::Borrowed`) to an owned `Cow<str>` (`Cow::Owned`):

```rust
fn main() {
    let s = String::from("text");
    let s_cow: Cow<str> = Cow::Borrowed(&s);
    let _s_cow_owned: Cow<str> = s_cow.into_static();
}
```

This is equivalent to:

```rust
fn main() {
    let s = String::from("text");
    let s_cow: Cow<str> = Cow::Borrowed(&s);
    let _s_cow_owned: Cow<str> = Cow::Owned(s_cow.into_owned());
}
```

If the `Cow` should not be consumed then `to_static()` can be used instead:

```rust
fn main() {
    let s = String::from("text");
    let s_cow: Cow<str> = Cow::Borrowed(&s);
    let _s_cow_owned: Cow<str> = s_cow.to_static();
}
```

### Converting a `struct`

Given a structure which can borrow:

```rust
struct Foo<'a> {
    bar: Cow<'a, str>,
    baz: Vec<Cow<'a, str>>,
}
```

And a function which requires its argument is bounded by the `'static` lifetime:

```rust
fn ensure_static<T: 'static>(_: T) {}
```

We can impl `ToBoundedStatic` for `Foo<'_>`:

```rust
impl ToBoundedStatic for Foo<'_> {
    type Static = Foo<'static>;

    fn to_static(&self) -> Self::Static {
        Foo { bar: self.bar.to_static(), baz: self.baz.to_static() }
    }
}
```

Such that:

```rust
#[test]
fn test() {
    let s = String::from("data");
    let foo = Foo { bar: Cow::from(&s), baz: vec![Cow::from(&s)] };
    let to_static = foo.to_static();
    ensure_static(to_static);
}
```

### Deriving `ToBoundedStatic` and `IntoBoundedStatic`

Both the `ToBoundedStatic` and the `IntoBoundedStatic` traits may be automatically derived using the custom derive
marcos provided in the `bounded_static_derive` crate:

```rust
#[derive(bounded_static_derive::ToBoundedStatic)]
struct Foo<'a> {
    bar: Cow<'a, str>,
    baz: Vec<Cow<'a, str>>,
}
```

## Why is this useful?

This is mainly useful when dealing with complex nested `Cow<T>` data structures.

## How does this differ from the `ToOwned` trait?

The [`ToOwned`](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) trait defines an associated type `Owned` which
is bound by [`Borrow<Self>`](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) but not by `'static`.  Therefore,
the follow will not compile:

```rust
use std::borrow::Cow;

fn main() {
    #[derive(Clone)]
    struct Foo<'a> {
        foo: Cow<'a, str>,
    }

    fn ensure_static<T: 'static>(_: T) {}

    let s = String::from("data");
    let foo = Foo { foo: Cow::from(&s) };
    ensure_static(foo.to_owned())
}
```

Results in:

```
error[E0597]: `s` does not live long enough
  --> src/lib.rs:12:36
   |
12 |     let foo = Foo { foo: Cow::from(&s) };
   |                          ----------^^-
   |                          |         |
   |                          |         borrowed value does not live long enough
   |                          argument requires that `s` is borrowed for `'static`
13 |     ensure_static(foo.to_owned())
14 | }
   | - `s` dropped here while still borrowed
```