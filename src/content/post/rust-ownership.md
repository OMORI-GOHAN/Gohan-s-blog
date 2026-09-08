---

title: "Rust 所有权：Move、Borrow 与 Copy"
publishDate: 2026-09-08
description: "从 Copy、Move 和 Borrow 出发，理解 Rust 所有权、引用以及借用规则。"
tags: [Rust, 所有权, Move, Borrow, Copy]
---

有别于 C++ 的奇怪答辩内存管理，Rust 采用了一套更为严苛的管理机制——**所有权**。

### 所有权的三条规则

1. 每个值都有一个所有者
2. 同一时刻只能有一个所有者
3. 所有者离开作用域，值自动释放

### COPY

在 Rust 中，只有满足特定条件的类型才能实现 `Copy`；通常这类类型可以安全地进行简单复制，而不需要额外的资源管理。

```rust
let x = 0;
let y = x;
```

此时 `x` 拥有一个值为 `0` 的 `i32`，同时 `y` 也单独拥有一个 `i32`。

实现了 `Copy` 的类型有：`i32`、`bool`、`char`、`f64`。

### MOVE

那么当遇到会产生所有权管理问题的类型时，如 `String` 时，Rust 又该如何操作？

事实上，我们将采用所有权移动的方式。

首先，变量本身并不存在字符串，它只保存了指针、长度、容量，真正的数据存在于堆上。

```rust
let s1 = String::from("hello");
let s2 = s1;
```

当我们尝试将 `s1` 赋值给 `s2` 时，`s1` 会将 `ptr`、`len`、`cap` 复制给 `s2`，然后 `s2` 获得字符串所有权，`s1` 作废。

**注意：函数参数也会 Move。**

```rust
fn takes(s: String) {
}

fn main() {
    let s = String::from("hello");

    takes(s);

    println!("{}", s); // ❌
}
```

此时所有权从 `s` 移动至 `takes` 中的参数，`s` 所有权作废，之后对于 `s` 相关的调用失败。

### BORROW

那么如果在函数传参时我不希望一个变量发生 Move，我就需要使用 Borrow。

Rust 允许我们通过使用引用（reference）`&` 来借用值。

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

总的来说，所有权就像一本书：我拥有书，我就有读写的权利；而移动就是将书转送给下一任主人，而自己则无法再操作书；而借用则是借给别人读取或修改内容。

butttttt——众所周知，如果借给别人看，那我可以给好几个人看；而给予别人修改，如果一次有多个人修改，那不就乱套了嘛。

所以 Rust 是这么规定的：

**同一时间，多个只读借用，或者一个可写借用。**

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
