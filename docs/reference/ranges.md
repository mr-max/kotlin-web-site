---
type: doc
layout: reference
category: "Syntax"
title: "Ranges"
---

# Ranges

Range expressions are formed with `rangeTo` functions that have the operator form `..` which is complemented by *in*{: .keyword } and *!in*{: .keyword }.
Range is defined for any comparable type, but for number primitives it is optimized. Here are some examples of using ranges

``` kotlin
if (i in 1..10) { // equivalent of 1 <= i && i <= 10
  println(i)
}

if (x !in 1.0..3.0) println(x)

if (str in "island".."isle") println(str)
```

Numerical ranges have an extra feature: they can be iterated over.
The compiler takes care of converting this analogously to Java's indexed *for*{: .keyword }-loop, without extra overhead.

``` kotlin
for (i in 1..4) print(i) // prints "1234"

for (i in 4..1) print(i) // prints nothing

for (x in 1.0..2.0) print("$x ") // prints "1.0 2.0 "
```

What if you want to iterate over numbers in reverse order? It's simple. You can use the `downTo()` function defined in the standard library

``` kotlin
for (i in 4 downTo 1) print(i) // prints "4321"
```

Is it possible to iterate over numbers with arbitrary step, not equal to 1? Sure, the `step()` function will help you

``` kotlin
for (i in 1..4 step 2) print(i) // prints "13"

for (i in 4 downTo 1 step 2) print(i) // prints "42"

for (i in 1.0..2.0 step 0.3) print("$i ") // prints "1.0 1.3 1.6 1.9 "
```


## How it works

There are two interfaces in the library: `Range<T>` and `Progression<N>`.

`Range<T>` denotes an interval in the mathematical sense, defined for comparable types.
It has two endpoints: `start` and `end`, which are included in the range.
The main operation is `contains`, usually used in the form of *in*{: .keyword }/*!in*{: .keyword } operators.

`Progression<N>` denotes an arithmetic progression, defined for number types.
It has `start`, `end` and a non-zero `increment`.
`Progression<N>` is a subtype of `Iterable<N>`, so it can be used in *for*{: .keyword }-loops and functions like `map`, `filter`, etc.
The first element is `start`, subsequent elements are the previous element plus `increment`.
Iteration over `Progression` is equivalent to an indexed *for*{: .keyword }-loop in Java/JavaScript:

``` java
// if increment > 0
for (int i = start; i <= end; i += increment) {
  // ...
}
```

``` java
// if increment < 0
for (int i = start; i >= end; i += increment) {
  // ...
}
```

For numbers, the `..` operator creates an object which implements both `Range` and `Progression`.
The result of the `downTo()` and `step()` functions is always a `Progression`.

## Range Specifications

### Use Cases

``` kotlin
// Checking if value of comparable is in range. Optimized for number primitives.
if (i in 1..10) println(i)

if (x in 1.0..3.0) println(x)

if (str in "island".."isle") println(str)

// Iterating over arithmetical progression of numbers. Optimized for number primitives (as indexed for-loop in Java).
for (i in 1..4) print(i) // prints "1234"

for (i in 4..1) print(i) // prints nothing

for (i in 4 downTo 1) print(i) // prints "4321"

for (i in 1..4 step 2) print(i) // prints "13"

for (i in (1..4).reversed()) print(i) // prints "4321"

for (i in (1..4).reversed() step 2) print(i) // prints "42"

for (i in 4 downTo 1 step 2) print(i) // prints "42"

for (x in 1.0..2.0) print("$x ") // prints "1.0 2.0 "

for (x in 1.0..2.0 step 0.3) print("$x ") // prints "1.0 1.3 1.6 1.9 "

for (x in 2.0 downTo 1.0 step 0.3) print("$x ") // prints "2.0 1.7 1.4 1.1 "

for (str in "island".."isle") println(str) // error: string range cannot be iterated over
```

### Common Interfaces Definition

There are two base interfaces: `Range` and `Progression`.

The `Range` interface defines a range, or an interval in the mathematical sense.
It has two endpoints, `start` and `end` as well as a `contains()` function which checks if the range contains a given number
(it can also be used with the *in*{: .keyword }/*!in*{: .keyword } operator, which is neater).
`start` and `end` are included in the range. If `start` == `end`, the range contains exactly one element.
If `start` > `end`, the range is empty.

``` kotlin
interface Range<T : Comparable<T>> {
  val start: T
  val end: T
  fun contains(element: T): Boolean
}
```

`Progression` defines a kind of arithmetical progression.
It has `start` (the first element of a progression), `end` (the last element which can be included)
and `increment` (the difference between each progression element and the previous; non-zero).
But the main feature of it is that the progression can be iterated over, so it is a subtype of `Iterable`.
`end` is not necessarily the last element of progression.
Also, progression can be empty if `start < end && increment < 0` or `start > end && increment > 0`.

``` kotlin
interface Progression<N : Any> : Iterable<N> {
  val start: N
  val end: N
  val increment: Number
  // fun iterator(): Iterator<N> is defined in Iterable interface
}
```

Iteration over `Progression` is equivalent to an indexed *for*{: .keyword }-loop in Java:

``` java
// if increment > 0
for (int i = start; i <= end; i += increment) {
  // ...
}

// if increment < 0
for (int i = start; i >= end; i += increment) {
  // ...
}
```


### Implementation Classes

To avoid unnecessary repetition, let's consider only one number type, `Int`.
For other number types the implementation is the same.
Note that instances can be created using constructors of these classes, but it's more useful to use `rangeTo()` (by method call or using the `..` operator), `downTo()`, `reversed()` and `step()` utility functions, which are introduced later.

The `IntProgression` class is pretty straightforward and simple:

``` kotlin
class IntProgression(override val start: Int, override val end: Int, override val increment: Int): Progression<Int> {
  override fun iterator(): Iterator<Int> = IntProgressionIteratorImpl(start, end, increment) // implementation of iterator is obvious
}
```

`IntRange` is a bit tricky: it implements `Progression<Int>` along with `Range<Int>`,
because it's natural to iterate over a range (the default increment value is 1 for both integer and floating-point types):

``` kotlin
class IntRange(override val start: Int, override val end: Int): Range<Int>, Progression<Int> {
  override val increment: Int
    get() = 1
  override fun contains(element: Int): Boolean = start <= element && element <= end
  override fun iterator(): Iterator<Int> = IntProgressionIteratorImpl(start, end, increment)
}
```

`ComparableRange` is also simple (remember that comparisons are translated into invocation of `compareTo()`):

``` kotlin
class ComparableRange<T : Comparable<T>>(override val start: T, override val end: T): Range<T> {
  override fun contains(element: T): Boolean = start <= element && element <= end
}
```

## Utility functions


### `rangeTo()`

The `rangeTo()` functions in number types simply call the constructors of `*Range` classes, e.g.:

``` kotlin
class Int {
  //...
  fun rangeTo(other: Byte): IntRange = IntRange(this, other)
  //...
  fun rangeTo(other: Int): IntRange = IntRange(this, other)
  //...
}
```

### `downTo()`

The `downTo()` extension function is defined for any pair of number types, here are two examples:

``` kotlin
fun Long.downTo(other: Double): DoubleProgression {
  return DoubleProgression(this, other, -1.0)
}

fun Byte.downTo(other: Int): IntProgression {
  return IntProgression(this, other, -1)
}
```

### `reversed()`

The `reversed()` extension functions are defined for each `*Range` and `*Progression` classes, and all of them return reversed progressions.

``` kotlin
fun IntProgression.reversed(): IntProgression {
  return IntProgression(end, start, -increment)
}

fun IntRange.reversed(): IntProgression {
  return IntProgression(end, start, -1)
}
```

### `step()`

`step()` extension functions are defined for `*Range` and `*Progression` classes,
all of them return progressions with modified `step` values (function parameter).
Note that the step value is always positive, therefore this function never changes the direction of iteration.

``` kotlin
fun IntProgression.step(step: Int): IntProgression {
  if (step <= 0) throw IllegalArgumentException("Step must be positive, was: $step")
  return IntProgression(start, end, if (increment > 0) step else -step)
}

fun IntRange.step(step: Int): IntProgression {
  if (step <= 0) throw IllegalArgumentException("Step must be positive, was: $step")
  return IntProgression(start, end, step)
}
```
