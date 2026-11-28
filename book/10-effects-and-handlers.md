# 10. Defining effects and handlers

Chapter 6 used the effects and handlers of the standard library. This chapter shows how to declare your own: `effect` defines an interface of operations, `handler` implements them, `layer` bundles handlers together, labels distinguish multiple uses of the same effect, and capability values (`use E`) let you pass a specific handler explicitly without letting it outlive its `with` (`[effect]`).

## Declaring an effect and a handler

An `effect` lists the operations a caller can invoke (`[effect.decl]`), and a `handler` implements every operation of one or more effects (`[effect.handler]`):

```emel
import Std.Console.{Console, SystemConsole}

effect Logger {
  fn info(msg: String) -> ()
}

handler PrefixLogger(prefix: String): Logger {
  init(prefix: String) {
    Self(prefix:)
  }

  fn info(msg) {
    console <- Console
    console.print_line("[#{self.prefix}] #{msg}")
  }
}

fn work(name: String) -> () uses { Logger } {
  log <- Logger
  log.info("started #{name}")
  log.info("finished #{name}")
}

fn main() {
  with SystemConsole, PrefixLogger(prefix: "app") {
    work("sync")
  }
}
```

```console
$ emela run
[app] started sync
[app] finished sync
```

- `PrefixLogger(prefix: String)` declares a field `prefix`. Any handler with fields must declare an `init` block, which runs once when entering `with` and constructs the handler value with `Self(...)` (or `PrefixLogger(...)`). A handler without fields needs no `init`, though it may write one (returning `Self` or its name) when it has setup work. A handler value can be constructed only inside its own `init`, so `init` and `release` can never be skipped.
- `init(prefix: String)` takes parameters whose types are written explicitly; callers supply them at the `with` as `PrefixLogger(prefix: "app")`. Arguments are evaluated outside the `with`, before any `init` runs (`[effect.with]`).
- Parameter and return types of operations come from `effect Logger`, so the handler leaves them out. Inside an operation body or `release`, `self` refers to the handler value and gives read access to its fields (`self.prefix`). `self` is also a capability of the handler and can be passed to a parameter of type `use Logger`.
- `PrefixLogger` uses `Console` inside `info`. An operation cannot write `uses`, and a handler header lists no requirements: the union of the effects used in `init`, `release`, and all operation bodies is the handler's *requirements* (`requires(H)`), which must be provided where the handler is placed in a `with`. Because requirements stay on the handler, replacing `PrefixLogger` with another handler never changes the `uses` of `work`.
- An operation in an `effect` may declare `fails E`. Its implementations may raise only errors within `E`, so replacing the handler never changes the caller's `E` either.
- A handler may implement several effects sharing one `self` (`handler SystemTx(conn: Connection): Query + Execute`), as long as their operation names do not clash. Both effects and handlers may take ordinary type parameters (`effect State[S]`, `handler Memory[S](value: S): State[S]`). Every type parameter of a handler must appear in its effects, in a field's type, or in an `init` parameter's type, and its type arguments are determined once at each `with`, `layer`, or `by` site (`[effect.with]`).

An effect declared in source must have at least one operation or handler member. Only built-in host effects (`Async`, `ProcessHost`, `PageHost`) and `External` have none.

## State and stateful handlers

Values in Emela never change, and handler fields are immutable after `init`. When a handler needs mutable state, such as an in-memory store or a test handler that records calls, it uses the standard library's `State[S]` effect (`Std.State`, `[effect.state]`):

```emel
import Std.Map
import Std.Map.{Map}
import Std.State.{State}

effect Store {
  fn get(key: String) -> Option[String]
  fn put(key: String, value: String) -> ()
}

handler MemoryStore: Store {
  fn get(key) {
    st <- State[Map[String, String]]
    Map.get(st.get(), key)
  }

  fn put(key, value) {
    st <- State[Map[String, String]]
    st.update(|m| Map.insert(m, key, value))
  }
}

fn register(id: String, name: String) -> Option[String] uses { Store } {
  store <- Store
  store.put(id, name)
  store.get(id)
}

fn main() {
  (found, final_map) = State.run(Map.empty(), || {
    with MemoryStore {
      register("u1", "Ada")
    }
  })
  dbg(found)
  dbg(Map.to_list(final_map))
  ()
}
```

```console
$ emela run
Some("Ada")
[("u1", "Ada")]
```

- `State[S]` provides `get() -> S`, `set(value: S) -> ()`, and `update(f: fn(S) -> S) -> ()`. Because `f` in `update` is pure, `update` runs atomically even when tasks share the state (chapter 12).
- `State.run(initial, body)` provides `State[S]` to `body` and returns `(result, final_state)`, removing `State[S]` from `R`. If `body` fails with an error, the state is discarded and only the error propagates; to keep the state on failure, `escape` inside `body`.
- Effects with different type arguments are distinct in `R`: `State[Int]` and `State[String]` do not collide. To keep two states of the same underlying type apart, wrap them in distinct single-field types (`type Hits(n: Int)`, `type Misses(n: Int)`) or use labels (below).

## Bundling handlers with `layer`

Real applications wire many handlers together, and handlers often depend on each other's effects. A `layer` names a bundle of handlers and other layers (`[effect.layer]`):

```emel
import Std.Console.{Console, SystemConsole}

effect Config {
  fn app_name() -> String
}

effect Logger {
  fn info(msg: String) -> ()
}

handler StaticConfig(name: String): Config {
  init(name: String) {
    Self(name:)
  }

  fn app_name() {
    self.name
  }
}

handler AppLogger(prefix: String): Logger {
  init {
    cfg <- Config
    Self(prefix: cfg.app_name())
  }

  fn info(msg) {
    console <- Console
    console.print_line("[#{self.prefix}] #{msg}")
  }
}

layer AppLive {
  AppLogger
  StaticConfig(name: "billing")
  SystemConsole
}

fn main() {
  with AppLive {
    log <- Logger
    log.info("ready")
  }
  with AppLive, StaticConfig(name: "staging") {
    log <- Logger
    log.info("ready")
  }
}
```

```console
$ emela run
[billing] ready
[staging] ready
```

- The order of items in a `layer` has no meaning. `AppLive` lists `AppLogger` before `StaticConfig`, even though `AppLogger`'s `init` uses `Config`. The compiler orders initialization from `requires(H)` and runs `release` in the reverse order (`[dyn.with]`, chapter 11).
- An item in a `layer` may pass arguments (`StaticConfig(name: "billing")`), but those arguments must be pure expressions (empty `E` and `R`) because layer items have no evaluation order. Pass effectful arguments at the `with` site instead.
- Two items in a `layer` providing the same effect do not compile, and neither do cycles among layers or among handler requirements.
- A layer may leave some requirements unsatisfied to be provided by an outer `with`. Listing a handler after a layer in a `with` (`with AppLive, StaticConfig(name: "staging")`) replaces only that handler; the overridden handler is not even initialized (`[effect.with]`).

## Named `with` items and capability types `use E`

An unnamed item in a `with` provides its effect *by name*, so any code inside the block can obtain it through an effect alias (`log <- Logger`). When you need two instances of the same effect at once, or want to hand a specific handler to a function, bind the item with a name (`[effect.with]`, `[effect.use]`):

```emel
import Std.Console.{Console, SystemConsole}

effect Logger {
  fn info(msg: String) -> ()
}

handler PrefixLogger(prefix: String): Logger {
  init(prefix: String) {
    Self(prefix:)
  }

  fn info(msg) {
    console <- Console
    console.print_line("[#{self.prefix}] #{msg}")
  }
}

fn mirror(primary: use Logger, secondary: use Logger, msg: String) -> () {
  primary.info(msg)
  secondary.info(msg)
}

fn main() {
  with SystemConsole {
    with app = PrefixLogger(prefix: "app"), audit = PrefixLogger(prefix: "audit") {
      mirror(app, audit, "user signed in")
    }
  }
}
```

```console
$ emela run
[app] user signed in
[audit] user signed in
```

- `app` and `audit` are *capabilities* (or *capability values*): values representing an initialized handler, with type `use Logger`. Operations are called on them directly (`primary.info(msg)`).
- A named item provides nothing by name: `with app = PrefixLogger(...)` makes `app` available through that binding, and does not provide `Logger` to `log <- Logger` or to requirements of other items in the same `with`. A named item is never overridden by a later item either.
- A `layer` cannot be bound as a named item (`with x = AppLive` does not compile), and neither can a label item that makes no new handler (below).
- `log <- Logger` is an *effect alias*, not a value: it can appear only as `log.op(...)` or `log.op`, and cannot be passed to a parameter of type `use Logger`.
- `use E` is a type and can appear in parameters, fields, tuples, lists, and return types. A handler implementing several effects produces a value of type `use E1 + E2`, which can be passed where `use E1` is expected (narrowing to only the operations of `E1`). A generic function that accepts any capability writes `[C: Capability]` (`[trait.builtin]`). `use E` satisfies no other trait: two capabilities cannot be compared with `==`.

## Handler members

Some effects act as a factory that opens scoped resources on demand: a database pool opening connections, a file system opening files, or a semaphore granting permits. An effect can declare a *handler member* for this (`[effect.decl]`, `[effect.handler]`):

```emel
import Std.Console.{Console, SystemConsole}

effect Vault {
  handler Open(scope: String): Bucket
}

effect Bucket {
  fn write(entry: String) -> ()
}

handler ConsoleVault: Vault {
  handler Open(scope: String): Bucket {
    init(scope: String) {
      console <- Console
      console.print_line("open #{scope}")
      Self(scope:)
    }

    release {
      console <- Console
      console.print_line("close #{self.scope}")
    }

    fn write(entry) {
      console <- Console
      console.print_line("#{self.scope}: #{entry}")
    }
  }
}

fn main() {
  with SystemConsole, ConsoleVault {
    with bucket = Vault.Open(scope: "orders") {
      bucket.write("created #1")
      bucket.write("paid #1")
    }
  }
}
```

```console
$ emela run
open orders
orders: created #1
orders: paid #1
close orders
```

- Every handler of `Vault` must implement `Open` as an inner `handler` with the same `init` parameters and `fails` clause. An inner handler cannot be named outside its outer handler, cannot refer to the `self` of its outer handler, and adds its own requirements to those of the outer handler (`ConsoleVault`).
- A handler member is opened only as a named `with` item, either through the effect name (`with bucket = Vault.Open(...)`) or through a capability (`with bucket = vault.Open(...)`), or inside another handler's `init` (chapter 11). Its effect is used only through the bound capability, and its `release` runs when that `with` ends.
- A handler member that may suspend when opened or released is declared `suspend handler` (as `Semaphore.Permits` is in chapter 12). Without `suspend`, its `init` and `release` cannot suspend.

## Labels

Sometimes a program uses the same effect for two distinct roles, such as an audit log and a debug log that both use `Console`, without threading capability parameters through every function. A *label* declares a separate effect that shares another effect's operations (`[effect.label]`):

```emel
import Std.Console.{Console, SystemConsole}

effect AuditLog of Console
effect DebugLog of Console default

handler PrefixConsole(tag: String): Console {
  init(tag: String) {
    Self(tag:)
  }

  fn print_line(text) {
    console <- Console
    console.print_line("[#{self.tag}] #{text}")
  }

  fn print_error_line(text) {
    console <- Console
    console.print_error_line("[#{self.tag}] #{text}")
  }
}

fn emit(msg: String) -> () uses { Console } {
  console <- Console
  console.print_line(msg)
}

fn process() -> () uses { AuditLog, DebugLog } {
  with AuditLog { emit("order accepted") }
  with DebugLog { emit("cache hit") }
}

fn main() {
  with SystemConsole, AuditLog by PrefixConsole(tag: "AUDIT") {
    process()
  }
}
```

```console
$ emela run
[AUDIT] order accepted
cache hit
```

- `effect AuditLog of Console` is a separate effect in `R` with the same operations and handler members as `Console`. You can call its operations directly through an effect alias (`audit <- AuditLog`), or serve `Console` from it with `with AuditLog { ... }` so ordinary functions like `emit` write to `AuditLog`.
- Its underlying effect must be a user effect (not another label or a built-in effect such as `Async` or `TaskGroup`). You cannot write `handler H: AuditLog`; instead you plug in any handler of `Console` with `AuditLog by H`, or merge it back into `Console` with `AuditLog by Console`.
- `PrefixConsole` implements `Console` and also uses `Console` inside `print_line`. A handler's requirement on its own effect is satisfied by the outer `Console` (here `SystemConsole` in the same `with`), not by itself, so it does not call itself (`[effect.with]`).
- `effect DebugLog of Console default` is provided automatically to the body of any unnamed item that provides `Console` (here `SystemConsole`), unless overridden by an explicit `DebugLog by P`. A `default` label does not satisfy requirements of other handlers in the same `with`, and a named item does not provide it. A label without `default` must be provided explicitly, so an audit log can never silently fall back to standard output.
- Labels also work on parameterized effects with concrete type arguments, such as `effect Hits of State[Int]`.

## Why capabilities cannot leak

A capability is valid only while the `with` that initialized its handler is active; once that `with` ends, the handler's `release` has run and its resource is gone. The type system therefore forbids any capability (directly, inside a data structure, inside an error, or captured in a closure) from leaving the `with` that made it (`[effect.escape]`):

```emel
import Std.Console.{Console}

effect Logger {
  fn info(msg: String) -> ()
}

handler ConsoleLogger: Logger {
  fn info(msg) {
    console <- Console
    console.print_line(msg)
  }
}

fn make_greeter() {
  with log = ConsoleLogger {
    || log.info("hi")
  }
}
```

```console
$ emela check
error[E4405]: capability `log` leaves the `with` that made it
    ╭─[ src/main.emel:16:5 ]
    │
 15 │   with log = ConsoleLogger {
    │              ──────┬──────
    │                    ╰──────── made here; it ends with this `with`
 16 │     || log.info("hi")
    │     ────────┬────────
    │             ╰──────────
────╯

1 error
```

The closure `|| log.info("hi")` captures `log`, so its `R` contains the capability `log`. Returning it from `with log = ConsoleLogger` would let someone call `log.info("hi")` after `ConsoleLogger` has already ended. Passing that closure to `List.each` *inside* the `with` is fine, because it never leaves the scope.

An effect alias (`log <- Logger`) behaves differently: it is not a capability of any particular `with`. A closure that calls `log.info("hi")` has `Logger` (the effect name) in its `R`, so it may be returned freely and will use whatever `Logger` handler is active where it is eventually called.

## Summary

- `effect` declares operations (optionally with `fails`, never `uses`) and optional handler members. `handler Name(fields): Effect` implements them; its requirements are inferred from `init`, `release`, and its operation bodies.
- Mutable state lives in `State[S]`, run with `State.run(initial, body)`. Stateful and test handlers require `State[S]`.
- `layer` bundles handlers and other layers. Initialization order follows handler requirements, and a later item in a `with` overrides a handler from a layer.
- A named `with` item (`with x = H`) binds a capability value of type `use E` and does not provide `E` by name. Capabilities can be passed to functions, narrowed (`use E1 + E2` to `use E1`), and opened via handler members (`x = E.M(...)`).
- Labels (`effect L of E` and `effect L of E default`) distinguish roles of the same effect and are wired with `L` and `L by P`.
- A capability cannot escape the `with` that created it, whether directly, in a data structure, in an error, or captured by a closure.

Chapter 11 examines the full lifecycle of handlers: how `init` and `release` guarantee deterministic resource cleanup in every outcome.
