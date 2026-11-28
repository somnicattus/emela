# 12. Suspension and structured concurrency

Chapter 6 showed that `clock.sleep(100)` looks like any other call, even though it waits before the program goes on. This chapter explains how suspension works, how the type system tracks whether code can suspend without marking calls or functions, and how structured concurrency builds tasks, combinators, and synchronization on top of handlers (`[effect.decl]`, `[effect.async]`, `[type.effect-bound]`, `[task]`, `[dyn.suspend]`).

## `suspend fn`, `resume`, `Async` and `Immediate`

Only an operation declaration in an `effect` says that it may suspend, by writing `suspend fn` instead of `fn` (`[effect.decl]`):

```emel
import Std.Clock.{Clock}

effect Delay {
  suspend fn pause(ms: Int) -> ()
}

handler InstantDelay: Delay {
  suspend fn pause(ms, resume) {
    resume(())
  }
}

handler RealDelay: Delay {
  suspend fn pause(ms, resume) {
    clock <- Clock
    clock.sleep(ms)
    resume(())
  }
}

fn step(n: Int) -> Int uses { Delay } {
  delay <- Delay
  delay.pause(50)
  n + 1
}

fn run_immediate[R: Immediate](f: fn() -> Int uses { R }) -> Int uses { R } {
  f()
}

fn main() {
  result = run_immediate(|| with InstantDelay { step(41) })
  dbg(result)
  ()
}
```

```console
$ emela run
42
```

- `step` calls `delay.pause(50)` with ordinary call syntax, and its signature writes only `uses { Delay }`. Whether a function can suspend follows from the effects in its `R`.
- The implementation of a `suspend fn` receives an extra last parameter `resume: fn(T) -> () fails {} uses {}`, where `T` is the operation's return type, and the implementation body itself must have type `()` (`[effect.handler]`). Calling `resume(v)` continues the suspended caller with `v` as the value of the call. A continuation may be resumed *at most once*; a second call is a defect (`[dyn.suspend]`).
- The handler decides whether it actually suspends. `InstantDelay` calls `resume(())` directly on the spot, so `InstantDelay` does *not* require `Async` (`[effect.async]`). When `with InstantDelay { step(41) }` provides `Delay`, `Delay` is removed from `R` and nothing non-Immediate is added, so the whole expression has an `Immediate` effect set and passes `run_immediate`. By contrast, `RealDelay` calls `clock.sleep(ms)`, so its requirements include `Clock`, and `with RealDelay { step(41) }` cannot be passed to `run_immediate`.
- `Async` is a host effect that means "may resume later" (`[effect.async]`, `[effect.host]`). Users cannot write a handler for `Async` or obtain it with `use`; only the host provides it (to `main`, tests, and exports). Whenever a `suspend fn` implementation uses `resume` in any way other than a direct call (storing it in a variable, capturing it in a closure, or passing it to a host callback), or calls an `@external suspend fn`, the compiler adds `Async` to the handler's requirements.
- In a `suspend fn` implementation, evaluating a call that may suspend after `resume` may already have been called does not compile (`[effect.handler]`). Work that must continue after resuming the caller is started in a `Task.Background` instead.
- An ordinary `fn` operation cannot suspend (`[effect.decl]`): its body must either have an `Immediate` `R` or contain no call that may suspend.

### Enforcing `Immediate`

Some places, notably callbacks passed to host functions (chapter 16), cannot suspend. Instead of marking every function and call in the language, Emela expresses "must not suspend" with the built-in effect-set constraint `R: Immediate` (`[type.effect-bound]`):

- An effect is `Immediate` when it has no `suspend fn`, no `suspend handler` member, and is not `Async`.
- An effect set is `Immediate` when all of its elements are `Immediate` (the empty set is `Immediate`).

Passing a function that uses `Clock` (whose `sleep` is a `suspend fn`) to `run_immediate` does not compile:

```emel
import Std.Clock.{Clock, SystemClock}

fn wait_and_add(n: Int) -> Int uses { Clock } {
  clock <- Clock
  clock.sleep(10)
  n + 1
}

fn run_immediate[R: Immediate](f: fn() -> Int uses { R }) -> Int uses { R } {
  f()
}

fn main() {
  with SystemClock {
    dbg(run_immediate(|| wait_and_add(41)))
    ()
  }
}
```

```console
$ emela check
error[E4302]: effect `Clock` may suspend, but `R` of `run_immediate` must be `Immediate`
    ╭─[ src/main.emel:15:9 ]
    │
 15 │     dbg(run_immediate(|| wait_and_add(41)))
    │         ──────┬──────
    │               ╰────────
────╯

1 error
```

If an operation body fails *before* calling `resume`, the error returns to the caller as the operation's `fails` (`[dyn.suspend]`). If it fails or hits a defect *after* calling `resume`, the caller has already moved on, so it becomes a defect at the top level of the program or test. Calling `resume` after `main` or a test has ended does nothing. If a `suspend fn` body returns without ever calling `resume`, the waiting continuation unwinds as *interrupted* (running every `release` on the way out).

## Tasks and fibers

Concurrent work runs as *tasks* on *fibers*: lightweight units of execution scheduled cooperatively on a single thread (`[task.basic]`). There are no OS threads, no locks, and no parallel execution in the 0.20 series (`[task.parallel]`): tasks speed up a program when waits overlap (network requests, timers, I/O), not by splitting pure CPU work.

Because fibers switch cooperatively, switches happen only at *suspension points*:

- a call to a `suspend fn` operation that requires `Async` (or opening or releasing a `suspend handler`);
- a yield inserted by the runtime for fairness after a fixed count of calls that can suspend. Because the runtime counts calls rather than elapsed time, yields fall in the same places on every run.

Any stretch of code with no call that can suspend runs *atomically* with respect to other tasks:

- `st.set(st.get() + 1)` cannot be interleaved by another task.
- An `Immediate` function runs as one indivisible step; no fairness yields are inserted inside it.
- In `x = st.get(); helper(); st.set(x + 1)`, another task can run during `helper()` unless `helper`'s `R` is `Immediate`.

## `Task.Group`

Every task belongs to a handler in a `with` that bounds its lifetime (`[task.group]`):

```emel
import Std.Clock.{Clock, TestClock}
import Std.Console.{Console, SystemConsole}
import Std.Instant
import Std.State.{State}
import Std.Task

fn fetch(name: String, ms: Int) -> String uses { Clock, Console } {
  clock <- Clock
  console <- Console
  clock.sleep(ms)
  console.print_line("done #{name}")
  name
}

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  with SystemConsole {
    (pair, end) = State.run(start, || {
      with TestClock {
        with tasks = Task.Group {
          a = tasks.start(|| fetch("user", 100))
          b = tasks.start(|| fetch("orders", 40))
          (a.join(), b.join())
        }
      }
    })
    console <- Console
    console.print_line("pair: #{pair}, elapsed: #{Instant.to_unix_milliseconds(end)}ms")
  }
}
```

```console
$ emela run
done orders
done user
pair: ("user", "orders"), elapsed: 100ms
```

Both fetches overlap: `orders` finishes at 40ms, `user` finishes at 100ms, and the total virtual time is 100ms rather than 140ms.

- `tasks.start(f)` queues `f` and returns a `Task[A]` immediately without suspending; the new task begins running the next time the current task suspends. `a.join()` waits for `a` to finish and returns its value (later `join`s return the remembered value immediately).
- A task may start sibling tasks through its group's capability.
- When the body of `with tasks = Task.Group` returns a value, any tasks still running are interrupted, and the group's `release` waits until every task has unwound (including its own `release`s).
- If any task fails (with `fail` or a defect), its siblings are interrupted and the failure is delivered to the body at its next suspension point; the `with` ends with that failure after all tasks have unwound. If multiple tasks fail, the first becomes the result and the rest appear as notes.
- A group closes when its body ends or a task fails. Calling `start` on a closed group is a defect.
- `Task[A]` is declared `@external pub type Task[A] uses TaskGroup`, a *capability-bound type* (`[type.rules]`): it carries the group's capability in its type, so a `Task[A]` cannot leave the `with` of its group.
- A helper function that starts tasks in a caller's group takes a capability parameter with set type parameters `E` and `R`, listing the effects its own tasks need (`fn crawl[E, R](tasks: use TaskGroup[E, { Http, Async, R }], url: String) -> ()`). A group with larger `E` or `R` satisfies a requirement for smaller ones (`[task.group]`).
- The `E` and `R` of a `Task.Group` are inferred from the tasks started in it and fixed at the `with` of the group (`[task.group]`). Providing an effect *inside* the group does not reach its tasks:

```emel
import Std.Console.{Console, SystemConsole}
import Std.Task

fn main() {
  with tasks = Task.Group {
    with SystemConsole {
      tasks.start(|| {
        console <- Console
        console.print_line("hi")
      })
    }
    ()
  }
}
```

```console
$ emela check
error[E4301]: effect `Console` is not provided in `main`
    ╭─[ src/main.emel:7:7 ]
    │
  7 │ ╭─▶       tasks.start(|| {
    ┆ ┆
 10 │ ├─▶       })
    │ │
    │ ╰───────────────
────╯
   = note: tasks get their handlers from outside the `with` of their group, so `with` inside it does not reach them
   = help: provide it with `with` outside the group

1 error
```

Move `SystemConsole` outside `Task.Group` (or before `tasks = Task.Group` in the same `with`) so that the handler outlives every task in the group.

## Combinators

Most everyday code does not open `Task.Group` directly. `Std.Task` builds five combinators in plain Emela on top of `Task.Group` and `Deferred` (`[task.combinators]`):

| Function | Behavior |
|---|---|
| `Task.both(f, g)` | Runs two functions of possibly different types concurrently and returns `(A, B)` |
| `Task.all(fs)` | Runs a `List` of functions without parameters concurrently and collects their results in order |
| `Task.map(xs, f)` | Applies `f` to every element concurrently and collects results in order |
| `Task.race(fs)` | Returns the first result (or error) to finish and interrupts the rest |
| `Task.timeout(ms, f)` | Returns `Some(a)` if `f` finishes within `ms` milliseconds on `Clock`, otherwise interrupts `f` and returns `None` |

```emel
import Std.Clock.{Clock, TestClock}
import Std.Instant
import Std.State.{State}
import Std.Task

fn slow(label: String, ms: Int) -> String uses { Clock } {
  clock <- Clock
  clock.sleep(ms)
  label
}

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  (summary, _) = State.run(start, || {
    with TestClock {
      pair = Task.both(|| slow("a", 50), || slow("b", 30))
      mapped = Task.map([10, 30, 20], |ms| slow("#{ms}ms", ms))
      winner = Task.race([|| slow("slow", 100), || slow("fast", 20)])
      timed = Task.timeout(50, || slow("late", 200))
      (pair, mapped, winner, timed)
    }
  })
  dbg(summary)
  ()
}
```

```console
$ emela run
(("a", "b"), ["10ms", "30ms", "20ms"], "fast", None)
```

Because `Task.timeout` uses the `Clock` effect, under `TestClock` it runs in virtual time and finishes immediately.

## `Task.Background`, `Exit.attempt` and `Cause`

When you want to launch work whose results are not awaited, such as audit logging or background cleanup, use `Task.Background` (`[task.background]`):

- Its capability has type `use Spawn[E, R]`, and `bg.start(f)` takes a function returning `()`.
- Unlike `Task.Group`, a background never closes: `bg.start(f)` is allowed even while the `release` waits, and never produces a defect.
- When the body ends (normally or with an error), `Background` does *not* interrupt its tasks: its `release` waits until all of them finish, including new tasks started while it waits.
- If a background task fails, its siblings and the body keep running; instead, `on_failure: fn(Cause[E]) -> () uses { R }` is called in that task after it unwinds. `on_failure` must have an empty `E`, and a defect in `on_failure` is treated like a defect in a `release`.
- Only an interruption of the `Task.Background` `with` itself from outside interrupts its tasks (in which case `on_failure` is not called).

Failures inside tasks or at boundaries are represented by `Exit[A, E]` (`Success(A)` or `Failure(Cause[E])`) and the opaque type `Cause[E]` (`[task.cause]`):

- A cause has one of two kinds (`Cause.kind`): `Fail` (from `fail`, tracked in `E`) or `Die` (from a defect). Interruption is *not* a cause and is never received as a value.
- `Exit.attempt(f)` runs `f` in place (without starting a task) and captures both `Fail` and `Die` as `Failure(cause)`. It is the boundary where a defect can be observed as a value; it never catches an interruption of its own task, and during unwinding it does not catch `Die` either.
- Because Emela has no union value type for "one of `E`", you cannot extract the error value directly from `Cause[E]`. Inspect it with `Cause.kind` and `Cause.to_string`, or re-raise it with `Cause.raise(cause)` and handle concrete errors with `escape`.

```emel
import Std.Cause
import Std.Console.{Console, SystemConsole}
import Std.Exit
import Std.Exit.{Failure, Success}
import Std.Task

error Missing(key: String) derive Show

fn lookup(key: String) -> String fails Missing {
  if key == "ok" { "found" } else { fail Missing(key:) }
}

fn main() {
  with SystemConsole {
    console <- Console
    with bg = Task.Background(
      on_failure: |cause| console.print_line("bg failed: #{Cause.to_string(cause)}"),
    ) {
      bg.start(|| {
        _ = lookup("nope")
        ()
      })
      console.print_line("main body done")
    }

    Exit.attempt(|| lookup("missing")) match {
      Success(v) -> console.print_line("got #{v}")
      Failure(cause) -> {
        console.print_line("kind=#{Cause.kind(cause)} msg=#{Cause.to_string(cause)}")
        recovered = Cause.raise(cause) escape {
          Missing(key:) -> "fallback for #{key}"
        }
        console.print_line(recovered)
      }
    }

    Exit.attempt(|| panic("bug")) match {
      Success(_) -> ()
      Failure(cause) -> console.print_line(
        "kind=#{Cause.kind(cause)} msg=#{Cause.to_string(cause)}",
      )
    }
  }
}
```

```console
$ emela run
main body done
bg failed: error: Missing(key: "nope")
kind=Fail msg=error: Missing(key: "missing")
fallback for missing
kind=Die msg=defect: bug
```

Notice that `"main body done"` printed before the background task started: `bg.start` queued the task, and it ran when the `with bg = Task.Background` entered its `release` and waited.

## Interruption

There is no function that interrupts a single task by handle (`[task.interrupt]`). Interruption comes only from structure: the body of a `Task.Group` ending, a sibling in a `Task.Group` failing, or an enclosing `with` being interrupted. To stop a specific task on command, place it in its own `Task.Group` or pass it a `Deferred` stop signal.

- Interruption takes effect at the target task's next suspension point.
- From that point, the task unwinds, running every `release` from the inside out.
- A task is never interrupted while a `release` is running; any pending interruption takes effect once that `release` finishes.
- Neither `escape` nor `Exit.attempt` can swallow an interruption: even if unwinding passes through an `escape`, the interruption resumes at the next suspension point.

## `Deferred`, `Semaphore` and `Queue`

Three built-in handlers coordinate tasks, all scoped to `with` just like `Task.Group` (`[task.deferred]`, `[task.semaphore]`, `[task.queue]`):

- `Deferred[A]` is filled at most once: `complete(v)` stores `v` the first time (returning `True`, then `False`), `await()` suspends until filled (waking all waiters), and `poll()` checks without suspending.
- `Semaphore(permits: n)` bounds concurrency to `n` permits, served in the order tasks began waiting. Permits are handler members (`Permits(n:)` suspends until available; `TryPermits(n:)` takes permits only when `n` are free and no task is waiting, otherwise failing with `NoPermits`), so permits are automatically returned by `release` on normal exit, error, defect, or interruption. `Semaphore.with_permit` / `with_permits` and `try_with_permit` / `try_with_permits` wrap the `with` for convenience. `available()` counts free permits at that instant and is not atomic with a later acquisition.
- `Queue[A](capacity: c)` is a bounded FIFO channel: `offer(v)` waits while full, `take()` waits while empty, `try_offer` and `poll` do not suspend, and `close()` closes the queue so remaining items drain and further `offer`s (or `take`s once empty) fail with `QueueClosed`. Both waiting senders and waiting receivers are served in FIFO order, and an interrupted `offer` adds nothing.

```emel
import Std.Clock.{Clock, TestClock}
import Std.Console.{Console, SystemConsole}
import Std.Deferred.{Deferred}
import Std.Instant
import Std.Queue.{Queue, QueueClosed}
import Std.Semaphore
import Std.Semaphore.{Semaphore}
import Std.State.{State}
import Std.Task

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  with SystemConsole {
    console <- Console
    (_, _) = State.run(start, || {
      with TestClock {
        # Deferred: one-shot signal
        with ready = Deferred, tasks = Task.Group {
          _ = tasks.start(|| {
            clock <- Clock
            clock.sleep(10)
            _ = ready.complete(8080)
            ()
          })
          port = ready.await()
          console.print_line("listening on #{port}")
        }

        # Semaphore: bound concurrency
        with limit = Semaphore(permits: 2) {
          results = Task.map([1, 2, 3, 4], |id| {
            Semaphore.with_permit(limit, || {
              clock <- Clock
              clock.sleep(10)
              id * 10
            })
          })
          console.print_line("results: #{results}")
        }

        # Queue: bounded FIFO with close
        with jobs = Queue(capacity: 2), tasks = Task.Group {
          _ = tasks.start(|| {
            List.each(["a", "b", "c"], |item| jobs.offer(item))
            jobs.close()
          })
          fn drain() {
            item = jobs.take()
            console.print_line("job: #{item}")
            drain()
          }
          drain() escape {
            QueueClosed -> console.print_line("queue closed")
          }
        }
      }
    })
  }
}
```

```console
$ emela run
listening on 8080
results: [10, 20, 30, 40]
job: a
job: b
job: c
queue closed
```

The type arguments of `Deferred` and `Queue` are inferred from how their capabilities are used, or can be written explicitly (`Deferred[Int]`, `Queue[String](capacity: 2)`). Because `Deferred`, `Semaphore`, and `Queue` are capabilities of their `with`, any task that uses them must live inside a `Task.Group` or `Task.Background` nested inside their `with` or written after them in the same `with`, so they are never used after their scope ends.

## Summary

- Only `suspend fn` declarations mark suspension; functions and calls carry no mark. A handler receives a one-shot `resume` and decides whether to resume on the spot (Immediate) or later (`Async`).
- `R: Immediate` statically forbids suspension where it would be wrong, such as in host callbacks and `fn` operations.
- Tasks run cooperatively on single-threaded fibers and switch only at suspension points; stretches without suspending calls run atomically, and fairness yields are deterministic.
- Every task lives inside a `Task.Group` (interrupts leftovers on exit, cancels siblings on failure, closes on exit or failure, and keeps `Task[A]` inside its scope) or a `Task.Background` (waits for tasks to finish, never closes, and routes failures to `on_failure`).
- `Task.both`, `all`, `map`, `race`, and `timeout` cover common concurrency patterns in plain Emela.
- `Exit.attempt` turns `Fail` and `Die` into `Exit[A, E]` values with an opaque `Cause[E]`, without catching interruption.
- `Deferred`, `Semaphore`, and `Queue` are scoped handlers for one-shot handoff, concurrency limits, and bounded work queues.

Chapter 13 brings data, `opaque`, traits, effects, and capabilities together to show how to structure programs by composition.
