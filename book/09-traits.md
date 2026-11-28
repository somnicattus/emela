# 9. Traits

Since chapter 1, `derive Eq, Ord, Show` has let values be compared with `==` and `<` and formatted inside `#{...}`. Those three names are *traits*. A trait names a set of functions that a type can provide, and lets a generic function require them on a type parameter (`[trait]`).

Traits describe behavior that belongs to a type: how two values compare, how a value turns into text, how it decodes from JSON. Behavior that depends on the world outside, such as printing or talking to a database, is an effect, not a trait (chapter 10).

## Declaring a trait

A `trait` lists the signatures of the functions that implementing types provide (`[trait.decl]`):

```emel
trait Describe {
  fn describe(self) -> String
}

type User(id: Int, name: String)

enum Role {
  Admin
  Member(team: String)
}

impl User: Describe {
  fn describe(self) {
    "#{self.name} (##{self.id})"
  }
}

impl Role: Describe {
  fn describe(self) {
    self match {
      Admin         -> "admin"
      Member(team:) -> "member of #{team}"
    }
  }
}

fn announce[A: Describe](value: A) -> String {
  "-> #{value.describe()}"
}

fn main() {
  ada = User(id: 1, name: "Ada")
  dbg(Describe.describe(ada))
  dbg(ada.describe())
  dbg(announce(ada))
  dbg(announce(Member(team: "core")))
  ()
}
```

```console
$ emela run
"Ada (#1)"
"Ada (#1)"
"-> Ada (#1)"
"-> member of core"
```

- `self` is the first parameter, and its type is `Self`, the type being implemented for. Only trait and `impl` functions take `self` as a parameter (handlers also use `self` and `Self` inside their bodies, chapter 10).
- A trait function whose return type is omitted returns `()`.
- A function can omit `self` and use `Self` only in its return type (below).

### Supertraits and default implementations

A trait can require another trait, called a *supertrait*, after a colon (`trait A: B`, or `trait A: B + C` for several). A function in a trait can also carry a body, which serves as a *default implementation* that an `impl` may leave out:

```emel
trait Priority: Eq {
  fn compare_priority(self, other: Self) -> Ordering

  fn higher(self, other: Self) -> Self {
    Priority.compare_priority(self, other) match {
      Less -> other
      _    -> self
    }
  }
}

enum Ticket {
  Low
  Normal
  Urgent
}
  derive Eq

fn rank(ticket: Ticket) -> Int {
  ticket match {
    Low    -> 0
    Normal -> 1
    Urgent -> 2
  }
}

impl Ticket: Priority {
  fn compare_priority(self, other) {
    Ord.compare(rank(self), rank(other))
  }
}

fn main() {
  first = Normal
  second = Urgent
  dbg(first.higher(second))
  dbg(Priority.higher(Urgent, Low))
  ()
}
```

```console
$ emela run
Urgent
Urgent
```

Because `Priority` has `Eq` as a supertrait, `impl Ticket: Priority` checks at once that `Ticket` implements `Eq`, and fails to compile if it does not. `Ticket` implements only `compare_priority` and inherits `higher`. Supertraits cannot form a cycle (`trait A: B` and `trait B: A` does not compile).

### Errors and effects of trait functions

Trait functions are pure by default (`[trait.effects]`). When a trait function may fail or use effects, its declaration says so with `fails` or `uses`:

```emel
error BadFormat(input: String) derive Show

trait FromText {
  fn from_text(input: String) -> Self fails BadFormat
}

type Port(value: Int)

impl Port: FromText {
  fn from_text(input) {
    n = Int.parse(input) escape {
      _ -> fail BadFormat(input:)
    }
    if n < 1 || n > 65535 { fail BadFormat(input:) }
    Port(value: n)
  }
}

fn main() {
  port: Port = FromText.from_text("8080")
  dbg(port)
  ()
}
```

```console
$ emela run
Port(value: 8080)
```

An implementation may raise errors and use effects only within the range the trait declared. As a result, a generic function with `[A: FromText]` knows its own errors and effects from the trait declaration alone, regardless of which concrete type `A` turns out to be.

## Implementing a trait

An implementation is written `impl Type: Trait` (`[trait.impl]`). The colon has the same meaning as in `[A: Ord]` and `trait Ord: Eq`: the left side satisfies the right side.

- Parameter and return types are fixed by the trait declaration, so an `impl` leaves them out. Every function without a default implementation must be written.
- An `impl` for a parameterized type puts the bounds on its type parameters in the head:

```emel
type Pair[A, B](first: A, second: B)

impl Pair[A: Eq, B: Eq]: Eq {
  fn eq(self, other) {
    self.first == other.first && self.second == other.second
  }
}

impl Pair[A: Show, B: Show]: Show {
  fn show(self) {
    "(#{self.first}, #{self.second})"
  }
}

fn largest[A: Ord](xs: List[A]) -> Option[A] {
  xs match {
    [] -> None
    [first, ..rest] -> Some(
      List.fold(rest, from: first, f: |best, x| if x > best { x } else { best }),
    )
  }
}

fn same_largest[A: Ord](xs: List[A], ys: List[A]) -> Bool {
  largest(xs) == largest(ys)
}

fn main() {
  p = Pair(first: 1, second: "one")
  dbg(p == Pair(first: 1, second: "one"))
  dbg("pair: #{p}")
  dbg(same_largest([3, 1, 4], [4, 2]))
  ()
}
```

```console
$ emela run
True
"pair: (1, one)"
True
```

Three rules guarantee that there is never any doubt about which implementation a call reaches:

- The head of an `impl` contains only the type name and distinct type parameters. `impl List[Int]: Show` and `impl Pair[A, A]: Eq` do not compile, and writing two `impl`s for the same `(Type, Trait)` pair anywhere in the program does not compile either. Implementations never overlap.
- An `impl` may be written only in the module that defines the type or in the module that defines the trait. If neither belongs to you, wrap the type in a single-field `type` of your own (which carries no run-time cost) and implement the trait for the wrapper:

  ```emel
  type MyDate(inner: ThirdParty.Date)

  impl MyDate: Encode { ... }
  ```

- When an `impl` implements a trait with supertraits, the bounds required by the supertrait's `impl` must follow from the bounds written on the head. `impl Pair[A: Ord, B: Ord]: Ord` is accepted because `A: Eq` and `B: Eq` follow from `A: Ord` and `B: Ord`.

## Calls and bounds

A trait function can be called in three ways (`[trait.call]`):

- Qualified by the trait name: `Describe.describe(ada)` or `ada |> Describe.describe`.
- As a method call: `ada.describe()`, when the function takes `self` and the trait is visible by name in the calling module (Prelude traits, traits defined in the module, or traits imported in braces; `[type.rules]`). On a value whose type is a bounded type parameter such as `[A: Describe]`, the functions of its bounds are found directly, as in `value.describe()` in `announce`.
- When a function has no `self` and `Self` appears only in its return type, as in `FromText.from_text("8080")`, the compiler chooses the `impl` from the expected return type. That type can come from a type annotation (`port: Port = ...`) or from how the result is used (`FromText.from_text(s) + 1` chooses `Int`). If nothing in the function determines it, the call does not compile:

```emel
error BadFormat(input: String) derive Show

trait FromText {
  fn from_text(input: String) -> Self fails BadFormat
}

fn main() {
  x = FromText.from_text("8080")
  dbg(x)
  ()
}
```

```console
$ emela check
error[E3307]: cannot infer which impl of `FromText` to use: add a type annotation
   ╭─[ src/main.emel:8:7 ]
   │
 8 │   x = FromText.from_text("8080")
   │       ─────────┬────────
   │                ╰──────────
───╯

1 error
```

A generic function constrains a type parameter with `[A: Ord]`, or `[A: Ord + Show]` for more than one trait. A bound includes its supertraits: `same_largest` above writes only `[A: Ord]` and still uses `==` on `Option[A]`, because `Ord` implies `Eq`. Bounds are checked at every reference to the function, even when it is passed as a value without being called.

## Built-in traits

Five traits are built into the compiler (`[trait.builtin]`):

| Trait | Used by | User `impl` |
|---|---|---|
| `Eq` | `==`, `!=` | Allowed |
| `Ord: Eq` | `<`, `<=`, `>`, `>=` | Allowed |
| `Show` | `#{...}`, `inspect` | Allowed |
| `Immediate` | Effect sets that do not suspend (chapter 12) | Determined by the compiler |
| `Capability` | Capability types `use E` (chapter 10) | Determined by the compiler |

Writing an `impl` for `Immediate` or `Capability` does not compile.

### `Ord` and `Float`

`Ord.compare` promises a total order where `Equal` agrees with `==`. `Float` cannot keep that promise because of `NaN`, which is not equal even to itself, so `Float` implements `Eq` and `Show` but not `Ord`.

Operators `<`, `<=`, `>`, and `>=` on `Float` directly are built-in IEEE 754 comparisons (every comparison with `NaN` is `False`). Inside a generic function, comparisons go through `Ord`, so passing a `List[Float]` to `largest` does not compile:

```console
$ emela check
error[E3302]: type `Float` does not implement `Ord`
    ╭─[ src/main.emel:11:7 ]
    │
 11 │   dbg(largest([1.5, 2.5]))
    │       ───┬───
    │          ╰─────
────╯

1 error
```

When a total order on floats is needed, pass `Float.compare` explicitly; it places `NaN` largest and treats two `NaN`s, and `0.0` and `-0.0`, as `Equal`:

```emel
fn main() {
  dbg(List.sort_by([3.5, 1.2, 2.8], Float.compare))
  ()
}
```

```console
$ emela run
[1.2, 2.8, 3.5]
```

### `Show`, `inspect` and `dbg`

A derived `Show` formats a value in the form that constructs it in source (`User(id: 1, name: "Ada")`). In string interpolation, a top-level `String` is embedded without quotes, while a `String` inside a list or constructor is quoted. `inspect[A: Show](value: A) -> String` requires `Show` and quotes even a top-level `String`:

```emel
type Label(text: String) derive Show

fn main() {
  text = "hello"
  dbg("#{text}")
  dbg(inspect(text))
  dbg("#{Label(text:)}")
  ()
}
```

```console
$ emela run
"hello"
"\"hello\""
"Label(text: \"hello\")"
```

`dbg` does not require `Show` and shows a value of any type, including types without `Show`, opaque types (showing their contents), functions (`<fn>`), and host objects (`<external Name>`). Its output goes only to standard error and cannot be captured as a `String`. Displaying a failed `assert` (chapter 8) or an error that reaches `main` (chapter 5) does not require `Show` either, though both show richer output when `Show` is present. Function types do not satisfy `Eq`, `Ord`, or `Show`.

## The details of `derive`

`derive` asks the compiler to generate ordinary `impl`s from the fields and variants of a type (`[trait.derive]`):

- Each field goes through its own type's `impl`. If a `User` holds a custom `Email` with a hand-written `Show`, a derived `Show` on `User` calls that `Show`. If any field's type does not implement the trait, `derive` does not compile (so a type with a `Float` field cannot derive `Ord`).
- Deriving for a parameterized type adds the bound to every type parameter (`enum Tree[A] { ... } derive Eq` generates `impl Tree[A: Eq]: Eq`), even if a parameter does not appear in a field.
- Supertraits are not derived implicitly: `derive Ord` requires `Eq` to be derived or implemented as well. Combining `derive Show` with a hand-written `impl T: Show` does not compile.
- User-declared types satisfy none of `Eq`, `Ord`, or `Show` unless `derive` or `impl` is written. Nothing is derived implicitly.
- On a private type, a derived trait that nothing in the program requires produces a warning (an exception is `Show` on an `error`, which is used if the error reaches `main` or a test).

The compiler owns the derivation rules for seven traits: `Eq`, `Ord`, and `Show` from the Prelude; `Encode`, `Decode`, and `ObjectKey` from `Std.Dynamic`; and `Shaped` from `Std.Shape`. Writing any other trait in `derive` does not compile; user-defined derivation rules are left for a later version.

The serialization traits pair with `@key` on fields and `@tag` on variants to control wire names:

```emel
import Std.Dynamic.{Decode, Encode}
import Std.Json

type Point(@key("x_pos") x: Int, @key("y_pos") y: Int)
  derive Eq, Encode, Decode

fn main() {
  raw = Json.encode(Point(x: 3, y: 4))
  dbg(raw)
  decoded: Point = Json.decode(raw)
  dbg(decoded == Point(x: 3, y: 4))
  ()
}
```

```console
$ emela run
"{\"x_pos\":3,\"y_pos\":4}"
True
```

`derive` treats `opaque` types (chapter 7) according to whether a trait reads or constructs values:

- `Eq`, `Ord`, `Show`, `Encode`, and `Shaped` can be derived for an `opaque` type. Because a derived `Show` or `Encode` displays the hidden fields, write the `impl` by hand when the contents should stay private.
- `Decode` and `ObjectKey` cannot be derived for an `opaque` type. A derived decoder would construct values directly, bypassing the functions of the module that check its invariants.

## What traits do not have

Emela keeps traits small on purpose (`[trait.not]`): there are no associated types, no traits over higher-kinded types, no run-time dynamic dispatch, and no user-defined operator traits. Every call to a trait function resolves to a single `impl` determined by the types at compile time. When you want to swap implementations at run time, or hide which implementation stands behind an interface, Emela uses effects and handlers instead.

## Summary

- A `trait` declares functions that a type can provide, with `self` of type `Self`, optional default implementations, and optional supertraits (`trait A: B`). Trait functions are pure unless they write `fails` or `uses`.
- An `impl Type: Trait` implements a trait in the module of the type or the module of the trait. There is at most one `impl` for any `(Type, Trait)` pair in the program, and heads cannot specialize type arguments.
- Generic functions constrain type parameters with `[A: Trait]` or `[A: T1 + T2]`. A bound includes its supertraits. Functions with `Self` only in the return type select their `impl` from the expected type.
- `Eq`, `Ord`, and `Show` are built in; `Float` omits `Ord` because `NaN` breaks total order. `dbg` does not require `Show`.
- `derive` generates ordinary `impl`s for `Eq`, `Ord`, `Show`, `Encode`, `Decode`, `ObjectKey`, and `Shaped`. `Decode` and `ObjectKey` cannot be derived for `opaque` types.

Chapter 10 turns to the other side of abstraction: defining your own effects, handlers, layers, labels, and capabilities.
