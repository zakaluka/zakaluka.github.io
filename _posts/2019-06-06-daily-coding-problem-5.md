---
layout: default
title: Daily Coding Problem 5 - First-class Functions
date: "2019-06-06T12:00:00.000-05:00"
author: zakaluka
tags:
  - Jupyter
  - Daily Coding Problem
  - Python
modified_time: "2019-06-06T12:00:00.000-05:00"
---

# Daily Coding Problem 5 - First-class Functions

This problem was asked by Jane Street.

`cons(a, b)` constructs a pair, and `car(pair)` and `cdr(pair)` returns the first and last element of that pair. For example, `car(cons(3, 4))` returns `3`, and `cdr(cons(3, 4))` returns `4`.

Given this implementation of cons:

```python
def cons(a, b):
    def pair(f):
        return f(a, b)
    return pair
```

Implement `car` and `cdr`.

# Solution

This is not a difficult problem, but requires that the person understand how to use functions as first-class members of the language. Admittedly, python is not a language I use often. I personally prefer statically typed languages and the compile-time goodness that comes along with them.

Regardless, the solution is quite simple. The key is that `cons` returns a function that expects a function as an argument. That argument is what must, in turn, return the correct value from the pair.

First, let's put the `cons` function into scope.

```python
def cons(a, b):
    def pair(f):
        return f(a, b)
    return pair
```

## Implementation of `car`

As the problem states, `car` needs to return the first element of the pair.

```python
def car(cns):
    # Returns the first parameter passed to the function.
    def fst(x, y):
        return x
    return cns(fst)
```

## Implementation of `cdr`

`cdr` needs to return the second element of the pair.

```python
def cdr(cns):
    # Returns the second parameter passed to the function.
    def snd(x, y):
        return y
    return cns(snd)
```

# Testing

We can perform some basic test to ensure that these implementation work correctly. Since the problem did not specify any test cases, i made some up using different types.

```python
a = cons(1, 2)
b = cons('a', 'b')
c = cons(5.0, 6.0)
d = cons(True, False)
```

```python
print ('a: ', car(a), ',', cdr(a))
print ('b: ', car(b), ',', cdr(b))
print ('c: ', car(c), ',', cdr(c))
print ('d: ', car(d), ',', cdr(d))
```

```text
    a:  1 , 2
    b:  a , b
    c:  5.0 , 6.0
    d:  True , False
```

# Conclusion

While this wasn't a difficult problem by any means, it was a fun little exercise that demonstrates the use of functions as first class citizens of a language.
