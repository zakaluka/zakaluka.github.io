---
layout: default
title: Left-Leaning Red-Black Tree in F#
author: zakaluka
tags:
  - Data Structure
  - Left-Leaning Red-Black Tree
  - FSharp
  - F#
date: "2020-06-05T12:00:00.000-05:00"
modified_time: "2020-06-05T12:00:00.000-05:00"
---

# Left-Leaning Red-Black Tree in F#

*tl;dr Wrote a new left-leaning red-black tree.  You can reference it from [NuGet][21] or download the source from [GitHub][1].*

So, I wrote a left-leaning red-black tree [library][1]. All this started with [Problem 3][2] from the Advent of Code 2019. The problem has users create 2 wires with multiple segments each, find all intersection points between the two wires, and identify the one closest to the central port. Here's a visual representation of the problem:

```text
...........
.+-----+...
.|.....|...
.|..+--X-+.
.|..|..|.|.
.|.-X--+.|.
.|..|....|.
.|.......|.
.o-------+.
...........
```

While there are multiple ways to solve the problem, I figured that such a problem must have already been solved, at least for graphics / game programming.  As you can imagine, this is not only true but there has been research into this area for quite a while.

After going down the rabbit-hole of line intersection algorithms ([line vs. line segment][5], [Shamos-Hoey algorithm][7], [sweep-line algorithms][12], etc.), I decided to implement the [Bentley-Ottmann algorithm][3].  Bentley-Ottmann seemed approachable and it looked like it could handle all edge cases.  Here is the Bentley-Ottmann algorithm in short:

1. Initialize a priority queue Q of potential future events, each associated with a point in the plane and prioritized by the x-coordinate of the point. So, initially, Q contains an event for each of the endpoints of the input segments.
2. Initialize a self-balancing binary search tree T of the line segments that cross the sweep line L, ordered by the y-coordinates of the crossing points. Initially, T is empty. (Even though the line sweep L is not explicitly represented, it may be helpful to imagine it as a vertical line which, initially, is at the left of all input segments.)
3. While Q is nonempty, find and remove the event from Q associated with a point p with minimum x-coordinate. Determine what type of event this is and process it according to the following case analysis:
    1. If p is the left endpoint of a line segment s, insert s into T. Find the line-segments r and t that are respectively immediately above and below s in T (if they exist); if the crossing of r and t (the neighbors of s in the status data structure) forms a potential future event in the event queue, remove this possible future event from the event queue. If s crosses r or t, add those crossing points as potential future events in the event queue.
    2. If p is the right endpoint of a line segment s, remove s from T. Find the segments r and t that (prior to the removal of s) were respectively immediately above and below it in T (if they exist). If r and t cross, add that crossing point as a potential future event in the event queue.
    3. If p is the crossing point of two segments s and t (with s below t to the left of the crossing), swap the positions of s and t in T. After the swap, find the segments r and u (if they exist) that are immediately below and above t and s, respectively. Remove any crossing points rs (i.e. a crossing point between r and s) and tu (i.e. a crossing point between t and u) from the event queue, and, if r and t cross or s and u cross, add those crossing points to the event queue.

Now, as you can tell from the first two steps, we need two primary data structures - a priority queue and self-balancing binary search tree.  I was able to find a [priority queue][13] fairly easily.  However, it wasn't as easy to find a self-balancing red-black tree.  The majority of references I found pointed to [one repository][15] ([archive][14]) by gradbot, which unfortunately is no longer available.

So, I did what any reasonable developer would do - I decided to write my own.  

## Search for an existing implementation

I started out by looking at red-black trees.  I've always enjoyed tree-like data structures due to their inherently 'recursive' nature.  I started with [Wikipedia][16] and quickly found myself on the page for [left-leaning red-black trees][17]. Unfortunately, the linked F# implementation was no longer available.  

From there, I wandered over to the [F# Programming wikibook][8].  I did find a red-black tree implementation there which was very elegant in terms of its balancing function but it did not have a `remove` function (which is needed for Bentley-Ottmann). I could have written one for it and that would have been the end of it.

However, I was already committed to the path of a left-leaning red-black tree - especially after reading a [paper][9] and looking at two separate implementations in [Haskell][10] and [Elm][11].  

## Writing my own

It took a few days to write my own implementation, based heavily on the Elm and Haskell code mentioned earlier.  I did try to do a few things though:

1. I based the API on the F# [List][18] and [Set][19] modules.
1. I've written a number of tests to ensure the structure is working correctly.  This includes property-based tests written using [Hedgehog][20].
1. I managed to publish a package on NuGet, which I consider a major accomplishment.  Just doing this took half as long as writing much of the API.

On the con side, I haven't used it in a program yet and I haven't benchmarked it yet.  However, I'm definitely planning to do that so that I can put it through its paces (and do my best to stamp out any remaining bugs).  If you'd like to use the [library][21] yourself, please feel free to reference it in your projects.

See you in the next one!

[1]: https://github.com/zakaluka/zn-llrbtree
[2]: https://adventofcode.com/2019/day/3
[3]: https://en.wikipedia.org/wiki/Bentley%E2%80%93Ottmann_algorithm
[5]: http://www.differencebetween.net/science/difference-between-line-and-line-segment/
[7]: http://euro.ecom.cmu.edu/people/faculty/mshamos/1976GeometricIntersection.pdf
[12]: http://courses.csail.mit.edu/6.854/17/Scribe/s25-sweepline.html
[13]: https://fsprojects.github.io/FSharpx.Collections/reference/fsharpx-collections-priorityqueue.html
[14]: https://archive.is/20121217160103/https://github.com/gradbot/f-sharp-llrbt
[15]: https://github.com/gradbot/f-sharp-llrbt
[16]: https://en.wikipedia.org/wiki/Red%E2%80%93black_tree
[17]: https://en.wikipedia.org/wiki/Left-leaning_red%E2%80%93black_tree
[8]: https://en.wikibooks.org/wiki/F_Sharp_Programming/Advanced_Data_Structures#Red_Black_Trees
[9]: https://www.cs.princeton.edu/~rs/talks/LLRB/LLRB.pdf
[10]: https://github.com/kazu-yamamoto/llrbtree/blob/master/Data/Set/LLRBTree.hs
[11]: https://github.com/Skinney/core/blob/master/src/Dict.elm
[18]: https://msdn.microsoft.com/visualfsharpdocs/conceptual/collections.list-module-%5bfsharp%5d
[19]: https://msdn.microsoft.com/visualfsharpdocs/conceptual/collections.set-module-%5bfsharp%5d
[20]: https://github.com/hedgehogqa/fsharp-hedgehog
[21]: https://www.nuget.org/packages/zn-llrbtree/
