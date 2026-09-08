---

title: "Rust Ownership: Move, Borrow, and Copy"
publishDate: 2026-09-08
description: "An introduction to Rust ownership, covering Copy, Move, Borrow, references, and Rust's borrowing rules."
tags: [Rust, Ownership, Move, Borrow, Copy]
---

Unlike C++'s rather strange approach to memory management, Rust adopts a much stricter system called **ownership**.

### The Three Rules of Ownership

There are three basic rules:

1. Every value has an owner.
2. There can only be one owner of a value at a time.
3. When the owner goes out of scope, the value is automatically dropped.

### COPY

In Rust, only certain types implement `Copy`. These types can usually be copied safely and simply because they do not require additional resource management.

```rust
let x = 0;
let y = x;
```

At this point, `x` owns an `i32` with the value `0`, and `y` also owns a separate `i32`.

Types such as `i32`, `bool`, `char`, and `f64` implement `Copy`.

### MOVE

So what happens when we deal with a type that requires ownership management, such as `String`?

Rust handles this through **moving ownership**.

A `String` variable itself does not contain the actual string data. Instead, it stores a pointer, a length, and a capacity, while the actual data lives on the heap.

```rust
let s1 = String::from("hello");
let s2 = s1;
```

When we assign `s1` to `s2`, the `ptr`, `len`, and `cap` are copied into `s2`. `s2` then becomes the owner of the string, and `s1` becomes invalid.

**Note: Function arguments can also cause a Move.**

```rust
fn takes(s: String) {
}

fn main() {
    let s = String::from("hello");

    takes(s);

    println!("{}", s); // ❌
}
```

Here, ownership moves from `s` into the parameter of `takes`. `s` is no longer valid, so any later use of `s` will fail.

### BORROW

But what if I don't want a variable to be moved when passing it to a function?

That's where **borrowing** comes in.

Rust allows us to borrow a value by using a reference, written as `&`.

```rust
fn print(s: &String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");

    print(&s);

    println!("{}", s); // ✅
}
```

In other words, ownership is a bit like owning a book.

If I own the book, I have the right to read or modify it. Moving the book means giving it to its next owner, after which I can no longer use it. Borrowing, on the other hand, is like lending the book to someone else.

Butttttt——as we all know, I can lend a book to several people at the same time if they only want to read it.

But what if someone wants to modify it? If several people try to modify the same book at the same time, things are going to get messy.

So Rust has a rule:

**At any given time, you can have either multiple immutable references, or one mutable reference.**

```rust
fn print(s: &mut String) {
    *s = String::from("HELLO");
    println!("{}", s);
}

fn main() {
    let mut s = String::from("hello");
    println!("{}", s); // hello

    print(&mut s); // HELLO

    println!("{}", s); // ✅ HELLO
}
```

This borrowing rule is one of the core mechanisms Rust uses to guarantee memory safety without relying on a garbage collector.
