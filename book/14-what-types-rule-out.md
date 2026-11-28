# 14. What the types rule out

Throughout Parts I and II, individual chapters introduced errors, effects, `Immediate`, capabilities, and `match`. Together they form one type and effect system (`[type]`). This chapter puts the pieces side by side and summarizes the guarantees the compiler checks before a program runs.

## Every expression has three parts: `T`, `E`, `R`

Type checking assigns every expression a triple (`[type.judgment]`):

```text
T ! E ; R
```

- `T` is the value type, inferred in the Hindley-Milner style. `Never` joins with any type at branch joins (`if`, `match`, `escape`) and expected-type positions, so `fail` and `panic` fit anywhere; inside type arguments, `List[Never]` and `List[Int]` remain distinct (`[type.rules]`).
- `E` is the set of error tags the expression may raise with `fail`. Empty means the expression cannot fail with a typed error.
- `R` is the set of effects, labels, and capabilities the expression needs. Empty means the expression does not depend on the environment.

A function whose `E` and `R` are both empty is *pure*.

How much you write in a function's signature determines what is inferred and what is enforced (`[type.inference]`):

| Part | When omitted | When written |
|---|---|---|
| Return type `-> T` | `T` and `E` are inferred | `T` is checked, and `E` must stay within `fails` |
| `fails` clause | Empty (no errors) if `-> T` is written | Fixes the function's errors to that set (the body's errors must stay within it) |
| `uses` clause | `R` is always inferred (even with `-> T`) | Fixes the function's effects to that set (the body's effects must stay within it) |

Even when `fails` or `uses` is written, the hidden error and effect sets of parameter functions and the capability sets of capability parameters are carried alongside the written set automatically. Writing `-> User` without `fails` guarantees that the function handles every error it could raise; writing `uses { Console }` guarantees that it uses no other effect. At `main`, tests, and `@export` functions, `R` must be empty except for the host effects the entry provides (`[effect.with]`, `[effect.host]`), so no required handler can ever be missing at run time.

## Function types and set type parameters

In a function type `fn(A) -> B fails E uses { R }`, what an omitted `fails` or `uses` means depends on *where* the type is written (`[type.fn]`):

| Position | Meaning when omitted |
|---|---|
| Inside a parameter type (`f: fn(A) -> B`, `fs: List[fn(A) -> B]`) | Any errors or effects (a hidden set variable that propagates to the caller) |
| Return type, binding annotation, anonymous function parameter | Inferred from the body |
| Inside a declaration (`type`, `enum`, `error`, `handler`, `effect`, `trait`, `const`) | Empty (pure) |

This split lets higher-order functions like `List.map` accept both pure and effectful callbacks without extra syntax, while preventing data structures and effect operations from hiding effects. To store an effectful function in a `type`, expose its error and effect sets as type parameters:

```emel
import Std.Console.{Console, SystemConsole}

error BadInput derive Show

type Endpoint[E, R](handle: fn(String) -> String fails E uses { R })

fn run_endpoint[E, R](ep: Endpoint[E, R], input: String) -> String fails E uses { R } {
  f = ep.handle
  f(input)
}

fn pure_echo(s: String) -> String {
  "echo: #{s}"
}

fn logged_echo(s: String) -> String fails BadInput uses { Console } {
  if s == "" { fail BadInput }
  console <- Console
  console.print_line("received #{s}")
  s
}

fn main() {
  with SystemConsole {
    endpoints: List[Endpoint[BadInput, Console]] = [
      Endpoint(handle: pure_echo),
      Endpoint(handle: logged_echo),
    ]
    out = List.map(endpoints, |ep| run_endpoint(ep, "hi"))
    dbg(out)
    ()
  }
}
```

```console
$ emela run
received hi
["echo: hi", "hi"]
```

Whether a type parameter stands for an ordinary type, an error set, or an effect set is inferred from where it is used (`fails E`, `uses { R }`), with no kind syntax needed. Multiple elements in a set type argument use the same notation as `fails` and `uses`: `Endpoint[BadInput | Timeout, { Console, Clock }]`, or `{}` for an empty set.

### Subtracting from sets in generic functions

Both `escape` and `with` subtract what they handle from `E` and `R`, but neither can subtract from an *opaque* set variable (`[error.escape]`, `[effect.with]`). In `fn run(f: fn() -> ()) { with ConsoleLogger { f() } }`, the hidden effect set of `f` stays in `run`'s `R` even though `ConsoleLogger` is provided inside.

To remove a specific error or effect in the type as well, name it next to a set type parameter so it sits outside the variable:

```emel
fn run[R](f: fn() -> () uses { Logger, R }) -> () uses { R } {
  with ConsoleLogger { f() }
}
```

## Containment of `E` and `R`

In the `Endpoint` example above, `pure_echo` (with empty `E` and `R`) is placed into `Endpoint[BadInput, Console]` alongside `logged_echo`. That works because of *containment* (`[type.rules]`):

- A function with a smaller `E` or `R` can be used anywhere a function with a larger `E` or `R` is expected. A pure function fits any error or effect set.
- When functions with different `E` or `R` meet at branches of an `if` or `match`, or as elements of a list, their `E` and `R` are joined by set union.
- Inside the parameter position of a function type, containment reverses. A runner that accepts a callback using `Console` (`fn(fn() -> Int uses { Console }) -> Int`) can be passed where a runner taking only a pure callback (`fn(fn() -> Int uses {}) -> Int`) is expected: anyone calling it through the expected type can only supply a pure callback, which the runner already accepts.
- Inside the type arguments of a named type (`Endpoint[E, R]`), sets match by equality: `Endpoint[{}, {}]` and `Endpoint[BadInput, Console]` are distinct types, while `Endpoint(handle: pure_echo)` works because `pure_echo` itself is checked against the expected function type `fn(String) -> String fails BadInput uses { Console }`.

Containment of `E` and `R` is the only form of subtyping in Emela (`[type.not]`). Value types themselves have no subtyping.

## `Immediate`: ruling out unexpected suspension

As chapter 12 showed, `R: Immediate` guarantees that an expression completes on the spot without suspending (`[type.effect-bound]`):

- Every `fn` operation of an `effect` (and every non-`suspend` handler member) is checked so callers can trust that calling it will never suspend, regardless of which handler is installed.
- Callbacks passed to `@external` functions default to `Immediate` (and any written `uses` clause on them must also be `Immediate`), so host code that expects synchronous return can never receive a suspending callback.
- Because `Immediate` is computed from the effects in `R`, and a handler that resumes on the spot removes its effect without adding `Async`, the same generic code works in both suspending and non-suspending contexts without duplicating functions.

## Capabilities that cannot leak

Capabilities (`use E`) and capability-bound types (`type Handle uses Files`, `Task[A] uses TaskGroup`) carry hidden capability sets in their types (`[type.rules]`, `[effect.escape]`):

- Calling an operation on a capability adds *that capability*, not the effect name `E`, to `R`. A capability is never widened to its effect name, so only the specific `with` that created it can remove it from `R`.
- Returning a capability, returning a capability-bound value like `Task[A]`, returning a closure that captures one, or raising an error whose field holds one out of its `with` does not compile:

```emel
import Std.Task.{Task}

fn leak_task() {
  with tasks = Task.Group {
    tasks.start(|| 1)
  }
}
```

```console
$ emela check
error[E4405]: capability `tasks` leaves the `with` that made it
   ╭─[ src/main.emel:5:5 ]
   │
 4 │   with tasks = Task.Group {
   │                ─────┬────
   │                     ╰────── made here; it ends with this `with`
 5 │     tasks.start(|| 1)
   │     ────────┬────────
   │             ╰──────────
───╯

1 error
```

- Trying to store a capability in an outer `State` or pass it to an outer handler operation also fails to type check, because the outer type was fixed before the inner `with` existed.
- A function type with a capability parameter (`fn(use ReadFile) -> String`) quantifies over an unknown capability set, so a closure passed to it can call operations on the capability while running, but cannot return it or smuggle it out.

Together with `init` and `release` (chapter 11), these rules eliminate use-after-close bugs at compile time.

## Exhaustiveness and refutability

Every `match` must cover every possible value of its scrutinee (`[fn.match]`):

```emel
enum Status {
  Pending
  Running(worker: Int)
  Done(code: Int)
}

fn label(s: Status) -> String {
  s match {
    Pending             -> "pending"
    Running(w) if w > 0 -> "running"
    Done(code:)         -> "done (#{code})"
  }
}
```

```console
$ emela check
error[E3102]: non-exhaustive match: pattern `Running(_)` not covered
   ╭─[ src/main.emel:8:3 ]
   │
 8 │   s match {
   │   ┬
   │   ╰──
───╯

1 error
```

- Arms with a guard (`if w > 0`) are not counted toward exhaustiveness, because the compiler does not evaluate arbitrary conditions.
- Adding a variant to an `enum` makes every `match` on it without a wildcard fail to compile until the new case is handled.
- Redundant arms already covered by earlier unguarded arms produce a warning.
- Even a `match` on `Never` must have at least one arm (an empty `match {}` does not compile).
- Bindings (`x = ...`) accept only irrefutable patterns; a refutable pattern like `Some(x) = opt` does not compile.
- By contrast, `escape` is *not* required to be exhaustive (`[error.escape]`): an error tag is removed from `E` only when unguarded arms for that error cover all of its values (`NotFound(_)`); partial patterns (`NotFound(id: 1)`) or guarded arms leave the tag in `E` and let unmatched errors propagate to the caller.

## What the type system leaves out

The guarantees above stay predictable because of what Emela chooses *not* to include (`[type.not]`):

- No general subtyping (only containment of `E` and `R`).
- No general union or intersection types (unions appear only in error sets `E`, and combinations of capabilities only in `use E1 + E2`).
- No higher-kinded types, no row polymorphism (field access and method calls require a known named type), and no polymorphic recursion.
- No null, no implicit type conversions, and no dynamic typing.

## Summary

- Every expression has a triple `(T, E, R)`: value type, error set, and effect set. All errors must be handled or declared in `fails`; all effects must be provided by `main` or a test.
- Omitted `fails` and `uses` clauses are polymorphic in function parameters, inferred in return types and bindings, and empty inside declarations.
- Containment lets functions with smaller `E` and `R` be used where larger sets are expected, reversing inside function parameters, with no general subtyping on value types.
- `Immediate` prevents suspending code from entering non-suspending operations or host callbacks.
- Capability tracking ensures neither capabilities (`use E`) nor capability-bound values (`Task[A]`) can outlive their `with`.
- Exhaustive `match`, irrefutable bindings, and exact error subtraction in `escape` ensure no unhandled case silently slips through.

## The end of Part II

Part II covered the foundation of the language: traits and `derive`, defining your own effects, handlers, layers, and labels, deterministic resource cleanup with `init` and `release`, suspension and structured concurrency, designing programs by composition, and the guarantees of the type and effect system.

Part III crosses the boundary to the host and targets: how the same program runs on JavaScript, WebAssembly, and C, how `@external` and `@export` bridge Emela to host code, and how the current compiler implements these features. Chapter 15 begins with the three targets.
