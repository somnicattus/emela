# 11. The lifetime of resources

Memory in Emela is managed automatically, and when a value's memory is freed is unspecified (`[dyn.memory]`). External resources such as database connections, open files, network sockets, and semaphore permits cannot wait for memory to be reclaimed: they must be acquired at a clear point and released promptly, even when something fails.

Emela ties resource lifetimes to `with`, not to values. There are no destructors attached to data types. Instead, a resource is a handler whose `init` acquires it on entering `with` and whose `release` cleans it up on leaving `with` (`[effect.handler]`, `[dyn.with]`).

## `init` and `release`

Every handler can declare an `init` block and a `release` block:

```emel
import Std.Console.{Console, SystemConsole}

error Broken

effect Session {
  fn query(sql: String) -> String fails Broken
}

handler FakeSession(name: String): Session {
  init(name: String) {
    console <- Console
    console.print_line("init #{name}")
    Self(name:)
  }

  release {
    console <- Console
    console.print_line("release #{self.name}")
  }

  fn query(sql) {
    console <- Console
    console.print_line("#{self.name}: #{sql}")
    if sql == "BAD" { fail Broken }
    "ok"
  }
}

fn main() {
  with SystemConsole {
    res = with FakeSession(name: "s1") {
      s <- Session
      s.query("SELECT 1")
    }
    console <- Console
    console.print_line("result: #{res}")

    recovered = with FakeSession(name: "s2") {
      s <- Session
      s.query("BAD")
    } escape {
      Broken -> "recovered"
    }
    console.print_line("result: #{recovered}")
  }
}
```

```console
$ emela run
init s1
s1: SELECT 1
release s1
result: ok
init s2
s2: BAD
release s2
result: recovered
```

In the second block, `s.query("BAD")` fails with `Broken`, and `FakeSession`'s `release` runs *before* `escape` outside the `with` catches the error. The pairing between `init` and `release` is structural:

- A handler value can be constructed only inside its own `init`, so there is no way to obtain a handler without entering a `with`.
- A capability cannot leave the `with` that created it (chapter 10), so no code can touch the resource after `release` has run.
- `release` runs once on leaving the `with`, however the body ends: normal value, `fail`, defect (`panic` or failed `assert`), or task interruption (chapter 12).
- A handler implementing several effects (`handler H: A + B`) runs its `init` once and its `release` once.

### `release` cannot fail

The error set `E` of `release` must be empty (`[effect.handler]`). Leaving a `with` may already be unwinding an error or defect, so cleanup cannot introduce a typed error of its own:

```emel
error CloseFailed

effect File {
  fn read() -> String
}

fn close_raw() -> () fails CloseFailed {
  fail CloseFailed
}

handler RawFile: File {
  release {
    close_raw()
  }

  fn read() {
    "data"
  }
}
```

```console
$ emela check
error[E4007]: `release` of handler `RawFile` cannot fail, but this may fail with error `CloseFailed`
    ╭─[ src/main.emel:13:5 ]
    │
 13 │     close_raw()
    │     ─────┬─────
    │          ╰───────
────╯
   = help: handle it with `escape`, or turn it into a defect with `panic`

1 error
```

If cleanup calls something that can fail, handle the error inside `release` with `escape` (for example, logging it and continuing), or turn it into a defect with `panic`. The return value of `release` is discarded, so its type does not matter. Unlike a `fn` operation, `release` is allowed to call `suspend fn` operations (chapter 12), and it is not interrupted while it runs.

## Order of initialization and release

A `with` with several items (`with H1, ..., Hn { body }`) runs in four steps (`[dyn.with]`):

1. Evaluate the arguments of `H(arguments)` from left to right outside the `with`, before any `init` runs. Effects provided by items of this `with` are not in scope for the arguments. If an unnamed item is overridden by a later item for the same effect, its arguments are still type-checked but are not evaluated, and its `init` never runs (`[effect.with]`).
2. Sort the items by their requirements and run each `init` once. Any unnamed handler whose effect satisfies `requires(H)` of another item in the same `with` initializes first, keeping the written order where requirements do not constrain it (named items provide nothing by name, so they do not satisfy requirements). A handler member item (`x = E.M(...)` or `x = c.M(...)`) is ordered only by `E` or `c`, not by the inner handler's requirements.
3. Evaluate the body.
4. Run the `release`s in the reverse order of initialization, then return the outcome of the body (value, error, defect, or interruption).

Because `release` runs in reverse initialization order, a handler's `release` can safely use any effect from `requires(H)`: the handlers that provide those effects are still active and will be released only afterward.

```emel
import Std.Console.{Console, SystemConsole}

error ConnectFailed derive Show

effect Config {
  fn dsn() -> String
}

effect Db {
  fn ping() -> String
}

handler EnvConfig: Config {
  init {
    console <- Console
    console.print_line("init Config")
    Self
  }

  release {
    console <- Console
    console.print_line("release Config")
  }

  fn dsn() {
    "db://local"
  }
}

handler LiveDb(url: String): Db {
  init(should_fail: Bool) {
    cfg <- Config
    console <- Console
    console.print_line("init Db (#{cfg.dsn()})")
    if should_fail { fail ConnectFailed }
    Self(url: cfg.dsn())
  }

  release {
    console <- Console
    console.print_line("release Db (#{self.url})")
  }

  fn ping() {
    "pong from #{self.url}"
  }
}

fn main() {
  with SystemConsole {
    console <- Console
    console.print_line("--- success ---")
    with LiveDb(should_fail: False), EnvConfig {
      db <- Db
      console.print_line(db.ping())
    }
    console.print_line("--- init fails ---")
    with LiveDb(should_fail: True), EnvConfig {
      db <- Db
      console.print_line(db.ping())
    } escape {
      ConnectFailed -> console.print_line("caught ConnectFailed")
    }
  }
}
```

```console
$ emela run
--- success ---
init Config
init Db (db://local)
pong from db://local
release Db (db://local)
release Config
--- init fails ---
init Config
init Db (db://local)
release Config
caught ConnectFailed
```

Even though `LiveDb` is written before `EnvConfig`, `LiveDb` requires `Config`, so `EnvConfig` initializes first and releases last.

## Failures during `init` and `release`

A `release` runs if and only if its `init` returned a value (`[dyn.with]`):

- If evaluating an argument of `H(args)` fails, no `init` has run yet, so nothing is released and the `with` ends with that error.
- If an `init` fails (as `LiveDb(should_fail: True)` did above), the body is skipped, `LiveDb`'s `release` does *not* run because its `init` returned no value, and every handler that *did* finish `init` (`EnvConfig`) is released in reverse order.

Defects inside `release` follow a fixed precedence so that no cleanup is skipped (`[dyn.defect]`):

- If the body ended with a normal value, an error, or an interruption, a defect in `release` becomes the result of the `with`.
- If the body already ended with a defect, that first defect stays the result, and each defect from a `release` is appended as a note (`note: the release of handler <name> also failed: <message>`).
- In every case, all remaining handlers still run their `release`.

## Opening resources inside `init`

A handler's `init` often needs to acquire a sub-resource from another effect's handler member, such as a repository handler leasing a connection from a `Pool`. Inside `init`, a handler member can be called directly as an expression (`Pool.Lease(...)` or `pool.Lease(...)`), returning the capability of the opened handler so it can be stored in a field (`[effect.handler]`, `[dyn.with]`):

```emel
import Std.Console.{Console, SystemConsole}

effect Pool {
  handler Lease(id: Int): Conn
}

effect Conn {
  fn exec(cmd: String) -> ()
}

effect Repo {
  fn save(item: String) -> ()
}

handler PrintPool: Pool {
  handler Lease(id: Int): Conn {
    init(id: Int) {
      console <- Console
      console.print_line("lease ##{id} acquired")
      Self(id:)
    }

    release {
      console <- Console
      console.print_line("lease ##{self.id} returned")
    }

    fn exec(cmd) {
      console <- Console
      console.print_line("conn ##{self.id}: #{cmd}")
    }
  }
}

handler RepoHandler(conn: use Conn): Repo {
  init {
    conn = Pool.Lease(id: 1)
    console <- Console
    console.print_line("repo ready")
    Self(conn:)
  }

  release {
    console <- Console
    console.print_line("repo closing")
  }

  fn save(item) {
    self.conn.exec("INSERT #{item}")
  }
}

fn main() {
  with SystemConsole, PrintPool {
    with RepoHandler {
      repo <- Repo
      repo.save("book")
    }
  }
}
```

```console
$ emela run
lease #1 acquired
repo ready
conn #1: INSERT book
repo closing
lease #1 returned
```

The lifetime of `Lease` is tied to `RepoHandler`:

- `RepoHandler`'s `release` runs *first* (while `self.conn` is still alive), and then every handler opened by its `init` is released in reverse order of opening.
- Calling `self.conn.exec(...)` inside `RepoHandler` does not add `Conn` to `requires(RepoHandler)`; `RepoHandler` requires only `Pool` (and `Console`), because `conn` lives as long as `RepoHandler` does (`[effect.use]`).
- If `RepoHandler`'s `init` fails partway through after opening some sub-handlers, those sub-handlers are released in reverse order before the error propagates.
- A sub-handler opened through an inner `with` inside `init` cannot outlive that inner `with` and cannot be stored in `Self(...)`; it is released when that inner `with` ends.

## When `release` does not run

`release` runs on every normal exit, typed error, defect, and interruption. The specification lists the only three situations where a `release` may not run or may be cut off (`[dyn.with]`):

- When the program cannot obtain the memory it needs, whether releases run is implementation-defined (`[dyn.memory]`).
- When the event loop becomes empty (no pending host work and no fiber able to make progress), suspended releases are cut off, except the release of a `Task.Group` or `Task.Background`, which waits for its interrupted tasks to unwind (`[dyn.suspend]`).
- When the process itself ends abruptly (via an OS signal such as Ctrl-C, an external function calling `abort`, or calling an `@export` function while a launch is already running on the same thread), no further Emela code runs.

In all other cases, entering a `with` and leaving it are strictly balanced.

## Summary

- Resources are managed by handlers (`init` and `release`) scoped to `with`, not by destructors on values.
- `release` runs once for every handler whose `init` returned a value, whether the body returns normally, fails, dies with a defect, or is interrupted.
- `release` must have an empty error set `E`, may call suspending operations, and is not interrupted while running.
- Within one `with`, arguments evaluate left to right outside the `with`, handlers initialize in requirement order, and releases run in reverse initialization order.
- A handler's `init` can open handler members directly (`Pool.Lease(...)`) and store their capabilities in fields; those sub-handlers are released in reverse order *after* the owning handler's `release`.
- Defects in `release` never skip remaining releases; when the body already died with a defect, release defects are recorded as notes.

Chapter 12 builds on effects, handlers, and `release` to introduce suspension and structured concurrency.
