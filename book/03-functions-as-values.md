# 3. Functions as values

A function is a value, like a number or a list. It can be passed to another function, returned from one, and stored in a binding. This chapter shows how to write functions as values, how to describe their types, and two ways of calling functions that keep a chain of steps readable: the pipe and method calls.

## Passing a function

`List.map` applies a function to each element of a list, and `List.filter` keeps the elements for which a function returns `True`. The function is an argument like any other:

```emel
fn double(n: Int) -> Int {
  n * 2
}

fn is_even(n: Int) -> Bool {
  n % 2 == 0
}

fn main() {
  xs = [1, 2, 3, 4]
  dbg(List.map(xs, double))
  dbg(List.filter(xs, is_even))
  dbg(List.map(xs, Some))
  ()
}
```

```console
$ emela run
[2, 4, 6, 8]
[2, 4]
[Some(1), Some(2), Some(3), Some(4)]
```

Writing the name of a function without calling it gives the function as a value. A constructor with fields is a function too: `Some` takes a value and returns an `Option`.

## Anonymous functions

A function used in one place does not need a name. An anonymous function lists its parameters between `|` and `|`, followed by its body:

```emel
fn main() {
  xs = [1, 2, 3, 4]
  dbg(List.map(xs, |n| n * 10))
  dbg(List.fold(xs, from: 0, f: |sum, n| sum + n))
  lines = List.map(xs, |n| {
    squared = n * n
    "#{n}^2 = #{squared}"
  })
  dbg(lines)
  scores = [("Ada", 90), ("Bob", 72)]
  dbg(List.map(scores, |(name, score)| "#{name}: #{score}"))
  greet = || "hello"
  dbg(greet())
  ()
}
```

```console
$ emela run
[10, 20, 30, 40]
10
["1^2 = 1", "2^2 = 4", "3^2 = 9", "4^2 = 16"]
["Ada: 90", "Bob: 72"]
"hello"
```

- Parameters are separated by commas: `|sum, n| sum + n`. `List.fold` combines the elements one at a time, starting from `from`; chapter 4 uses it for repetition.
- The body is one expression, and it extends as far to the right as it can. For more than one statement, the body is a block: `|n| { ... }`.
- A parameter can be a pattern that always matches, as `|(name, score)|` takes a pair apart.
- A function without parameters is written with `||`.
- The types of the parameters are usually left out. They come from where the function is passed: in `List.map(xs, |n| n * 10)`, `n` is an `Int` because `xs` is a list of `Int`. They can be written when nothing else decides them, as in `|n: Int| n + 1`.

Because the body takes everything to its right, an anonymous function that is called on the spot is put in parentheses: `(|x| x + 1)(2)` is `3`.

## Function types

A parameter that takes a function has a function type, written `fn(parameter types) -> result type`:

```emel
fn apply_twice(f: fn(Int) -> Int, x: Int) -> Int {
  f(f(x))
}

fn adder(n: Int) -> fn(Int) -> Int {
  |x| x + n
}

fn main() {
  dbg(apply_twice(|x| x + 3, 10))
  add5 = adder(5)
  dbg(add5(1))
  dbg(apply_twice(add5, 0))
  ()
}
```

```console
$ emela run
16
6
10
```

`adder` returns a function. The function it returns uses `n`, a parameter of `adder`, after `adder` has returned: an anonymous function *captures* the variables it uses. Since variables never change, a captured variable keeps the value it had when the function was made:

```emel
fn main() {
  n = 1
  f = || n
  n = 2
  dbg(f())
  dbg(n)
  ()
}
```

```console
$ emela run
1
2
```

The second `n = 2` is a new binding (chapter 2). `f` still holds the first one.

A few rules set function values apart from functions declared with `fn`.

- A function value has no parameter names, so it is called with positional arguments only. `add5(x: 1)` does not compile. The type `fn(Int) -> Int` has no names in it.
- An anonymous function bound to a name cannot call itself, because the name is visible only after the binding. A function that calls itself is declared with `fn`, at the top level or locally (below).
- Functions cannot be compared or shown: a function type has neither `Eq` nor `Show`.

### Errors and effects pass through

A function type can also say which errors and effects the function may have, with `fails` and `uses` (chapters 5 and 6). When a parameter's function type leaves them out, as `f: fn(A) -> B` in `List.map` does, it accepts a function with any errors and effects, and they become errors and effects of the call that receives it. So `List.map` with a function that can fail is a call that can fail, and `List.map` with a function that waits is a call that waits. One `List.map` serves all of them; there is no separate version for functions with effects.

## The pipe

A computation in several steps is written inside out with ordinary calls: `List.fold(List.map(List.filter(orders, ...), ...), ...)` is read from the innermost call outwards. The pipe `|>` writes the same steps in the order they happen:

```emel
type Order(item: String, price: Int, paid: Bool)

fn main() {
  orders = [
    Order(item: "tea", price: 300, paid: True),
    Order(item: "cake", price: 450, paid: False),
    Order(item: "coffee", price: 350, paid: True),
  ]
  total = orders
    |> List.filter(|o| o.paid)
    |> List.map(|o| o.price)
    |> List.fold(from: 0, f: |sum, price| sum + price)
  dbg(total)
  unpaid = orders |> List.filter(|o| !o.paid) |> List.length match {
    0 -> "all paid"
    n -> "#{n} unpaid"
  }
  dbg(unpaid)
  ()
}
```

```console
$ emela run
650
"1 unpaid"
```

`x |> f(a)` is the same as `f(x, a)`: the value on the left becomes the first positional argument of the call on the right. When the right side is not a call, `x |> f` is `f(x)`, as in `|> List.length`. Functions of the standard library take the value they work on as their first parameter, so that they can be used this way.

- A line that starts with `|>` continues the line before it (chapter 2), so each step can stand on its own line.
- The pipe binds more loosely than the other operators, and `match` binds more loosely still. `orders |> List.filter(...) |> List.length match { ... }` matches on the result of the whole chain.

## Method calls

A function can also be called with its first argument written before it, followed by a dot:

```emel
type Money(cents: Int)

fn add(a: Money, b: Money) -> Money {
  Money(cents: a.cents + b.cents)
}

fn describe(m: Money) -> String {
  "#{m.cents} cents"
}

fn main() {
  price = Money(cents: 1250)
  tax = Money(cents: 100)
  dbg(price.add(tax).describe())
  dbg(describe(add(price, tax)))
  dbg(price |> add(tax) |> describe)
  dbg([3, 1, 2].length())
  ()
}
```

```console
$ emela run
"1350 cents"
"1350 cents"
"1350 cents"
3
```

`price.add(tax)` is the same call as `add(price, tax)`. A method call `x.f(a)` looks for `f` in three places, all decided by the type of `x` (`[type.rules]`):

1. a field of that type named `f`, which then holds a function;
2. a function named `f` in the module that defines the type, whose first parameter has that type;
3. a function `f` of a trait the type implements (chapter 9).

The second place is why `[3, 1, 2].length()` works: `List.length` is in the module `List`, which defines `List`, and takes a list first. A function in some other module is not found this way, however it is imported.

The pipe and method calls overlap. A method call finds only functions that belong to the type, without naming their module; the pipe works with any function, and names it. `price.add` without parentheses does not make a function value. To pass `add` with `price` already filled in, write `|other| add(price, other)`.

## Local functions

A function can be declared inside a block, with `fn` as at the top level. It is visible from its declaration to the end of the block, and it can use the variables visible where it is declared:

```emel
fn count_down(from: Int) -> List[Int] {
  fn go(n: Int, acc: List[Int]) -> List[Int] {
    if n > from { acc } else { go(n + 1, [n, ..acc]) }
  }
  go(1, [])
}

fn main() {
  dbg(count_down(5))
  ()
}
```

```console
$ emela run
[5, 4, 3, 2, 1]
```

`go` uses `from`, a parameter of `count_down`, and calls itself. `[n, ..acc]` is a list with `n` in front of the elements of `acc`. Chapter 4 explains why a function written like `go` can repeat as many times as it needs to.

A local function follows the rules of a top-level one: its parameter types are written, and its result type, errors and effects can be inferred. In addition:

- It cannot have type parameters of its own. It can use those of the function around it.
- It cannot be the last statement of its block, since nothing after it could call it.
- Two local functions cannot call each other, because each is visible only from its own declaration on.
- It is not found by a method call.

## Summary

- A function name without a call, a constructor, or `|params| body` is a function value.
- Function types are written `fn(A) -> B`. A function value is called with positional arguments, and cannot be compared or shown.
- An anonymous function captures the variables it uses, with the values they had when it was made.
- A function passed to `List.map` and similar functions passes its errors and effects on to the call.
- `x |> f(a)` is `f(x, a)`. `x.f(a)` finds `f` among the fields of the type of `x`, the functions of the module that defines it, and its traits.
- Local functions are declared with `fn` inside a block and can call themselves.

Chapter 4 writes repetition without loops: recursion, `List.fold`, and the guarantee that makes deep recursion safe.
