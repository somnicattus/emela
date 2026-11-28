# 2. Functions and expressions

Chapter 1 defined data. This chapter writes the functions that work on it: how to declare and call a function, how a function body is built from bindings and expressions, and how `if` and `match` choose what to compute.

Everything in a function body is an expression and has a value: a block, an `if`, a `match`. The only thing that is not an expression is a binding, which gives a name to a value.

## Declaring a function

```emel
fn add(x: Int, y: Int) -> Int {
  x + y
}

fn main() {
  dbg(add(1, 2))
  ()
}
```

```console
$ emela run
3
```

A declaration starts with `fn`, then the name, the parameters with their types, and the type of the result after `->`. The body is a block, and the value of its last expression is the result of the function. There is no `return`: a function ends by reaching the end of its body.

The types of the parameters are always written. The type of the result can be left out, and the compiler infers it from the body:

```emel
fn double(n: Int) {
  n * 2
}
```

`double` returns an `Int`, and a caller that expects something else does not compile. The editor shows the inferred type (chapter 18). A function that other modules use usually writes its result type, so that its signature says what it returns (chapter 7).

A declaration can also say which errors the function can fail with (`fails`, chapter 5) and which effects it uses (`uses`, chapter 6). The effects are inferred whenever `uses` is left out. The errors are inferred together with the result type when both are left out; a function that writes its result type and no `fails` says that it does not fail, and the compiler holds it to that.

### Arguments and their names

The name of a parameter is also the name of the argument at a call. Arguments can be given by position, by name, or by position first and then by name:

```emel
fn connect(host: String, port: Int) -> String {
  "#{host}:#{port}"
}

fn main() {
  dbg(connect("localhost", 5432))
  dbg(connect("localhost", port: 5432))
  dbg(connect(host: "localhost", port: 5432))
  port = 8080
  dbg(connect("example.com", port:))
  ()
}
```

```console
$ emela run
"localhost:5432"
"localhost:5432"
"localhost:5432"
"example.com:8080"
```

- Positional arguments come first. `connect(host: "localhost", 5432)` does not compile.
- `port:` alone is short for `port: port`, when a variable has the name of the parameter.
- Naming an argument makes a call easier to read where the values alone do not say what they are, as in `resize(img, width: 640, height: 480)`.
- Because callers can write them, parameter names are part of what a function promises. Renaming a parameter of a public function can break its callers.

Constructors follow the same rules (chapter 1), so `User(1, name: "Ada")` is a call like any other.

### Type parameters

A function can work on values of any type, written as type parameters in `[ ]` after its name:

```emel
fn first[A](xs: List[A]) -> Option[A] {
  xs match {
    [x, ..] -> Some(x)
    []      -> None
  }
}

fn main() {
  dbg(first([3, 1, 2]))
  dbg(first(["a"]))
  ()
}
```

```console
$ emela run
Some(3)
Some("a")
```

A call does not write the type arguments. They come from the arguments: `first([3, 1, 2])` uses `first` with `A` as `Int`. The body of `first` does not know what type `A` is, so it can only hold an `A`, put it into other values and pass it on; it cannot compare one or show one. Chapter 9 shows how a bound such as `[A: Show]` lets it do more.

## Blocks and bindings

A binding is written `name = expression`. There is no keyword before it.

```emel
fn main() {
  count = 1
  count = count + 1
  dbg(count)
  (q, r) = (17 / 5, 17 % 5)
  dbg(q)
  dbg(r)
  total: Int64 = 1
  dbg(total)
  ()
}
```

```console
$ emela run
2
3
2
1
```

- A variable never changes. `count = count + 1` makes a new binding named `count`; from that line on, the name refers to the new value. Code that already used the old value keeps it. There is no assignment.
- The left side can be a pattern, as long as the pattern always matches: `(q, r)` takes a pair apart. A pattern that can fail to match, such as `Some(x)`, does not compile in a binding, because there would be no value to bind when it fails. Use `match` for that.
- A type can be written after the name, as in `total: Int64 = 1`. It is needed only where the value alone does not decide the type, as for the literal `1`, which would otherwise be an `Int`.

A block `{ ... }` is an expression too. Its bindings are visible only inside it, and its value is the value of its last expression:

```emel
fn main() {
  area = {
    width = 3
    height = 4
    width * height
  }
  dbg(area)
  ()
}
```

```console
$ emela run
12
```

A line break ends a statement, so there are no semicolons. An expression continues onto the next line while a `(` or `[` is open, or when the next line starts with a binary operator such as `+` or `&&`, with `.`, or with `else` (`[lex.continuation]`). An operator at the end of a line does not continue it. `-` and `||` at the start of a line start a new statement instead, so a subtraction or an `||` that spans lines is written inside parentheses.

## Operators

| Operators | Kind |
|---|---|
| `!x` `-x` | Negation |
| `*` `/` `%` | Multiplication, division, remainder |
| `+` `-` | Addition, subtraction |
| `==` `!=` `<` `<=` `>` `>=` | Comparison |
| `&&` | And |
| `\|\|` | Or |

Operators higher in the table bind more tightly: `!a && b` is `(!a) && b`, and `1 + 2 * 3` is `7` (`[fn.operator]`). The pipe and `match` bind more loosely than all of these (chapter 3). A few rules are worth knowing from the start.

- Arithmetic works on `Int`, `Int64` and `Float`, and both sides have the same type. `1 + 2.0` does not compile; write `1.0 + 2.0`, or convert one side with a function.
- `+` does not join strings. Use interpolation, `"#{a}#{b}"`.
- `==` and `!=` work on types with `Eq`, and `<` and the others on types with `Ord` (chapter 1).
- Comparisons do not chain. `0 < a < 3` does not compile; write `0 < a && a < 3`.
- `&&` and `||` evaluate their right side only when the left side does not already decide the result.
- A type you declare cannot define its own operators. Write a function, such as `Money.add(a, b)`.
- There are no bitwise operators. They are functions: `Int.bit_and(a, b)`, `Int.shift_left(a, 2)`.

## if

`if` chooses between two blocks, and its value is the value of the block it chose. The condition is a `Bool` and needs no parentheses.

```emel
fn describe(n: Int) -> String {
  sign = if n < 0 { "negative" } else if n == 0 { "zero" } else { "positive" }
  parity = if n % 2 == 0 { "even" } else { "odd" }
  "#{n} is #{sign} and #{parity}"
}

fn main() {
  dbg(describe(-3))
  dbg(describe(10))
  ()
}
```

```console
$ emela run
"-3 is negative and odd"
"10 is positive and even"
```

An `if` without `else` has the value `()`, so it is used only for a block that is run for what it does, not for a value. `x = if ready { 1 }` does not compile: there would be no value when `ready` is `False`. Since `if` already has a value, there is no separate conditional operator.

## match

`match` takes a value apart by its shape. It is written after the value:

```emel
enum Shape {
  Circle(radius: Float)
  Rect(Float, Float)
  Empty
}

fn area(shape: Shape) -> Float {
  shape match {
    Circle(radius:) if radius > 0.0 -> 3.14 * radius * radius
    Circle(_)                       -> 0.0
    Rect(w, h)                      -> w * h
    Empty                           -> 0.0
  }
}

fn main() {
  dbg(area(Circle(radius: -1.0)))
  ()
}
```

```console
$ emela run
0.0
```

The arms are tried from top to bottom, and the first arm whose pattern matches gives the value of the `match`. An arm can add a *guard*, `if condition`, after its pattern; the arm is chosen only when the pattern matches and the condition is `True`. The names a pattern binds can be used in its guard and on the right of `->`.

The value comes first so that code reads from left to right in the order it runs. That matters most when the value is itself a longer expression, such as a chain of calls (chapter 3). `match value { ... }` with the `match` first does not compile.

### Patterns

| Pattern | Matches |
|---|---|
| `_` | Anything, without binding it |
| `x` | Anything, and binds it to `x` |
| `0`, `-1`, `"yes"` | A value equal to the literal |
| `MAX_SIZE` | A value equal to a constant |
| `Empty`, `Rect(w, h)`, `Circle(radius: r)`, `Circle(radius:)` | A value made with that constructor, with patterns for its fields |
| `User(name:, ..)` | A value of a constructor, ignoring the fields not written |
| `(a, b)` | A tuple, element by element |
| `[]`, `[x]`, `[x, y]`, `[x, ..rest]`, `[x, ..]` | The empty list, a list of exactly one or two elements, or a first element followed by the rest of the list |

Patterns nest: `Some(Some(n))` matches an `Option[Option[Int]]` that holds a number, and `(0, _)` matches a pair whose first element is `0`. A pattern for a constructor with named fields either names all of them, or ends with `..` to say that the rest do not matter; leaving a field out without `..` does not compile.

```emel
type User(id: Int, name: String, admin: Bool)

fn greeting(user: User) -> String {
  user match {
    User(admin: True, name:, ..) -> "Welcome back, administrator #{name}"
    User(name:, ..)              -> "Hello, #{name}"
  }
}

fn fizzbuzz(n: Int) -> String {
  (n % 3, n % 5) match {
    (0, 0) -> "FizzBuzz"
    (0, _) -> "Fizz"
    (_, 0) -> "Buzz"
    _      -> Int.to_string(n)
  }
}

fn summary(xs: List[Int]) -> String {
  xs match {
    []          -> "empty"
    [x]         -> "just #{x}"
    [x, y]      -> "#{x} and #{y}"
    [x, ..rest] -> "#{x} and #{List.length(rest)} more"
  }
}

fn main() {
  dbg(greeting(User(id: 1, name: "Ada", admin: True)))
  dbg(greeting(User(id: 2, name: "Bob", admin: False)))
  dbg(fizzbuzz(9))
  dbg(fizzbuzz(15))
  dbg(summary([1]))
  dbg(summary([1, 2, 3, 4]))
  ()
}
```

```console
$ emela run
"Welcome back, administrator Ada"
"Hello, Bob"
"Fizz"
"FizzBuzz"
"just 1"
"1 and 3 more"
```

`fizzbuzz` matches on a tuple it builds on the spot, to branch on two values at once.

### Every case is handled

A `match` that does not cover every possible value does not compile, and the error shows an example of a value it misses:

```console
$ emela check
error[E3102]: non-exhaustive match: pattern `Circle(_)` not covered
   ╭─[ src/main.emel:8:3 ]
   │
 8 │   shape match {
   │   ──┬──
   │     ╰────
───╯

1 error
```

This is the message for `area` above without its `Circle(_)` arm. `emela check` reports errors without building or running the program. An arm with a guard does not count as covering its pattern, because its condition might be `False`, so `area` needs the second `Circle` arm.

The check also works the other way: an arm that can never be chosen, because the arms above it already match everything it would, gets a warning (W3101).

## Summary

- A function declares the types of its parameters. Its result type and its effects are inferred when left out, and so are its errors when the result type is left out too.
- The last expression of a block is its value. There is no `return` and no semicolon.
- Arguments go by position, then by name. Parameter names are part of a function's interface.
- Variables never change; binding a name again makes a new binding.
- `if` and `match` are expressions. `match` comes after its value, tries its arms in order, and must handle every case.

Chapter 3 treats functions as values: passing them to other functions, writing them inline, and chaining calls with the pipe.
