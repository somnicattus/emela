# 4. Repetition without changing values

Emela has no `for` and no `while`. A loop of that kind works by changing a variable on each pass, a counter or a running total, and Emela variables do not change. Repetition is written in one of two ways instead:

- with a function that goes over a collection, such as `List.map` or `List.fold`, when the work is "do this for each element";
- with a function that calls itself, when the work decides for itself when to stop.

This chapter covers both, and the rule that lets a function call itself a million times without running out of stack.

## Going over a list

The `List` module has a function for each common kind of pass over a list:

```emel
fn longest(words: List[String]) -> String {
  List.fold(words, from: "", f: |best, word| {
    if String.length(word) > String.length(best) { word } else { best }
  })
}

fn main() {
  xs = List.range(1, 11)
  dbg(xs)
  dbg(List.map(xs, |n| n * n))
  dbg(List.filter(xs, |n| n % 3 == 0))
  dbg(List.fold(xs, from: 0, f: |sum, n| sum + n))
  dbg(List.any(xs, |n| n > 9))
  dbg(List.find(xs, |n| n * n > 20))
  dbg(longest(["fig", "banana", "kiwi"]))
  ()
}
```

```console
$ emela run
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
[3, 6, 9]
55
True
Some(5)
"banana"
```

| Function | Gives |
|---|---|
| `List.range(from, to)` | The integers from `from` up to, but not including, `to` |
| `List.map(xs, f)` | A list of `f` applied to each element |
| `List.filter(xs, f)` | The elements for which `f` is `True` |
| `List.fold(xs, from: v, f: f)` | One value, combined from all the elements |
| `List.any(xs, f)` | Whether `f` is `True` for some element |
| `List.find(xs, f)` | The first element for which `f` is `True`, as an `Option` |
| `List.each(xs, f)` | Nothing; it calls `f` on each element for what `f` does |

`List.fold` is the general one: the results of most of the others can be computed with it. It starts from the value `from`, and for each element in order, it calls `f` with the value so far and the element, and keeps the result as the new value so far. In `longest`, the value so far is the longest word seen yet. What a loop would keep in a variable that changes, `fold` passes from one call of `f` to the next.

The functions call `f` on the elements in order, from the first, at most once each. `List.any` and `List.find` stop at the first element that answers the question. `List.each` is for functions with effects, such as printing, which chapter 6 introduces.

## Functions that call themselves

When the steps do not follow the elements of a list, a function calls itself. A list is either empty or a first element followed by the rest, and a function over a list can follow that shape:

```emel
fn sum(xs: List[Int]) -> Int {
  xs match {
    []          -> 0
    [x, ..rest] -> x + sum(rest)
  }
}

fn main() {
  dbg(sum([1, 2, 3]))
  ()
}
```

```console
$ emela run
6
```

The sum of an empty list is `0`, and the sum of any other list is its first element plus the sum of the rest. Each call works on a shorter list, so the calls end.

### Tail calls

`sum` has a cost. `x + sum(rest)` has to wait for `sum(rest)` before it can add `x`, so every call stays open until the calls after it finish, and each open call takes room on the stack. On a long enough list, the stack runs out. What happens then is up to the implementation (`[dyn.memory]`); with the current compiler, the program stops with the error of the host, such as `RangeError: Maximum call stack size exceeded` on JS.

A call that is the last thing its function does is a *tail call*, and Emela guarantees that a tail call of a function to itself takes no stack (`[fn.loop]`). It repeats in the space of one call, however many times it repeats. To use the guarantee, give the function a parameter that carries the result so far, and make the call to itself the last thing it does:

```emel
fn length[A](xs: List[A]) -> Int {
  fn go(rest: List[A], count: Int) -> Int {
    rest match {
      []           -> count
      [_, ..later] -> go(later, count + 1)
    }
  }
  go(xs, 0)
}

fn main() {
  dbg(length(List.range(0, 1000000)))
  ()
}
```

```console
$ emela run
1000000
```

`go(later, count + 1)` is the whole value of its arm, so nothing in `go` waits for it. `count` plays the part of the running total in a loop, and the local function `go` keeps that extra parameter out of the signature of `length`.

A call is in tail position when it is:

- the last expression of the function body;
- the last expression of a block that is itself in tail position;
- the body of a `match` arm, or a branch of an `if`, when the `match` or `if` is in tail position;
- the right side of `&&` or `||` in tail position.

So `x + sum(rest)` is not a tail call (the addition comes after it), and neither is a call inside an anonymous function, which runs as part of some other call. A call inside the body of a `with` (chapter 6) is not a tail call either, because the `with` still has work to do after its body ends (chapter 11), and neither is a call inside an `escape` (chapter 5).

The guarantee covers a function that calls itself, including a local function that calls its own name. Tail calls between two functions that call each other are not guaranteed.

### Building a list

A list is read from the front and built a little at a time (`[lib.layers]`). A function that builds one puts each new element in front, `[n, ..steps]`, and when the order matters, it reverses the list once at the end:

```emel
fn collatz(start: Int) -> List[Int] {
  fn go(n: Int, steps: List[Int]) -> List[Int] {
    if n == 1 {
      List.reverse([1, ..steps])
    } else if n % 2 == 0 {
      go(n / 2, [n, ..steps])
    } else {
      go(3 * n + 1, [n, ..steps])
    }
  }
  go(start, [])
}

fn main() {
  dbg(collatz(6))
  ()
}
```

```console
$ emela run
[6, 3, 10, 5, 16, 8, 4, 2, 1]
```

Both calls of `go` are branches of an `if` in tail position, so both are tail calls.

## Choosing between them

- To transform, select, or combine the elements of a list, use the `List` functions. The name says what the code does, and there is no stopping condition to get wrong.
- To repeat until some condition holds, or to walk a structure that is not a list, write a function that calls itself, with an accumulating parameter and a tail call when the repetition can be long.

## Values that do not change

Repetition is the place where a language with variables that change most often uses them, and Emela does without them there as everywhere else. What it gains is that a value means the same thing wherever it is used: a list passed to a function is the same list after the call, and a value captured by a function (chapter 3) is never changed behind its back.

Some programs do need state that changes over time, such as a counter shared by several parts of a program. In Emela that is an effect, State, which chapter 10 covers. A value itself never changes (`[dyn.memory]`).

## Summary

- There is no `for` or `while`. Repetition is a `List` function or a function that calls itself.
- `List.fold` carries a value from one element to the next, in place of a variable that a loop would change.
- A call of a function to itself in tail position takes no stack, so it can repeat any number of times. Other calls use stack, and deep enough recursion runs out of it.
- To make a recursion a tail call, pass the result so far as a parameter, often in a local function.
- State that changes over time is an effect, not a variable.

Chapter 5 is about errors: how a function says that it can fail, and how its caller handles the failure.
