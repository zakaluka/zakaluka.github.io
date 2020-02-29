---
layout: default
title: Daily Coding Problem 7 - Decoding a Message
author: zakaluka
tags:
  - Daily Coding Problem
  - Rust
  - Microbenchmark
  - Property-based testing
  - Criterion.rs
  - proptest
date: "2019-12-15T12:00:00.000-05:00"
modified_time: "2019-12-15T12:00:00.000-05:00"
---

# Problem 7

The problem statement is as follows:

> Given the mapping a = 1, b = 2, ... z = 26, and an encoded message, count
> the number of ways it can be decoded. For example, the message '111'
> would give 3, since it could be decoded as 'aaa', 'ka', and 'ak'. You can
> assume that the messages are decodable. For example, '001' is not allowed.

_NOTE: All the code in this post, including this write-up itself, can be
found or generated from the
[GitHub repository](https://github.com/zakaluka/DailyCodingProblems)._

# Solution

This solution seems like a relatively straight-forward variation on a graph
problem. It seemed to me that the easiest solution is along the lines of a
depth- or breadth-first traversal through the tree. Implementation-wise, a
recursive function can go through the possibilities without double-counting
any combinations.

In this instance, I chose a depth-first search where each successful search
that results in all valid combinations of characters counts as `1` valid
decode combination. The following is pseudo code representing the high-level
logic for the solution.

The first function is a simple check for whether an integer (encoded as a
set of characters / a string) is valid or not.

```fsharp
let lower_limit = 1
let upper_limit = 26

let is_valid (s: string) : boolean ->
  let n = string_to_int(s)
  s.starts_with() != 0
    AND n >= lower_limit
    AND n <= upper_limit
```

The next function is the primary / naive implementation to the problem.

```fsharp
let rec calc (s: string) : int ->
  if length(s) == 0 then 1
  else
    let one_letter =
      if is_valid(s.[0]) then calc(s.[1..])
      else 0
    let two_letters =
      if is_valid(s.[0..1]) then calc(s.[2..])
      else 0
    one_letter + two_letters
```

## Variation 1 - Memoization

One optimization on the above algorithm is to memoize the number of results
produced by certain substrings. That way, the results do not have to
be re-calculated for every permutation in the earlier part of the string.
Memoization trades storage space for processing time and, with a large
enough input string, the processing savings can be enormous.

The following is pseudo code representing the high-level logic for a
memoization solution.

```fsharp
let rec calc_memo (s: string) : int ->
  if memo.has_key(s) then memo.get_value(s)
  elseif length(s) == 0 then 1
  else
    let one_letter =
      if is_valid(s.[0]) then calc_memo(s.[1..])
      else 0
    let two_letters =
      if is_valid(s.[0..1]) then calc_memo(s.[2..])
      else 0
    memo.add(s, one_letter + two_letters)
    one_letter + two_letters
```

## Variation 2 - Tail Recursion

A second idea is to create a tail recursive algorithm. Implementing a
depth-first search using a tail-recursive algorithm requires two "manual"
book-keeping activities:

- Maintain the collection of yet-to-be investigated branches in a function
  parameter. This accomplishes two tasks:
  - The program does not have to maintain the call stack back to the root
    of the tree.
  - If the tree depth is very large, it (usually) mitigates this problem
    by maintaining the "stack" on the heap rather than within the call
    stack itself.
- Maintain an on-going "answer" to the problem, so that it can be returned
  to the caller once the last entry in the stack has been investigated.

The following is pseudo code representing the high level logic for a tail
recursive solution. This solution can use either a breadth-first or a
depth-first approach, depending on whether new entries are added at the
beginning or at the end of the list of uninvestigated branches.

```fsharp
let rec calc_tr (ls: list<string>, ans: int) : int ->
  if ls.length == 0 then ans
  else
    let s = ls.head
    if length(s) == 0 then calc_tr(ls.tail, ans + 1)
    else
      if is_valid(s.[0]) then ls.add(s.[1..])
      if is_valid(s.[0..1]) then ls.add(s.[2..])
      calc_tr(ls.tail, ans)
```

One concern I had for this implementation is the amount of memory used to
maintain a manual stack.

### Variable 2b - "Fake" Tail Recursion

I also implemented a "fake" version of tail recursion to see if that had any
impact on performance or not. What I mean by "fake" is that I only maintain
one of the "manual" book-keeping activities listed above:

- Maintain an on-going "answer" to the problem, so that it can be returned
  to the caller once the last entry in the stack has been investigated.

This approach does not prevent "springing" or "bouncing" through the call
stack. However, it does prevent having to return to the "top" once all
branches have been investigated. I implemented this to compare performance
against the naive implementation and the proper tail recursive solution.
The follow pseudo code provides high level logic for this variation.

```fsharp
let rec cal_tri (s: string, ans: int) : int ->
  if length(s) == 0 then ans + 1
  else
    let one_letter =
      if is_valid(s.[0]) then calc(s.[1..], ans)
      else ans
    let two_letters =
      if is_valid(s.[0..1]) then calc(s.[2..], one_letter)
      else one_letter
    two_letters
```

# Implementation

You can find the implementation of each of these methods in the
[GitHub repository](https://github.com/zakaluka/DailyCodingProblems).

- Primary Solution = `p7(input_str: &str) -> i32`
- Variation 1 (Memoized) = `p7_memoized(input_str: &str) -> i32`
- Variation 2 (Tail Recursive) = `p7_tail_recursive(input_str: &str) -> i32`
- Variation 2b (Fake Tail Recursion) = `p7_tail_recursive_ish(input_str: &str) -> i32`

# Testing and Benchmarking

I think that Rust has truly made testing and benchmarking incredibly easy
and convenient. Here was my approach to each.

## Testing

I implemented property-based testing for every method and constant. The
property-based tests for methods revealed a number of errors in my code,
from panics to infinite loops. The property-based tests for constants were
really there to ensure that if the constant is changed by mistake, the
corresponding test will fail and alert the responsible party.

For testing, I used the [proptest](https://crates.io/crates/proptest) crate.
I know that [quickcheck](https://crates.io/crates/quickcheck) is more
popular on crates.io, but it was VERY easy to get started with `proptest`
using the examples and by reading the
[proptest book](https://altsysrq.github.io/proptest-book/intro.html).

The following is an example of one of the property-based tests I wrote that
found errors (including a `panic`) with my initial implementation:

```rust
#[test]
fn test_is_valid_pb_all() {
  // Test for crashes
  proptest!(|(x in "[0-9]*")| {
    is_valid(x.as_bytes())
  });
}
```

## Benchmarking

For this problem, I ended up using the
[Criterion](https://crates.io/crates/criterion) crate for benchmarking. I
tried my best to use the Rust-provided benchmarking harness. However, both
finding the latest documentation and trying to understand how it worked was
next to impossible. All the online documentation I found differed not only
from each other, but from what I was seeing in the IDE. I read some posts
about how this feature should be removed if it's not going to be a
first-class citizen. Based on my experience so far (admittedly very
limited), I agree with that sentiment.

Criterion also made it extremely easy to run the benchmarks and do so in a
comparative fashion. All the plots / charts and numbers you will see below
were generated by Criterion, and re-running the benchmarks is as easy as
executing `cargo bench`. The
[criterion.rs book](https://bheisler.github.io/criterion.rs/book/index.html)
was an excellent resource for getting started with this create.

When I think back to how difficult and complicated it was to get similar
data from my previous F# solutions, I think the Rust and Criterion teams
should be extremely proud of how easy they've made it to quickly execute
micro-benchmarks without adding noise to the primary codebase.

The following is a simple benchmark function that I ran with Criterion, for
illustration purposes. The remainder can be found in the GitHub repository.

```rust
fn bench_is_valid(c: &mut Criterion) {
  c.bench_function("is_valid", |b| {
    b.iter(|| {
      for i in 0..100 {
        is_valid(black_box(format!("{}", i).as_bytes()));
      }
    })
  });
}
```

# Results

So, let's talk about the results.

First, given the nature of the problem, each additional character that is
added to the input string results in exponentially longer running time for
most of the algorithms (i.e. `O(k^n)` for some constant `k`). The only
algorithm that did not exhibit this behavior is the Memoization variant.
This makes sense because memoization is specifically designed to provide a
space-time tradeoff, and I did not measure memory usage for these benchmarks
(something for the next problem).
[heapsize](https://crates.io/crates/heapsize) seems like a crate that can
help with such a measurement, if I can find some good _(read: simple)_
examples for how it works.

I ran the benchmarks on 6 input strings, as shown below:

- `"1".repeat(20)`
- `"2".repeat(20)`
- `"12".repeat(10)`
- `"123".repeat(10)`
- `"124782193651078432562974".to_string()` (result = `18`)
- `"12131415161718191010918171615145141313121".to_string()` (result =
  `245760`)

I wanted a combination of repeating and non-repeating strings in the
benchmark to compare their effect on the Memoization variant of the
algorithm. Also, for the Memoization variant, I did not exclude "large
drops" or other memory measurements from the benchmarks. I figure that part
of memoization is dealing with the memory required to make it work, so it
should remain part of the benchmark numbers.

Here are the results for these strings, by algorithm. All times in the below
table are in &micro;s (microseconds).

|        |    Primary | Memoize |         TR |    Fake TR |
| ------ | ---------: | ------: | ---------: | ---------: |
| 1x20   |   `365.60` |  `5.44` |   `421.45` |   `444.28` |
| 2x20   |   `363.65` |  `5.40` |   `427.29` |   `444.94` |
| 12x10  |   `361.92` |  `5.40` |   `422.68` |   `445.25` |
| 123x10 |  `2490.00` |  `8.14` |  `2827.10` |  `2972.60` |
| 1247.. |     `4.58` |  `4.50` |     `5.02` |     `5.23` |
| 1213.. | `10546.00` |  `9.94` | `10945.00` | `11909.00` |

As you can tell from the results, the Memoized variant of the algorithm was
the fastest in all problems, including #5 where all the variants had
relatively close runtimes.

Below youy can find some charts / plots to show how the different variants
compare.

## Line Chart

![Comparison of time vs input][1]

This chart shows the mean measured time for each function as the input (or
the size of the input) increases.

As you can see, the Memoize variant is so fast that is barely registers on
the graph (it's hugging the x-axis at the bottom of the graph).
Surprisingly, the Primary solution is the next fastest. I was expecting the
Fake Tail Recursion solution to be faster than the Primary Solution due to
less movement along the call stack, but it turned out to be the slowest.
The Tail Recursive solution is slower than the Primary solution, but that
seems reasonable since that solution requires significant memory
manipulation through the use of a secondary data structure.

# Conclusion

This is my second Daily Coding Problem challenge in Rust, and I am really
appreciating the language. I do wish that the story for things like full
applications (desktop, mobile) was better, but with the evolution of web
frameworks and WebAssembly, that may be a moot point.

See you in the next one!

# P.S. Additional Charts and Graphs

The following charts and graphs are courtesy of Criterion. I have organized
them by problem / benchmark (rather than by algorithm), so it is easier to
compare algorithms to each other.

I know these images are small, but they are all SVG files. So if you are
interested in the details, you should be able to zoom in without any issues.

## `"1".repeat(20)`

### Primary

![Comparison of iterations vs average time][2]
![Comparison of sample time vs iterations][3]

### Memoize

![Comparison of iterations vs average time][4]
![Comparison of sample time vs iterations][5]

### Tail Recursive

![Comparison of iterations vs average time][6]
![Comparison of sample time vs iterations][7]

### Fake Tail Recursive

![Comparison of iterations vs average time][8]
![Comparison of sample time vs iterations][9]

## `"2".repeat(20)`

### Primary

![Comparison of iterations vs average time][10]
![Comparison of sample time vs iterations][11]

### Memoize

![Comparison of iterations vs average time][12]
![Comparison of sample time vs iterations][13]

### Tail Recursive

![Comparison of iterations vs average time][14]
![Comparison of sample time vs iterations][15]

### Fake Tail Recursive

![Comparison of iterations vs average time][16]
![Comparison of sample time vs iterations][17]

## `"12".repeat(10)`

### Primary

![Comparison of iterations vs average time][18]
![Comparison of sample time vs iterations][19]

### Memoize

![Comparison of iterations vs average time][20]
![Comparison of sample time vs iterations][21]

### Tail Recursive

![Comparison of iterations vs average time][22]
![Comparison of sample time vs iterations][23]

### Fake Tail Recursive

![Comparison of iterations vs average time][24]
![Comparison of sample time vs iterations][25]

## `"123".repeat(10)`

### Primary

![Comparison of iterations vs average time][26]
![Comparison of sample time vs iterations][27]

### Memoize

![Comparison of iterations vs average time][28]
![Comparison of sample time vs iterations][29]

### Tail Recursive

![Comparison of iterations vs average time][30]
![Comparison of sample time vs iterations][31]

### Fake Tail Recursive

![Comparison of iterations vs average time][32]
![Comparison of sample time vs iterations][33]

## `"124782193651078432562974".to_string()`

### Primary

![Comparison of iterations vs average time][34]
![Comparison of sample time vs iterations][35]

### Memoize

![Comparison of iterations vs average time][36]
![Comparison of sample time vs iterations][37]

### Tail Recursive

![Comparison of iterations vs average time][38]
![Comparison of sample time vs iterations][39]

### Fake Tail Recursive

![Comparison of iterations vs average time][40]
![Comparison of sample time vs iterations][41]

## `"12131415161718191010918171615145141313121".to_string()`

### Primary

![Comparison of iterations vs average time][42]
![Comparison of sample time vs iterations][43]

### Memoize

![Comparison of iterations vs average time][44]
![Comparison of sample time vs iterations][45]

### Tail Recursive

![Comparison of iterations vs average time][46]
![Comparison of sample time vs iterations][47]

### Fake Tail Recursive

![Comparison of iterations vs average time][48]
![Comparison of sample time vs iterations][49]

[1]: ../../../assets/images/2019-10-13-daily-coding-problem-7/lines.svg "Comparison of time vs input"
[2]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7_1a.svg "Comparison of iterations vs average time"
[3]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7_1b.svg "Comparison of sample time vs iterations"
[4]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7m_1a.svg "Comparison of iterations vs average time"
[5]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7m_1b.svg "Comparison of sample time vs iterations"
[6]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7tr_1a.svg "Comparison of iterations vs average time"
[7]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7tr_1b.svg "Comparison of sample time vs iterations"
[8]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7tri_1a.svg "Comparison of iterations vs average time"
[9]: ../../../assets/images/2019-10-13-daily-coding-problem-7/1/p7tri_1b.svg "Comparison of sample time vs iterations"
[10]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7_2a.svg "Comparison of iterations vs average time"
[11]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7_2b.svg "Comparison of sample time vs iterations"
[12]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7m_2a.svg "Comparison of iterations vs average time"
[13]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7m_2b.svg "Comparison of sample time vs iterations"
[14]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7tr_2a.svg "Comparison of iterations vs average time"
[15]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7tr_2b.svg "Comparison of sample time vs iterations"
[16]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7tri_2a.svg "Comparison of iterations vs average time"
[17]: ../../../assets/images/2019-10-13-daily-coding-problem-7/2/p7tri_2b.svg "Comparison of sample time vs iterations"
[18]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7_12a.svg "Comparison of iterations vs average time"
[19]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7_12b.svg "Comparison of sample time vs iterations"
[20]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7m_12a.svg "Comparison of iterations vs average time"
[21]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7m_12b.svg "Comparison of sample time vs iterations"
[22]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7tr_12a.svg "Comparison of iterations vs average time"
[23]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7tr_12b.svg "Comparison of sample time vs iterations"
[24]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7tri_12a.svg "Comparison of iterations vs average time"
[25]: ../../../assets/images/2019-10-13-daily-coding-problem-7/12/p7tri_12b.svg "Comparison of sample time vs iterations"
[26]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7_123a.svg "Comparison of iterations vs average time"
[27]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7_123b.svg "Comparison of sample time vs iterations"
[28]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7m_123a.svg "Comparison of iterations vs average time"
[29]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7m_123b.svg "Comparison of sample time vs iterations"
[30]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7tr_123a.svg "Comparison of iterations vs average time"
[31]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7tr_123b.svg "Comparison of sample time vs iterations"
[32]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7tri_123a.svg "Comparison of iterations vs average time"
[33]: ../../../assets/images/2019-10-13-daily-coding-problem-7/123/p7tri_123b.svg "Comparison of sample time vs iterations"
[34]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7_124a.svg "Comparison of iterations vs average time"
[35]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7_124b.svg "Comparison of sample time vs iterations"
[36]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7m_124a.svg "Comparison of iterations vs average time"
[37]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7m_124b.svg "Comparison of sample time vs iterations"
[38]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7tr_124a.svg "Comparison of iterations vs average time"
[39]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7tr_124b.svg "Comparison of sample time vs iterations"
[40]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7tri_124a.svg "Comparison of iterations vs average time"
[41]: ../../../assets/images/2019-10-13-daily-coding-problem-7/124/p7tri_124b.svg "Comparison of sample time vs iterations"
[42]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7_121a.svg "Comparison of iterations vs average time"
[43]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7_121b.svg "Comparison of sample time vs iterations"
[44]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7m_121a.svg "Comparison of iterations vs average time"
[45]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7m_121b.svg "Comparison of sample time vs iterations"
[46]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7tr_121a.svg "Comparison of iterations vs average time"
[47]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7tr_121b.svg "Comparison of sample time vs iterations"
[48]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7tri_121a.svg "Comparison of iterations vs average time"
[49]: ../../../assets/images/2019-10-13-daily-coding-problem-7/121/p7tri_121b.svg "Comparison of sample time vs iterations"
