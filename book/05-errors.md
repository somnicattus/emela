# 5. Errors

Some operations cannot always produce their result: a user may not exist, a string may not be a number. Emela calls such an outcome an *error*. An error is part of what a function promises: it appears in the function's type, the compiler checks that every caller deals with it, and the code that deals with it is written apart from the code that computes the result.

A mistake in the program itself, such as dividing by zero, is not an error in this sense. It is a *defect*, which no type mentions and which stops the program. This chapter covers both, and how to tell which one a situation calls for.

There are two tools: `fail` raises an error, and `escape` handles one. Between the two, nothing needs to be written: an error that a function does not handle goes on to its caller.

## Declaring an error

An error is declared with `error`, in the same form as a type:

```emel
error NotFound(id: Int) derive Show

error Banned(name: String) derive Show

type User(id: Int, name: String, banned: Bool)

const USERS = [User(id: 1, name: "Ada", banned: False), User(id: 2, name: "Bob", banned: True)]

fn find_user(id: Int) -> User fails NotFound {
  USERS |> List.find(|u| u.id == id) |> Option.or_fail(NotFound(id:))
}

fn active_user(id: Int) -> User fails NotFound | Banned {
  user = find_user(id)
  if user.banned { fail Banned(name: user.name) }
  user
}
```

- `error NotFound(id: Int)` declares a type and a constructor, as a `type` does. `NotFound(id: 1)` is a value that can be bound, passed and stored like any other.
- An error can also have no fields: `error Unauthorized`.
- An error is told apart from other errors by its name, its *tag*. `NotFound(id: 1)` and `NotFound(id: 2)` are the same error with different contents.
- `derive Show` lets the error be shown, which matters when it reaches `main` (below).

### Raising an error

`fail` ends the computation with an error. In `active_user`, `fail Banned(name: user.name)` ends the function, and the caller receives the error in place of a result.

The type of `fail ...` is `Never`, the type with no values from chapter 1: the expression never produces a value, so it fits anywhere a value of any type is expected. That is why `if user.banned { fail ... }` needs no `else`, and why `fail` can stand in one branch of an `if` that produces a value in the other.

`Option.or_fail(opt, err)` is the bridge from an `Option`: it returns the contents of `Some`, or fails with `err` on `None`.

### The errors of a function

A function's errors are part of its type. `find_user` is written `-> User fails NotFound`: it returns a `User`, or fails with `NotFound`. The errors are a set of tags separated by `|`, as in `fails NotFound | Banned`; their order does not matter.

`active_user` calls `find_user` and does nothing about `NotFound`. The error goes on to the caller of `active_user` by itself, and `NotFound` is in the `fails` of `active_user` too. There is no operator to write at the call: every call can pass an error on.

The signature decides how strictly the errors are checked (`[error.fails]`):

| Written | Meaning |
|---|---|
| No result type | The errors are inferred, together with the result type |
| `-> T` | The function does not fail. An error it does not handle is a compile error |
| `-> T fails A \| B` | The function fails with exactly these errors |

So writing `-> User` for `active_user`, without its `fails`, does not compile:

```console
$ emela check
error[E3201]: unhandled error `NotFound`
    ╭─[ src/main.emel:14:10 ]
    │
 14 │   user = find_user(id)
    │          ──────┬──────
    │                ╰────────
────╯
   = help: handle it with `escape`, or declare it with `fails`

error[E3201]: unhandled error `Banned`
    ╭─[ src/main.emel:15:20 ]
    │
 15 │   if user.banned { fail Banned(name: user.name) }
    │                    ──────────────┬─────────────
    │                                  ╰───────────────
────╯
   = help: handle it with `escape`, or declare it with `fails`

2 errors
```

For each error that the signature does not allow, the compiler points at the first expression that raises it.

## Handling an error

`escape` handles the errors of the expression before it. It is written after the expression, with arms, in the same form as `match`. The following functions use the declarations above:

```emel
error Forbidden(reason: String) derive Show

fn describe(id: Int) -> String {
  active_user(id).name escape {
    NotFound(_)   -> "nobody"
    Banned(name:) -> "#{name} (banned)"
  }
}

fn name_or_guest(id: Int) -> String fails Banned {
  active_user(id).name escape {
    NotFound(_) -> "guest"
  }
}

fn checked_name(id: Int) -> String fails NotFound | Forbidden {
  active_user(id).name escape {
    Banned(name:) -> fail Forbidden(reason: "#{name} is banned")
  }
}

fn main() {
  dbg(describe(1))
  dbg(describe(2))
  dbg(describe(3))
  dbg(name_or_guest(3))
  dbg(checked_name(1))
  refused = checked_name(2) escape {
    _ -> "refused"
  }
  dbg(refused)
  ()
}
```

```console
$ emela run
"Ada"
"Bob (banned)"
"nobody"
"guest"
"Ada"
"refused"
```

When the expression gives a value, the value goes through and the arms are not used. When it fails, the arm for the error gives the value instead. Each function shows one way to use this.

- `describe` recovers from every error. The arms handle both `NotFound` and `Banned`, so `describe` does not fail, and its signature `-> String` says so.
- `name_or_guest` handles only `NotFound`. An `escape` does not have to handle every error: the others go on to the caller, and `Banned` stays in its `fails`.
- `checked_name` turns one error into another. An arm can `fail` too, and its error is added to the errors of the function.
- `_` handles every error. It is for when the error itself does not matter, as for `refused`.

An arm of an `escape` matches an error by its constructor, with patterns for its fields, or with `_`. It cannot bind the error to a variable: the errors handled by one `escape` can be of different types, and no type holds "one of `NotFound` or `Banned`". An arm whose pattern matches only some values of an error, such as `NotFound(id: 1)`, handles those values; the error stays among the errors of the function, because the other values still go on. An arm for an error that the expression cannot raise gets a warning (W3201).

An arm gives a value of the same type as the expression, or fails. So to keep the error as information, change the value first: `Some(find_user(id)) escape { NotFound(_) -> None }` turns the error into an `Option`.

### Errors from functions passed as arguments

A function that takes a function passes the errors of that function on (chapter 3):

```emel
error Negative(n: Int) derive Show

fn check(n: Int) -> Int fails Negative {
  if n < 0 { fail Negative(n:) }
  n
}

fn main() {
  dbg(List.map([1, 2], check))
  fixed = List.map([1, -2, 3], check) escape {
    Negative(n:) -> [n]
  }
  dbg(fixed)
  ()
}
```

```console
$ emela run
[1, 2]
[-2]
```

`List.map(xs, check)` fails with `Negative` when `check` does, and stops at that element. A function that accepts only functions that do not fail says so in the parameter's type, `f: fn(Int) -> Int fails {}`; passing `check` to it does not compile.

## Option or error

Both an `Option` and an error say that there may be no result. The standard library chooses between them in a way that works for your own functions too (`[lib.api-rules]`):

- When looking something up may find nothing, or a position may be out of range, return an `Option`: `List.get`, `List.find`. A missing result is an ordinary answer, and the caller usually decides what to do next on the spot.
- When the input is malformed, fail with an error: `Int.parse(s)` fails with `ParseError`. The caller rarely has a sensible value to use in place of the result.
- When something outside the program goes wrong, such as a file that cannot be read, the operation of the effect fails with an error (chapter 6).
- When the caller has broken a rule that it was required to keep, it is a defect.

Converting is short in both directions: `Option.or_fail(opt, err)` from an `Option` to an error, and `Some(f(x)) escape { _ -> None }` from an error to an `Option`.

## When an error reaches main

An error that no function handles reaches `main`, and the program ends:

```emel
error NotFound(id: Int) derive Show

fn find(id: Int) -> String fails NotFound {
  if id == 1 { "Ada" } else { fail NotFound(id:) }
}

fn main() {
  dbg(find(1))
  dbg(find(9))
  ()
}
```

```console
$ emela run
"Ada"
error: NotFound(id: 9)
$ echo $?
1
```

The line `error: ...` goes to standard error, and the exit code is 1. The error is shown with its `Show`; an error without `Show` is shown by its name alone, as `error: NotFound`.

## Defects

A defect is a mistake that the program cannot be expected to recover from. It does not appear in any type, and `escape` does not handle it.

```emel
fn average(xs: List[Int]) -> Int {
  List.fold(xs, from: 0, f: |sum, x| sum + x) / List.length(xs)
}

fn main() {
  dbg(average([1, 2, 3]))
  dbg(average([]))
  ()
}
```

```console
$ emela run
2
defect: division by zero
$ echo $?
101
```

Defects come from:

- `panic(message)`, for a situation the code has ruled out, and `todo()`, for code not written yet;
- dividing an integer by zero, and a few other operations whose input is outside what they accept, such as converting `NaN` to an `Int`;
- a failed `assert` in a test (chapter 8);
- an exception thrown by a function of the host (chapter 16).

When a defect reaches the top of the program, it writes `defect: ...` to standard error and exits with code 101, so that it can be told apart from an error (1). Before it does, the clean-up of the resources the program holds still runs (chapter 11).

To turn an error into a defect, `panic` in an arm: `load_config(path) escape { _ -> panic("the bundled config must load") }`. A defect cannot be turned back into an error inside ordinary code. It is observed only at a boundary: the top of the program, the test runner, or `Exit.attempt` (chapter 12).

`average([])` fails because the code divides by the length without asking whether the list is empty. Whether that should be a defect or an error is a decision about `average`: if an empty list is a mistake of the caller, the defect is right; if it is an input to expect, `average` should check it and return an `Option` or fail with an error of its own.

## Summary

- An error is declared with `error`, raised with `fail`, and handled with `escape`. An error that is not handled goes on to the caller without anything written at the call.
- A function's errors are in its type, as `fails A | B`. A function that writes its result type and no `fails` does not fail.
- `escape` can handle all the errors of an expression, some of them, or turn them into other errors or into values.
- An error is told apart from others by its tag. Arms match tags, or `_`.
- Use an `Option` for a result that may be absent, an error for input that may be wrong, and a defect for a mistake in the program.
- An error that reaches `main` ends the program with exit code 1; a defect ends it with 101.

Chapter 6 is about the other thing in a function's type besides its errors: the effects it uses.
