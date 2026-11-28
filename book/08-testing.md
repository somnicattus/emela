# 8. Testing

A test is a function marked `@test`, and a check inside it is an `assert`. Code with effects is tested by running it under handlers that the test controls, the same mechanism chapter 6 used to replace the console. Nothing else is needed: there is no framework to set up and no mechanism for mock objects.

## Writing a test

The example of this chapter is a small project, `launch`, that counts down before a launch:

`src/countdown.emel`:

```emel
import Std.Clock.{Clock}
import Std.Console.{Console}

pub fn countdown(from: Int) -> () uses { Clock, Console } {
  clock <- Clock
  console <- Console
  List.each(List.range(0, from), |i| {
    console.print_line(label(from - i))
    clock.sleep(1000)
  })
  console.print_line("liftoff")
}

fn label(n: Int) -> String {
  if n == 1 { "1 second" } else { "#{n} seconds" }
}

@test
fn label_of_one_is_singular() {
  assert label(1) == "1 second"
}

@test
fn label_of_three() {
  assert label(3) == "3 seconds"
}
```

`src/main.emel`:

```emel
import Countdown
import Std.Platform.{Platform}

fn main() {
  with Platform {
    Countdown.countdown(3)
  }
}
```

```console
$ emela test --reporter plain
running 4 tests
test Countdown.label_of_one_is_singular ... ok
test Countdown.label_of_three ... ok
test CountdownTest.prints_each_second ... ok
test CountdownTest.takes_three_seconds ... ok

test result: ok. 4 passed; 0 failed; 0 filtered out
```

(The two tests in `CountdownTest` are in the next section.)

- `@test` goes on a top-level function without parameters. `emela test` runs every such function in the project, one at a time.
- A test can be written in the same file as the code it tests, as here. It can then call private functions such as `label`.
- A test is named after its function, with its module in front if it is not in the entry module: `Countdown.label_of_three`.
- Test functions are left out of a normal build, and only another test can call one.

`emela test` chooses how to show its results. In a terminal, it shows them for a person to read, in a layout that may change between versions; `--reporter plain` gives the line format used in this book, which stays the same for the same results (`[cmd.test.reporter]`).

To run only some of the tests, give a filter after the path: `emela test . label` runs the tests whose names contain `label`.

### When an assertion fails

`assert` takes a `Bool`, and when it is `False`, the test fails. If `label_of_three` expected `"3 second"`:

```console
$ emela test --reporter plain . label_of_three
running 1 test
test Countdown.label_of_three ... FAILED

failures:

---- Countdown.label_of_three ----
 --> src/countdown.emel:25:3
defect: assertion failed: label(3) == "3 second"
  left: "3 seconds"
 right: "3 second"

failures:
    Countdown.label_of_three

test result: FAILED. 0 passed; 1 failed; 3 filtered out
$ echo $?
1
```

The report gives the location of the `assert`, its source text, and, for a comparison, both sides as values. The values are shown when both sides have `Show`; otherwise only the source text appears. A failed `assert` is a defect (chapter 5): it ends that test, and the other tests still run. `assert` works outside tests too, as a check that the program stops on when it fails.

A test also fails when an error that it does not handle comes out of it, or when it stops with any other defect. To check that something fails, turn the error into a value and compare it (chapter 5):

```emel
error Empty

fn first(xs: List[Int]) -> Int fails Empty {
  xs match {
    [x, ..] -> x
    []      -> fail Empty
  }
}

@test
fn first_of_empty_fails() {
  result = Some(first([])) escape {
    Empty -> None
  }
  assert result == None
}
```

## Testing code with effects

`countdown` prints and sleeps. A test cannot call it directly: the effects of a test must all be provided inside the test, as for `main`. A test `needs_handlers` that calls `countdown(1)` without a `with` does not compile:

```console
$ emela test
error[E4301]: effects `Console`, `Clock` are not provided in test function `needs_handlers`
    ╭─[ src/countdown.emel:30:3 ]
    │
 30 │   countdown(1)
    │   ──────┬─────
    │         ╰───────
────╯
   = help: provide it with `with`

1 error
```

Running `countdown` under `SystemConsole` and `SystemClock` would work, but the test would print to the terminal, take three seconds, and have nothing to compare. The standard library has test handlers for several of them, named with `Test` in front (`[lib.layers]`):

- `TestConsole` appends the printed lines to a list, kept in a State.
- `TestClock` keeps a virtual time in a State. `sleep` waits only until nothing else in the program can run; then the clock moves forward to the time of the earliest sleep and wakes it. A test that sleeps for an hour finishes at once.
- `TestStdin`, `TestStdout` and `TestStderr` stand in for the standard streams of the process.

The handlers keep what they record in the State effect, so each is used inside `State.run`, which provides the State and returns its final value next to the result (chapter 6). With two of them, there are two `State.run`s, one inside the other. This test is in `test/countdown_test.emel`:

```emel
import Countdown
import Std.Clock.{TestClock}
import Std.Console.{TestConsole}
import Std.Instant
import Std.State.{State}

@test
fn prints_each_second() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  ((_, lines), _) = State.run(start, || {
    State.run([], || {
      with TestClock, TestConsole {
        Countdown.countdown(3)
      }
    })
  })
  assert lines == ["3 seconds", "2 seconds", "1 second", "liftoff"]
}

@test
fn takes_three_seconds() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  (_, end) = State.run(start, || {
    State.run([], || {
      with TestClock, TestConsole {
        Countdown.countdown(3)
      }
    })
  })
  assert Instant.to_unix_milliseconds(end) == Int64.from_int(3000)
}
```

The outer `State.run` holds the time for `TestClock`, and the inner one holds the lines for `TestConsole`. Each test sees what `countdown` did, the lines it printed and the time it spent, without a change to `countdown` and without passing it anything. Both tests together run in well under a second.

The same pattern scales to effects of your own. An application that declares its own effects, say for a database, also declares handlers for its tests, usually bundled in a layer, and a test runs the code under `with AppTest { ... }` (chapter 10).

## The test root

A project can keep tests apart from its sources, under `test/`, the *test root*:

```text
launch/
├── Pome.toml
├── src/
│   ├── countdown.emel
│   └── main.emel
└── test/
    └── countdown_test.emel     # module CountdownTest
```

- Every module under `test/` is only for testing. Its declarations are left out of a normal build even without `@test`, so it can also hold helpers for other tests, such as functions that build test data.
- A module under `test/` can import the modules under `src/`, but sees only their public names. It can also import the Pomes in `[dev-dependencies]` (chapter 7).
- A module under `src/` cannot import a module under `test/`.

Helpers for assertions, generators of test data and property testing are not in the standard library. They are in the Pome `github.com/emela-lang/brix`, which a project adds as a test dependency (`emela pome add --dev github:emela-lang/brix`).

## Summary

- A test is a function with `@test` and no parameters; `emela test` runs all of them. An `assert` that is `False` fails the test, with both sides shown.
- A test fails on a failed `assert`, an unhandled error, or any other defect. The other tests still run.
- A test provides all the effects of the code it tests. Test handlers such as `TestConsole` and `TestClock` record what the code does, in a State that `State.run` returns.
- Tests can sit next to the code, where they see private functions, or under `test/`, where they see only public ones.

## The end of Part I

Part I covered everything needed to write an application: data, functions, repetition, errors, effects, modules and tests. Everything in it used only the effects and handlers of the standard library.

Part II is the foundation those were built from. It is for writing libraries and the boundaries of a program: traits, effects and handlers of your own, resources that are released in time, concurrent tasks, and the ways these combine to structure a larger program. Chapter 9 starts with traits, which give names to what `derive Eq, Ord, Show` has been providing since chapter 1.
