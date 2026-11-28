# Introduction

Emela is a functional programming language with typed errors and typed effects. It compiles to JavaScript, WebAssembly and C, and the same program gives the same result on all three.

This chapter says what kind of language Emela is, how to install it, and what a small program in it looks like. The rest of the book takes the pieces of that program one at a time.

## Three pillars

Emela keeps three things simple, and every feature of the language belongs to one of them (`[intro.pillars]`). A feature that belongs to none is not added.

- **Defining data**, with `type`, `enum` and `error`. Data is defined first, as the cases it can be and the fields each case has.
- **Functions that operate on data**, with `fn`, `match` and the pipe `|>`. A function takes values apart by their shape and builds new ones.
- **Making side effects explicit**, with `effect`, `use` and `with`. What a function does to the world outside, such as printing, reading a file or waiting, is part of its type, and the code that calls it decides how it is done.

A few decisions follow from these and do not change (`[intro.premises]`):

- Values never change, and there is no loop syntax. Repetition is written with functions over collections, or with functions that call themselves.
- There is no null. A value that may be missing has an `Option` type.
- Errors are not effects. A function's errors are in its type, apart from its effects, and a mistake in the program, such as dividing by zero, is a third thing, a *defect*, which no type mentions.
- There are no implicit conversions between types and no operators defined by users.

## Two levels

The language has two levels (`[intro.layers]`). People who write applications use a small part of it: data, functions, `match`, errors, and the effects and handlers that the standard library provides. People who write libraries and the boundaries of a program also declare their own effects and handlers, traits, and functions of the host. Writing an application never requires the second level.

The book follows the same split. Part I is everything needed to write an application. Part II is the foundation that Part I is built on. Part III is about the hosts and targets a program runs on, and Part IV is a reference for the tools.

## Installing Emela

Prebuilt binaries are published for macOS (Apple Silicon), Linux (x86_64) and Windows (x86_64). On macOS and Linux:

```console
$ curl -fsSL https://emela.dev/install.sh | sh
$ emela --version
```

This installs `emela` into `$HOME/.emela/bin`. On Windows, download `emela-<version>-x86_64-pc-windows-msvc.zip` from the releases of the repository `github.com/emela-lang/emela`, and put `emela.exe` on the `PATH`.

Running a program needs a host for its target:

| Target | Needs |
|---|---|
| JavaScript, the default | Node.js 22 or later, or Bun 1.4.3 or later |
| WebAssembly (`--target wasm`) | Wasmtime 49 or later |
| Native (`--target native`) | A C11 compiler, as `cc` |

`emela doctor` reports which of them are installed. The native target does not run on Windows yet.

The repository also has syntax highlighting and language server setups for several editors, under `editors/`. The language server shows the inferred types, errors and effects of every function, which helps from the first chapter on.

## A first look

Here is a small program that prices orders at a café. It is the whole of `src/main.emel` in a project made with `emela new cafe`:

```emel
import Std.Console.{Console, TestConsole}
import Std.Platform.{Platform}
import Std.State.{State}

# Data: a closed set of cases, each with its own fields.
enum Item {
  Coffee(shots: Int)
  Tea
  Cake(slices: Int)
}

# An error is declared like data, and is part of a function's type.
error UnknownItem(name: String) derive Show

fn parse(name: String) -> Item fails UnknownItem {
  name match {
    "espresso" -> Coffee(shots: 1)
    "doppio"   -> Coffee(shots: 2)
    "tea"      -> Tea
    "cake"     -> Cake(slices: 1)
    _          -> fail UnknownItem(name:)
  }
}

fn price(item: Item) -> Int {
  item match {
    Coffee(shots:) -> 250 + 100 * shots
    Tea            -> 300
    Cake(slices:)  -> 450 * slices
  }
}

# Printing is an effect: it is in the type, and provided from outside.
fn receipt(order: List[String]) -> () fails UnknownItem uses { Console } {
  console <- Console
  total = order |> List.map(parse) |> List.map(price) |> List.fold(from: 0, f: |sum, p| sum + p)
  console.print_line("#{List.length(order)} items: #{total} yen")
}

fn main() {
  with Platform {
    receipt(["doppio", "cake", "tea"])
    console <- Console
    receipt(["espresso", "scone"]) escape {
      UnknownItem(name:) -> console.print_line("sorry, no #{name}")
    }
  }
}

# A test runs the same code with a handler that records what it prints.
@test
fn prints_the_total() {
  (_, lines) = State.run([], || {
    with TestConsole {
      receipt(["tea", "tea"])
    }
  })
  assert lines == ["2 items: 600 yen"]
}
```

```console
$ emela run
3 items: 1200 yen
sorry, no scone
$ emela test --reporter plain
running 1 test
test prints_the_total ... ok

test result: ok. 1 passed; 0 failed; 0 filtered out
```

`--reporter plain` selects the line format of test results that this book shows (chapter 8).

Read from the top, the program uses most of Part I.

- `Item` is an `enum`: an item is a coffee with some number of shots, a tea, or some slices of cake, and nothing else (chapter 1).
- `parse` and `price` take values apart with `match`, which has to handle every case. A new kind of item would not compile until `price` handles it (chapter 2).
- `receipt` chains its steps with the pipe `|>`, passes functions to `List.map`, and adds up the prices with `List.fold` instead of a loop (chapters 3 and 4).
- `parse` can fail with `UnknownItem`, and says so in its type. `receipt` does not handle the error, so the error goes on through `List.map` to its caller, and `receipt` says `fails UnknownItem` too. `main` handles it with `escape` (chapter 5).
- `receipt` prints, so its type says `uses { Console }`. It does not choose how printing is done: `main` provides the handlers of the platform with `with Platform`, and the test provides `TestConsole`, which collects the lines in a list instead (chapters 6 and 8).

The same program runs on WebAssembly without a change:

```console
$ emela run --target wasm
3 items: 1200 yen
sorry, no scone
```

`emela run --target native` builds an executable with the C compiler. The specification requires the same result from all three targets; chapter 15 explains what that covers and where the targets differ today.

## How to read this book

The chapters build on each other in order. If you already know some of the ideas from another language, the reading paths in the [README](README.md) suggest where to start and what to skip. The book does not compare Emela with other languages in its chapters; Appendix A does that, one language at a time.

The book describes the language as the specification in `spec/` defines it. Where a rule matters, the book gives the name of its section in brackets, as `[intro.pillars]`, so that you can look up the exact wording. Where the book and the specification disagree, the specification is right.

The outputs shown are those of the JS target unless the text says otherwise. `emela run` first writes the command it runs, a line starting with `running`, to standard error; the book leaves that line out.

Chapter 1 starts with data.
