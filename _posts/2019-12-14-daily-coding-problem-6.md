---
layout: default
title: Daily Coding Problem 6 - XOR Linked List
author: zakaluka
tags:
  - Daily Coding Problem
  - Rust
  - Criterion.rs
  - XOR Linked List
  - Data Structures
date: "2019-12-14T12:00:00.000-05:00"
modified_time: "2019-12-14T12:00:00.000-05:00"
---

# Problem 6 - XOR Linked List

The problem statement is as follows:

> An XOR linked list is a more memory efficient doubly linked list. Instead
> of each node holding `next` and `prev` fields, it holds a field named
> `both`, which is an XOR of the next node and the previous node. Implement
> an XOR linked list; it has an `add(element)` which adds the element to the
> end, and a `get(index)` which returns the node at index.
>
> If using a language that has no pointers (such as Python) you can assume
> you have access to `get_pointer` and `dereference_pointer` functions that
> converts between nodes and memory addresses.

_NOTE: All the code in this post, including this write-up itself, can be
found or generated from the [GitHub repository][5]._

# Preface to Solution

This problem represents a large gap in my recent blog posts because, when I
encountered this problem, I saw it as an opportunity to learn [Rust][3].

For those who are not familiar, here is a short description from
[Mozilla Research][4]:

> Rust is an open-source systems programming language that focuses on speed,
> memory safety and parallelism. Developers are using Rust to create a wide
> range of new software applications, such as game engines, operating
> systems, file systems, browser components and simulation engines for
> virtual reality.

Rust is a "low-level" language whose syntax is quite close to C/C++.
However, it also provides a number of higher-level concepts from other
object-oriented and functional programming languages.

In particular, Rust has a very strong concept of memory safety through
concepts of [ownership][1] and [borrowing][2]. While there are lots of other
articles about this feature and its implementation, here is how it finally
made sense to me: Rust's memory safety is somewhat similar to a pessimistic
database lock. Because mutability is very explicit in Rust, there can be as
many `readers` as desired for a single location in memory. However, when
a `writer` wants to mutate that memory, it receives and keeps exclusive
access to that memory until the work is complete.

As you can imagine, doubly linked lists are particularly annoying in Rust
because a common question that comes up is 'Who owns the current node, the
previous or the next node'. There is an excellent [article][6] about linked
lists in Rust for those wanting a correct and much better introduction to
linked lists in Rust.

It took me the better part of the last few months to learn enough about Rust
to get this exercise working. I gave up pretty much once a month, wrote this
solution partially in [Go][7] and in [F#][8], and did a lot of
experimentation to get to this point. In the end, I was not able to make my
code completely 'safe' in Rust, but I am not sure that is possible with an
XOR linked list. In any case, all feedback is welcome and greatly
appreciated.

# Solution

## Data model

The key to this problem, in my mind, is to design the correct data structure
that can work with an XOR linked list while also ensuring that objects are
not prematurely destroyed (a problem in Rust since the language calls the
[destructor][9] as soon as the last reference to an object goes out of
scope).

Going with the classic model, a linked list is made of a series of nodes.
My choice was to create two data structures that, together, represent an XOR
linked list:

1. An `XORLinkedListNode` is a [struct][10] (similar to a F# record, a Java
   class without methods, or a data transfer object) that holds the following
   information:
   1. `element` holds an integer, representing the value of the node.
   1. `both` holds the XOR'ed memory address.
1. An `XORLinkedList` is a struct that holds the following information:
   1. `nodes` keeps a reference to all `XORLinkedListNode` objects so that Rust does not `drop` while they are still part of the list.

In addition to the above, there are two additional considerations to keep in
mind:

- The Rust compiler is free to move objects around in memory for efficiency
  purposes, whether on the stack or the heap. Because of the strong memory
  safety that Rust provides, this is considered a relatively safe action.
  However, if the compiler chooses to move items around, that would
  invalidate the `both` pointer. To prevent this, each node has a 3rd
  element in its struct called `_pin`.
  - `_pin` forces Rust to pin an object to a memory address. More information
    can be found [here][11].
- By default, Rust creates objects on the stack. To create objects on the
  heap so that they outlive the execution of a function, Rust provides
  [boxed][12] values.
  - The types of `nodes` in the `XORLinkedList` is
    `Vec<Pin<Box<XORLinkedListNode>>>`, i.e. a vector of boxed, pinned
    `XORLinkedListNode` values.

## XORLinkedListNode

`XORLinkedListNode` provides two methods. THe first is the equivalent of a
simple constructor that creates new nodes. The second is a `set_both`
function that changes the value of `both` for a boxed, pinned node.

```rust
unsafe fn set_both(new_both: usize, node: &mut Pin<Box<XORLinkedListNode>>)
{
  let x = node.as_mut();
  Pin::get_unchecked_mut(x).both = new_both;
}
```

## XORLinkedList

`XORLinkedList` provides the two methods requested in the original problem
statement, `add` and `get`.

`add` is responsible for two key actions:

1. Creating the new node.
1. Fixing the `both` attribute of the last node in a non-empty linked list.

```rust
// Adds an item to the end of the linked list.
fn add(&mut self, v: i32) -> &XORLinkedList {
  // Linked list is empty, so add the value with an `EMPTY` both
  // pointer.
  if self.nodes.is_empty() {
    // First, create the new node.
    let n = XORLinkedListNode::new(v, None);

    // Add node to the internal vector.
    self.nodes.push(n);

    self
  } else {
    // Convenience variable for current length.
    let l = self.nodes.len();

    // Create new tail node with `both` set to the address of the old tail.
    // Then, adds the new tail to the internal vector.
    self.nodes.push(XORLinkedListNode::new(
      v,
      Option::from(self.nodes[l - 1].deref()),
    ));

    // Fixes the old tail's `both` to point to the address of the new tail.
    unsafe {
      XORLinkedListNode::set_both(
        self.nodes[l - 1].both ^ address(self.nodes[l].deref()),
        &mut self.nodes[l - 1],
      );
    }

    self
  }
}
```

`get` uses a recursive helper function to walk the list and retrieve the
desired index. In its current implementation, `get` will `panic` if the
requested index is outside the length of the linked list.

```rust
// Returns the `XORLinkedListNode` at the given index. This will panic if
// the index is outside the length of the linked list.
fn get(&self, index: usize) -> &XORLinkedListNode {
  // (Tail-)Recursive function that returns the node at the given index. `n`
  // represents the current node under investigation. `idx` represents the
  // index being counted down (0-based). `prev_address` represents the
  // memory address of the previous node, so that it can be XORed with
  // `n.both` to get the next node's address.
  fn get_helper(
    n: &XORLinkedListNode,
    idx: usize,
    prev_address: usize,
  ) -> &XORLinkedListNode {
    if idx == 0 {
      n
    } else {
      // Get the address of the next node.
      let next_address = n.both ^ prev_address;
      let next_n = next_address as *const XORLinkedListNode;
      get_helper(unsafe { &(*next_n) }, idx - 1, address(n))
    }
  }

  // Call `get_helper`. The previous node of the first entry in the list is
  // always `EMPTY`.
  get_helper(&self.nodes[0], index, EMPTY)
}
```

## Utility methods

The code uses one utility constant and one utility function to make coding
easier.

The first is a constant called `EMPTY` which represents the so-called empty
address, i.e. `0x0`.

The second is a function that returns the address of an object.

```rust
// Gets the memory address of an object.
fn address<T>(elt: &T) -> usize {
  usize::from_str_radix(format!("{:p}", elt).trim_start_matches("0x"), 16)
    .unwrap()
}
```

# Testing

Testing in Rust was extremely easy, thanks to built-in support using the
`cargo test` command.

For this exercise, I did not perform any property-based testing. By the end,
I just wanted to get my code working. However, this exercise really
highlighted the value of unit testing. It was through testing that I
realized that Rust is free to move objects around in memory, even if they
were allocated on the heap. I don't know if other languages also have this
behavior, but it was very surprising to me. That is what led me down the
rabbit hole of both pinned and boxed values.

# Conclusion

Even though this was my first coding challenge in Rust, I don't think I
could have picked a better problem to help me learn about some key concepts
in Rust:

1. Heap allocations
1. Memory safety
1. Unsafe functions
1. Immutability vs. mutability

I am planning to write the next few solutions in Rust to build up my
knowledge of the language.

See you in the next one!

[1]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[2]: https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html
[3]: https://www.rust-lang.org/
[4]: https://research.mozilla.org/rust/
[5]: https://github.com/zakaluka/DailyCodingProblems
[6]: https://rust-unofficial.github.io/too-many-lists/
[7]: https://golang.org/
[8]: https://fsharp.org/
[9]: https://doc.rust-lang.org/nomicon/destructors.html
[10]: https://doc.rust-lang.org/book/ch05-01-defining-structs.html
[11]: https://doc.rust-lang.org/nightly/std/pin/index.html
[12]: https://doc.rust-lang.org/std/boxed/index.html
