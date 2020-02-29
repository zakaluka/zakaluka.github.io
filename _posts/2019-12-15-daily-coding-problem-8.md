---
layout: default
title: Daily Coding Problem 8 - Count Unival Trees
author: zakaluka
tags:
- Daily Coding Problem
- Rust
- Microbenchmark
- Property-based testing
- Criterion.rs
- proptest
- valgrind
- massif
date: "2020-01-18T12:00:00.000-05:00"
modified_time: "2020-01-18T12:00:00.000-05:00"
---

# Problem 8 - Count Unival Trees

The problem statement is as follows:

> A unival tree (which stands for "universal value") is a tree where all
> nodes under it have the same value. Given the root to a binary tree, count the
> number of unival subtrees.
>
> For example, the following tree has 5 unival subtrees:
>
> ```text
>   0
>  / \
> 1   0
>    / \
>   1   0
>  / \
> 1   1
> ```

_NOTE: All the code in this post, including this write-up itself, can be
found or generated from the
[GitHub repository](https://github.com/zakaluka/DailyCodingProblems)._

# Solution

The solution has two major parts:

- The data model, which uses a "classical" representation of a tree.
- The algorithms, which uses a depth-first search.

## Assumptions

The solution makes the following assumptions:

- Each tree is a full binary tree.
- The tree is not deep enough that it will cause a stack overflow (as part
  of a [depth-first search][2] algorithm).
- Instead of making the data structure generic, `Branch`es and `Leaf`s of a
  `Tree` hold a boolean as their value. Making this value an integer or
  even a generic type adds complexity without adding any value to the final
  solution.

According to Wikipedia, a [full binary tree][1] is defined as follows:

> A full binary tree (sometimes referred to as a proper or plane binary
> tree) is a tree in which every node has either 0 or 2 children. Another
> way of defining a full binary tree is a recursive definition. A full
> binary tree is either:
>
> - A single vertex.
> - A tree whose root node has two subtrees, both of which are full binary
>   trees.

## Data model

Here is the data structure used to hold the `Tree`:

```rust
pub enum Tree {
  Leaf(bool),
  Branch(bool, Box<Tree>, Box<Tree>),
}
```

The sub-trees are `Box`ed so that the compiler knows the size of a `Branch`
when one is created.

## Algorithm

The solution performs a depth-first search to find unival trees, going down
the "left" side of the tree first. Each function call takes a tree,
specifically a `Box<Tree>`, as input. The output is a tuple consisting of
an `Option` and an integer, `(Option<bool>, i32)`. If the `Option` is
holding a value, it indicates that the sub-tree which was analyzed is a
unival tree and the value represents the boolean value of that unival tree.
The integer represents the number of unival trees found so far as part of
the search.

```rust
pub(in crate) fn depth_first_search_helper(
  tree: Box<Tree>,
) -> (Option<bool>, i32) {
  match *tree {
    Tree::Leaf(x) => (Option::from(x), 1),
    Tree::Branch(x, lt, rt) => {
      // Count the number of univals along the left child.
      let (l_unival, l_count) = depth_first_search_helper(lt);

      // Count the number of univals along the right child.
      let (r_unival, r_count) = depth_first_search_helper(rt);

      match (l_unival, r_unival) {
        // If the left or right sub-trees don't have the same value, then
        // just send over any counts from both sub-trees.
        (None, _) => (None, l_count + r_count),
        (_, None) => (None, l_count + r_count),

        // If the left and sub-trees both have the same values within their
        // respective sub-trees, check if the two sub-trees have the same
        // value and if it matches the current node.
        (Some(lv), Some(rv)) =>
          if lv == rv && lv == x {
            (Some(x), l_count + r_count + 1)
          } else {
            (None, l_count + r_count)
          },
      }
    }
  }
}
```

# Testing

This code was tested using the example provided by the original problem and
by using property-based testing. Unfortunately, since there is only one
implementation of the algorithm, it is difficult to independently verify the
results. The property-based testing checked for two main properties:

- Number of unival tress > 0
- Number of unival tress >= number of leaves in the tree.

# Benchmarking

The solution was benchmarked in two ways:

1. Execution time was benchmarked using [Criterion.rs][3].
1. Memory usage was benchmarked using [Valgrind Massif][5]

## Strategy

To benchmark the application, I used input values from 1 to 20 to indicate
the number of levels in the input's tree. Values, i.e. the `bool` stored
within each node, were picked randomly using the [`rand`][4] package.

For benchmarking purposes, the system produces "perfect" binary trees, i.e.
binary trees where all interior nodes are `Branch`es and all `Leaf`s have
the same depth. Even though the answer (in terms of unival trees) will vary
for each execution, the number of nodes that are being iterated will remain
the same. This may produce some variance in results, but could potentially
reveal anomalies over multiple benchmark runs. For completely deterministic
results, some strategies include using the same values for all nodes or
alternating values by depth.

## Results

The following line chart depicts CPU usage against the number of elements in
the tree.

![Comparison of time vs elements][10]

The following chart depicts memory usage against the number of elements in
the tree.

![Comparison of memory vs elements][11]

# Conclusion

This project marked a few "firsts" for me:

1. Using [cargo-make][7] to manage builds, documentation, and benchmarks. I
   expect to use it a lot more going forward.

1. Using [Valgrind Massify][5] to benchmark memory usage. I also learned
   that [massif-visualizer][8] and [ms_print][9] interpret "peaks" differently
   in the snapshots, leading to different understanding of program behavior
   when it comes to memory usage. Personally, I prefer massif-visualizer
   because it correctly identified the snapshots with the greatest memory usage
   (as a sum of heap and stack space).

1. Using Rust to quickly generate recursive data structures without
   prototyping them in another language first (such as F#). I hope this is a
   sign that I am getting at least a little more comfortable with the language.

I originally wrote out the solution on a piece of paper while on a long
flight (using pseudocode), and I'm glad to see that the algorithm I wrote
there worked with minimal changes being required. Converting that
pseudocode into Rust was a lot of fun, and there was a decent amount of
wrestling with the compiler to get this to work. However, it was a good
challenge and I'm looking forward to more.

See you in the next one!

# Appendix A - Chart data

The following table shows the data used to create the above memory benchmark
chart in [Gnumeric][6]. All numbers are in bytes, except the element count.

| **Elements** | **Total (B)** | **Useful Heap** | **Extra Heap** | **Stack** |
| -----------: | ------------: | --------------: | -------------: | --------: |
|            2 |   167,779,000 |     100,665,510 |     67,109,042 |     4,448 |
|            4 |   167,779,728 |     100,665,984 |     67,109,232 |     4,512 |
|            8 |   167,780,584 |     100,666,602 |     67,109,574 |     4,408 |
|           16 |   167,782,000 |     100,667,529 |     67,109,895 |     4,576 |
|           32 |   167,783,744 |     100,668,591 |     67,110,633 |     4,520 |
|           64 |   167,787,264 |     100,670,821 |     67,111,739 |     4,704 |
|          128 |   167,792,448 |     100,673,928 |     67,113,816 |     4,704 |
|          256 |   167,803,688 |     100,680,746 |     67,118,118 |     4,824 |
|          512 |   167,822,000 |     100,691,820 |     67,125,508 |     4,672 |
|        1,024 |   167,862,000 |     100,716,227 |     67,140,941 |     4,832 |
|        2,048 |   167,939,936 |     100,762,825 |     67,172,023 |     5,088 |
|        4,096 |   168,107,136 |     100,863,087 |     67,238,897 |     5,152 |
|        8,192 |   168,370,928 |     101,021,429 |     67,344,475 |     5,024 |
|       16,384 |   169,037,368 |     101,421,288 |     67,611,056 |     5,024 |
|       32,768 |   169,893,848 |     101,935,562 |     67,953,382 |     4,904 |
|       65,536 |   172,712,472 |     103,626,428 |     69,080,636 |     5,408 |
|      131,072 |   178,122,752 |     106,872,547 |     71,244,733 |     5,472 |
|      262,144 |   186,988,728 |     112,193,353 |     74,790,223 |     5,152 |
|      524,288 |   208,617,856 |     125,170,799 |     83,441,857 |     5,200 |
|    1,048,576 |   249,538,088 |     149,722,922 |     99,809,918 |     5,248 |

[1]: https://en.wikipedia.org/wiki/Binary_tree
[2]: https://en.wikipedia.org/wiki/Depth-first_search
[3]: https://bheisler.github.io/criterion.rs/book/
[4]: https://rust-random.github.io/book/
[5]: https://valgrind.org/docs/manual/ms-manual.html
[6]: http://gnumeric.org/
[7]: https://github.com/sagiegurari/cargo-make
[8]: https://github.com/KDE/massif-visualizer
[9]: http://valgrind.org/docs/manual/ms-manual.html
[10]: ../../../assets/images/2020-01-18-daily-coding-problem-8/cpu-benchmark.svg "Comparison of time vs elements"
[11]: ../../../assets/images/2020-01-18-daily-coding-problem-8/memory-benchmark.svg "Comparison of memory vs elements"
