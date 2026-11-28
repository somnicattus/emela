# 6. Using effects

A function that prints a line, reads the clock or draws a random number does more than compute its result: it acts on the world outside the program, or depends on it. Emela calls such an ability of a function, to print or to read the clock, an *effect*. The effects a function uses are part of its type, next to its errors, and they are provided from outside: the function says that it needs to print, and the code that calls it decides how printing is done.

That second half is what makes effects useful. The same function can print to the terminal when the program runs and into a list when a test runs, without a change to the function and without passing anything to it by hand.

This chapter uses the effects of the standard library. Chapter 10 shows how to declare your own.

## Hello, world, again

The program that `emela new` writes (chapter 1) is a small example of everything in this chapter:

```emel
import Std.Console.{Console, SystemConsole}

fn greet(name: String) {
  console <- Console
  console.print_line("Hello, #{name}!")
}

fn main() {
  with SystemConsole {
    greet("Ada")
    greet("Bob")
  }
}
```

```console
$ emela run
Hello, Ada!
Hello, Bob!
```

- `import Std.Console.{Console, SystemConsole}` makes two names of the module `Std.Console` visible: the effect `Console` and the handler `SystemConsole`. The effects and handlers of the platform live in modules under `Std` (chapter 7 covers modules and imports).
- `console <- Console` in `greet` says that the rest of the block uses the effect `Console`, under the name `console`. Such a name is an *effect alias*. `console.print_line(...)` calls an *operation* of the effect.
- `with SystemConsole { ... }` in `main` provides `Console` to everything that runs in its block. `SystemConsole` is a *handler*: it implements the operations of `Console` by writing to the standard output of the process.

`greet` does not say which handler it uses, and it could not: it only asks for `Console`. The handler is chosen by the `with` around the call.

## Effects in a function's type

The effects a function uses are part of its type, written after `uses`:

```emel
import Std.Console.{Console}
import Std.Platform.{Platform}

fn report(scores: List[(String, Int)]) -> () uses { Console } {
  console <- Console
  List.each(scores, |(name, score)| console.print_line("#{name}: #{score}"))
}

fn main() {
  with Platform {
    report([("Ada", 90), ("Bob", 72)])
  }
}
```

```console
$ emela run
Ada: 90
Bob: 72
```

`uses { Console }` can be left out, as in `greet`: the effects are always inferred when there is no `uses` (chapter 2), and the editor shows them. A function that calls `greet` or `report` uses `Console` too, and so on outwards, until a `with` provides it.

The effects in a type are a set, as errors are: `uses { Console, Clock }`. A function with no effects in its type does not act on the world, apart from the debugging output of `dbg` (chapter 1), and its result depends only on its arguments.

`List.each` calls a function on each element for what that function does (chapter 4). The anonymous function here prints, so the call of `List.each` uses `Console`: the effects of a function passed as an argument go on to the call, as its errors do (chapter 3).

### Nothing provides it

`main` is where every effect has to be provided. A program whose `main` calls `greet` without a `with` does not compile:

```console
$ emela check
error[E4301]: effect `Console` is not provided in `main`
   ╭─[ src/main.emel:9:3 ]
   │
 9 │   greet("Ada")
   │   ──────┬─────
   │         ╰───────
───╯
   = help: provide it with `with`

1 error
```

The only effects that `main` may leave to the host are the ones the host itself provides, such as being able to wait (`[effect.with]`, chapter 12).

## Providing effects

`with` lists one or more items, separated by commas, and provides their effects to its block. The value of the `with` is the value of its block.

The standard library has a handler for each effect of the platform, named after the effect with `System` in front: `SystemConsole`, `SystemClock`, `SystemRandom`, and so on (`[lib.layers]`). Two *layers* bundle them:

- `Platform` (in `Std.Platform`) has the handlers that every host has: Console, Clock, Random and Http.
- `Process` (in `Std.Process`) has `Platform` and the handlers that need a process: its arguments, standard input and output, environment variables, files and TCP.

So `main` is usually `with Platform { ... }`, or `with Process { ... }` when it reads files or arguments. A layer is used in a `with` like a handler.

Operations of the effects of the platform can fail. Reading a line of standard input, for example, can fail with `InvalidInput` when the input is not valid text. An operation that can fail says so with `fails` in the declaration of its effect, and a call of it fails like a call of a function that can fail (chapter 5).

### Replacing a handler

Because a function names only the effect, the handler can be replaced around any call. When items of one `with` provide the same effect, the later one wins:

```emel
import Std.Clock.{Clock}
import Std.Console.{Console, TestConsole}
import Std.Platform.{Platform}
import Std.State.{State}

fn countdown(from: Int) {
  clock <- Clock
  console <- Console
  List.each(List.range(0, from), |i| {
    console.print_line("#{from - i}")
    clock.sleep(100)
  })
  console.print_line("go")
}

fn main() {
  (_, lines) = State.run([], || {
    with Platform, TestConsole {
      countdown(3)
    }
  })
  dbg(lines)
  ()
}
```

```console
$ emela run
["3", "2", "1", "go"]
```

`with Platform, TestConsole` takes Clock and the rest from `Platform`, and Console from `TestConsole`. `TestConsole` does not write anywhere; it appends each line to a list kept in the State effect. `State.run([], || ...)` runs a function with that list, starting empty, and returns the function's result together with the final list. `countdown` is unchanged, and still sleeps between lines on the real clock.

This is how the code that uses an effect stays apart from the code that decides how it is done. A test uses the same mechanism to replace the console, the clock or a database with handlers it controls (chapter 8). There is nothing to pass by hand: the handler reaches every function called inside the `with`, at any depth.

## Effects and functions as values

A function value carries its effects in its type, as `fn() -> () uses { Console }`, and the effects are provided where the function is called, not where it is written. An anonymous function that prints is fine to create anywhere; it is calling it that needs `Console`.

A parameter whose function type writes `uses {}` accepts only functions without effects:

```emel
fn evaluate(f: fn(Int) -> Int uses {}, x: Int) -> Int {
  f(x)
}
```

Passing an anonymous function that prints to `evaluate` does not compile. Leaving `uses` out of a parameter's function type, as `List.map` and `List.each` do, accepts any effects and passes them on to the call (chapter 3).

## Waiting

Some operations wait. `clock.sleep(100)` in `countdown` waits a tenth of a second before the program goes on. In the declaration of `Clock`, `sleep` is a `suspend fn`, an operation that may suspend the code that calls it.

Nothing marks the call: `clock.sleep(100)` is written like any other call, and so is a call of `countdown`. Whether a call can wait follows from the effects in the type of what it calls, so the program does not need to say it again at each call. Chapter 12 shows what waiting makes possible: running several tasks at the same time.

## Summary

- An effect is something a function asks to be able to do, such as printing or reading the clock. A function uses it with `name <- Effect` and calls its operations as `name.operation(...)`.
- The effects of a function are in its type, as `uses { A, B }`, and are inferred when `uses` is left out. They go on to every caller until a `with` provides them.
- `with Handler { ... }` provides the effects of a handler to its block. `main` provides everything, usually with the layer `Platform` or `Process`.
- A function never names its handler, so a later item in a `with` can replace one, for example `TestConsole` for `SystemConsole`.
- Functions passed as arguments pass their effects on to the call; `uses {}` in a parameter's type accepts only functions without effects.
- A call that may wait is written like any other call.

Chapter 7 is about organizing a program into modules, and about using code from other packages.
