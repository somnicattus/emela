# 1. Data

An Emela program starts from its data. You say what the values of your problem are, and then write functions that take those values apart and build new ones. This chapter is about the first half: how to define data.

There are three ways to define it.

- A `type` has fields: a user has an id and a name.
- An `enum` has cases: a shape is a circle, a rectangle, or nothing.
- A tuple groups a few values without naming them: `(1, "one")`.

Values never change. To "change" a value, you make a new one from it.

## Running the examples

Create a project and run it:

```console
$ emela new hello
created application `hello`
$ cd hello
$ emela run
Hello, world!
```

A project is a directory with a `Pome.toml` and its sources under `src/`. The program starts at `main` in `src/main.emel`. To try an example of this chapter, replace the contents of `src/main.emel` with it.

`emela run` first writes the command that runs the program, a line starting with `running`, to standard error. The outputs in this book leave that line out.

The program that `emela new` writes prints with the Console effect, which chapter 6 explains. Until then, the examples use `dbg`, which writes a value to standard error in the form that would construct it, and returns the value unchanged.

```emel
fn main() {
  dbg(1 + 2)
  ()
}
```

```console
$ emela run
3
```

`dbg` is meant for debugging, and it is the one kind of output that is not part of the type of a function. Any other output is a side effect, and side effects are part of the type of a function in Emela, so printing properly waits for chapter 6.

The value of the last expression of a function body is its result. `dbg` returns its argument, so a `main` that ends with `dbg(1 + 2)` would return `3`, and the compiler warns that the value is thrown away. The `()` at the end makes the result Unit, the value that carries nothing.

## Basic types

| Type | Values |
|---|---|
| `Int` | 32-bit signed integers: `42`, `1_000_000`, `0xFF`, `0b1010` |
| `Int64` | 64-bit signed integers |
| `Float` | 64-bit floating-point numbers: `3.14`, `1.0e9` |
| `Bool` | `True` and `False` |
| `String` | Text: `"hello"`, and `"""` for text over several lines |
| `()` | Unit. Its only value is `()` |
| `Never` | No values at all. Chapter 5 shows where it appears |

A few rules about them come up early.

- An integer literal is an `Int`, unless the place it is written needs an `Int64`. It is never a `Float`: write `2.0`, not `2`, where a `Float` is needed.
- There are no implicit conversions between number types. `Int64.from_int(n)` turns an `Int` into an `Int64`, and `Float.round(x)` turns a `Float` into an `Int`.
- Integer arithmetic wraps around on overflow: `2147483647 + 1` is `-2147483648`. Dividing an integer by zero stops the program (a *defect*, chapter 5).
- `True` and `False` are not keywords. They are the two constructors of `Bool`, written in upper case like every constructor.
- There is no character type. A single character is a `String` of length 1.

A string can contain other values with `#{...}`:

```emel
fn main() {
  name = "Ada"
  dbg("Hello, #{name}! 1 + 2 = #{1 + 2}")
  ()
}
```

```console
$ emela run
"Hello, Ada! 1 + 2 = 3"
```

`name = "Ada"` binds the name `name` to a value. There is no keyword for it, and there is no assignment: binding the same name again makes a new binding, and the value of the old one does not change. Chapter 2 covers bindings.

## Types with fields

A `type` declaration gives a name to a group of fields.

```emel
type User(id: Int, name: String)
```

This declares the type `User` and a constructor of the same name. A value is made by calling the constructor, and a field is read with `.`:

```emel
type User(id: Int, name: String)

fn main() {
  ada = User(id: 1, name: "Ada")
  dbg(ada.name)
  dbg(ada)
  ()
}
```

```console
$ emela run
"Ada"
User(id: 1, name: "Ada")
```

The arguments of a constructor follow the same rule as the arguments of any function (`[fn.args]`): they can be given by position, by name, or by position first and then by name. `User(1, "Ada")`, `User(1, name: "Ada")` and `User(id: 1, name: "Ada")` make the same value. When a variable has the same name as the field, `User(id:, name:)` is short for `User(id: id, name: name)`.

Types are told apart by name. Two types with the same fields and different names are different types, and a `User` cannot be passed where some other type with an `id` and a `name` is expected. There are no record types without a name.

### Changing some fields

A value does not change, but a new value can be made from it with some fields replaced. Write the fields to replace, then `..` and the value to take the rest from:

```emel
type User(id: Int, name: String)

fn main() {
  ada = User(id: 1, name: "Ada")
  grace = User(name: "Grace", ..ada)
  dbg(grace)
  dbg(ada)
  ()
}
```

```console
$ emela run
User(id: 1, name: "Grace")
User(id: 1, name: "Ada")
```

`ada` is the same after `grace` is made from it.

## Enums

An `enum` lists the cases a value can be, and nothing else. Each case is a *variant*.

```emel
enum Shape {
  Circle(radius: Float)
  Rect(Float, Float)
  Empty
}
```

A variant carries named fields (`Circle`), positional fields (`Rect`), or nothing (`Empty`). One enum can mix the three forms, but one variant cannot mix named and positional fields. Variants are written one per line, or separated by commas.

A variant is a constructor of its enum, not a type of its own. `Circle(radius: 1.0)` is a `Shape`, and a function cannot declare that it takes only circles. For the same reason, the fields of a variant cannot be read with `.`: a `Shape` might not be a `Circle`. To look inside, use `match`:

```emel
enum Shape {
  Circle(radius: Float)
  Rect(Float, Float)
  Empty
}

fn area(shape: Shape) -> Float {
  shape match {
    Circle(radius:)     -> 3.14 * radius * radius
    Rect(width, height) -> width * height
    Empty               -> 0.0
  }
}

fn main() {
  dbg(area(Circle(radius: 1.0)))
  dbg(area(Rect(2.0, 3.0)))
  ()
}
```

```console
$ emela run
3.14
6.0
```

Each arm of the `match` has a pattern on the left of `->`, and the result for that case on the right. A pattern has the same form as the constructor call that would make the value, and names in it bind the fields: `Circle(radius:)` binds `radius`, and `Rect(width, height)` binds the two positional fields. Chapter 2 covers `match` and patterns in full.

An enum is closed: nothing outside its declaration can add a variant. That is what lets the compiler check that a `match` handles every case. A `match` that leaves one out does not compile, so adding a variant to `Shape` shows every `match` that has to handle it.

## Tuples

A tuple groups two or more values by position. Its type is written like its value:

```emel
fn main() {
  pair: (Int, String) = (1, "one")
  (number, word) = pair
  dbg(number)
  dbg(word)
  ()
}
```

```console
$ emela run
1
"one"
```

`(number, word) = pair` takes the tuple apart. A binding can use a pattern on its left when the pattern always matches, as a tuple pattern does.

There is no tuple of one element: `(x)` is `x` in parentheses. The tuple of no elements is `()`, the Unit value from the table of basic types. When the parts of a tuple deserve names, a `type` is the better choice.

## Type parameters

A type can leave the types of some fields open, as type parameters written in `[ ]`:

```emel
type Pair[A, B](first: A, second: B)

fn main() {
  p = Pair(first: 1, second: "one")
  dbg(p)
  q = Pair(first: "uno", ..p)
  dbg(q)
  ()
}
```

```console
$ emela run
Pair(first: 1, second: "one")
Pair(first: "uno", second: "one")
```

The type arguments come from the values: `p` is a `Pair[Int, String]`, and `q`, whose `first` is replaced by a `String`, is a `Pair[String, String]`.

Two types with type parameters are available everywhere without an import:

- `Option[A]` is either `Some(value)` or `None`. Emela has no null; a value that may be missing has an `Option` type, and the code that uses it has to handle `None`.
- `List[A]` is a list of values of one type, written `[1, 2, 3]`.

There is no syntax for indexing a list. `List.get(xs, 0)` returns an `Option`, because the list might be shorter:

```emel
fn main() {
  xs = [10, 20, 30]
  dbg(List.get(xs, 0))
  dbg(List.get(xs, 5))
  ()
}
```

```console
$ emela run
Some(10)
None
```

`List.get` is called with the name of its module in front. Apart from a few names that are always visible (`[lib.prelude]`), functions of the standard library are called this way.

## Constants

A `const` gives a name to a value at the top level of a module. Its name is in upper case.

```emel
type Config(port: Int, host: String)

const DEFAULT_PORT = 8080

const DEFAULT_CONFIG = Config(port: DEFAULT_PORT, host: "localhost")

fn main() {
  dbg(DEFAULT_CONFIG)
  ()
}
```

```console
$ emela run
Config(port: 8080, host: "localhost")
```

The right-hand side can call functions, but it cannot use effects or `fail` (`[data.const]`). It is evaluated the first time the constant is used.

## Comparing and showing values

`==` compares two values, and `#{...}` in a string shows one. Neither works on a type you declare until you say so, because what equality and display mean is a decision about the type. The simplest way to say so is `derive`, which has the compiler write the code from the fields:

```emel
type Point(x: Float, y: Float)
  derive Eq, Show

enum Direction {
  North
  East
  South
  West
}
  derive Eq, Ord

fn main() {
  p = Point(x: 1.0, y: 2.0)
  dbg(p == Point(1.0, 2.0))
  dbg("p is #{p}")
  dbg(North < South)
  ()
}
```

```console
$ emela run
True
"p is Point(x: 1.0, y: 2.0)"
True
```

- `Eq` gives `==` and `!=`. A derived `Eq` compares the fields.
- `Ord` gives `<`, `<=`, `>` and `>=`. A derived `Ord` orders variants in the order they are declared, and compares fields from first to last. It needs `Eq` as well.
- `Show` gives display in `#{...}`. A derived `Show` writes the value in the form that constructs it.

Without `derive Eq`, `p == Point(1.0, 2.0)` does not compile, and without `derive Show`, neither does `"p is #{p}"`. Only `dbg` shows a value of any type, which is why it worked on the types of the earlier examples, none of which derive anything.

`Int`, `Int64`, `String` and `Bool` have all three. `Float` has `Eq` and `Show` but not `Ord`, so a type with a `Float` field cannot derive `Ord`. Tuples, `List` and `Option` have each of them when their elements do. These names are traits; chapter 9 explains what a trait is and how to write one by hand.

## Summary

- A `type` has fields, an `enum` has cases, and a tuple groups values by position. All three are built and taken apart in the same `( )` form.
- Values do not change. `Type(field: value, ..old)` makes a new value from an old one.
- Variants are constructors, not types. Their contents are read with `match`, which must handle every case.
- Missing values are `Option`, not null. Lists have no index syntax.
- Equality, ordering and display are given to a type with `derive Eq, Ord, Show`.

Chapter 2 writes functions over this data.
