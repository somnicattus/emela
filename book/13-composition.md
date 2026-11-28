# 13. Designing with composition

Emela has no classes, no inheritance, and no run-time dynamic dispatch. Larger programs are built by composing three small mechanisms that Part I and Part II have already introduced: `opaque` types to guard invariants, traits to abstract over data types, and effects with handlers to abstract over behavior and resources.

This chapter compares when to use each tool and shows the main pattern for layering behavior: delegating to a capability held in a handler field.

## Hiding representation with `opaque`

Whenever a type carries a rule that its values must satisfy (an email address contains `@`, a money amount is not negative, a list stays sorted), make it `pub opaque` in its own module (chapter 7).

`src/money.emel`:

```emel
pub error NegativeAmount(cents: Int) derive Show

pub opaque type Money(cents: Int) derive Eq, Ord

pub fn from_cents(cents: Int) -> Money fails NegativeAmount {
  if cents < 0 { fail NegativeAmount(cents:) }
  Money(cents:)
}

pub fn add(a: Money, b: Money) -> Money {
  Money(cents: a.cents + b.cents)
}

impl Money: Show {
  fn show(self) {
    rest = self.cents % 100
    pad = if rest < 10 { "0" } else { "" }
    "$#{self.cents / 100}.#{pad}#{rest}"
  }
}
```

`src/main.emel`:

```emel
import Money

fn main() {
  price = Money.from_cents(1250)
  tax = Money.from_cents(125)
  total = price.add(tax)
  dbg("#{total}")
  ()
}
```

```console
$ emela run
"$13.75"
```

- Outside `Money`, values can be constructed only through `Money.from_cents` and `Money.add`, so a `Money` anywhere in the program is guaranteed to be non-negative.
- Callers cannot read `cents` or match on `Money(...)`, so the module can change its representation later (for example to `Int64`) without breaking callers.
- `Decode` and `ObjectKey` cannot be derived for an `opaque` type (chapter 9), because a derived decoder would construct values without running `from_cents`.
- Notice that `Money` writes `impl Money: Show` by hand instead of deriving `Show`: a derived `Show` would print `Money(cents: 1375)` and reveal the hidden field.

`opaque` solves data integrity and representation hiding. It does not choose between different implementations at run time.

## Traits as interfaces over data

Use a `trait` when an operation depends only on the type of a value and has one canonical meaning per type: equality (`Eq`), ordering (`Ord`), formatting (`Show`), serialization (`Encode`, `Decode`, `Shaped`), or domain conversions such as `FromText` in chapter 9.

- Coherence guarantees a single `impl Type: Trait` in the whole program (`[trait.impl]`). You never have to pass an implementation around or worry that two modules picked different orderings for the same type.
- Every call resolves to its `impl` at compile time, with no run-time indirection.
- Trait functions are pure by default (`[trait.effects]`). They do not hold open connections, read the clock, or swap behavior between production and tests.

If a single type seems to need two different `impl`s of the same trait (one for production and one for tests, or one writing to a file and one to memory), the interface belongs in an `effect`, not a `trait`.

## Effects as interfaces over behavior

Use an `effect` when an interface describes what a component does to the world and callers should remain independent of how it is done:

```emel
import Std.Console.{Console, SystemConsole}
import Std.Map
import Std.Map.{Map}
import Std.State.{State}

error DuplicateUser(id: String) derive Show

type Account(id: String, name: String)

effect Accounts {
  fn find(id: String) -> Option[Account]
  fn save(account: Account) -> ()
}

effect Notifier {
  fn send(to: String, message: String) -> ()
}

fn sign_up(id: String, name: String) -> Account fails DuplicateUser uses { Accounts, Notifier } {
  accounts <- Accounts
  notifier <- Notifier
  accounts.find(id) match {
    Some(_) -> fail DuplicateUser(id:)
    None -> {
      account = Account(id:, name:)
      accounts.save(account)
      notifier.send(id, "Welcome, #{name}!")
      account
    }
  }
}

handler InMemoryAccounts: Accounts {
  fn find(id) {
    st <- State[Map[String, Account]]
    Map.get(st.get(), id)
  }

  fn save(account) {
    st <- State[Map[String, Account]]
    st.update(|m| Map.insert(m, account.id, account))
  }
}

handler ConsoleNotifier: Notifier {
  fn send(to, message) {
    console <- Console
    console.print_line("notify #{to}: #{message}")
  }
}

layer App {
  InMemoryAccounts
  ConsoleNotifier
  SystemConsole
}

fn main() {
  (account, _) = State.run(Map.empty(), || {
    with App {
      sign_up("u1", "Ada")
    }
  })
  dbg(account)
  ()
}
```

```console
$ emela run
notify u1: Welcome, Ada!
Account(id: "u1", name: "Ada")
```

- `sign_up` does not know whether `Accounts` talks to a SQL database or an in-memory `Map`, or whether `Notifier` sends email or prints to the console.
- `sign_up` takes no handler arguments or configuration records; its signature lists only the effects it needs (`uses { Accounts, Notifier }`), and wiring happens at the boundary (`layer App`, or a test `layer` under `test/`).
- Each handler's own requirements (`State[Map[String, Account]]`, `Console`) stay hidden from `sign_up` and surface only where the handler is placed in a `with`.

Keeping each effect focused on one role (`Accounts` separate from `Notifier`) lets every function's `uses` list show precisely what it can touch, and lets tests replace one handler without touching the rest.

## Delegating to a capability held in a field

Sometimes a handler wraps another implementation of the *same* effect: adding caching, counting reads, logging calls, or retrying requests.

If a `handler CountingStore: Store` wrote `store <- Store` inside its own `get`, it would require an outer `Store` provided by name. You can instead make delegation explicit and composable by storing a capability (`use Store`) in a field (`[effect.use]`, `[effect.handler]`):

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

type Reads(count: Int)

handler CountingStore(inner: use Store): Store {
  init(inner: use Store) {
    Self(inner:)
  }

  fn get(key) {
    reads <- State[Reads]
    reads.update(|r| Reads(count: r.count + 1))
    self.inner.get(key)
  }

  fn put(key, value) {
    self.inner.put(key, value)
  }
}

fn load_pair(a: String, b: String) -> (Option[String], Option[String]) uses { Store } {
  store <- Store
  (store.get(a), store.get(b))
}

fn main() {
  ((result, reads), _) = State.run(Map.empty(), || {
    State.run(Reads(count: 0), || {
      with raw = MemoryStore {
        with CountingStore(inner: raw) {
          store <- Store
          store.put("x", "1")
          load_pair("x", "y")
        }
      }
    })
  })
  dbg(result)
  dbg(reads)
  ()
}
```

```console
$ emela run
(Some("1"), None)
Reads(count: 2)
```

- Calling `self.inner.get(key)` does *not* add `Store` to `requires(CountingStore)`. Because `init`'s arguments are evaluated outside `with CountingStore(inner: raw)`, the compiler already knows that `raw` belongs to an enclosing `with` that outlives `CountingStore` (`[effect.use]`, `[effect.handler]`).
- `raw` is a named item, so it provides nothing by name; `CountingStore` is the only unnamed provider of `Store` inside its body, so `load_pair` is guaranteed to go through the counter. Several wrappers can be stacked (`with a = MemoryStore { with b = CachingStore(inner: a) { with CountingStore(inner: b) { ... } } }`) without labels or ambiguity.
- Suspension is still tracked: if `self.inner` has a `suspend fn` operation, calling it still counts for `Immediate` inside operation bodies, and calling it in `release` adds `Async` to `requires(H)`.

When a handler instead owns a sub-resource for its lifetime (like `RepoHandler` leasing a connection in chapter 11), `init` opens the handler member (`Pool.Lease(...)`) and stores the capability in a field; again, calling operations on `self.conn` adds no extra requirement and the sub-resource is released automatically after the wrapper's `release`.

## Choosing the right tool

| Need | Tool |
|---|---|
| Protect a value's invariants or hide its fields | `pub opaque type` / `pub opaque enum` |
| One canonical operation per data type, checked statically | `trait` and `impl` |
| Swappable behavior, I/O, or test handlers | `effect` and `handler`, wired with `layer` and `with` |
| Two roles of the same effect wired by name | Label (`effect L of E`) |
| Two instances passed explicitly, or wrapping one handler around another | Named `with` item + capability parameter or field (`use E`) |
| Scoped resource acquired and released deterministically | Handler with `init` and `release`, or handler member |

## Summary

- `opaque` types enforce invariants at module boundaries and keep representations free to change.
- Traits abstract over data types with one globally coherent, statically resolved `impl` per `(Type, Trait)`.
- Effects and handlers abstract over behavior and environment; `layer` wires them together at the edge of the program, leaving business logic dependent only on the effects it names.
- A handler can wrap another handler of the same effect by taking `inner: use E` in `init` and storing it in a field, without adding `E` to its requirements and with its lifetime verified by `with`.

Chapter 14 steps back and surveys the whole type and effect system: the exact set of bugs that Emela's types rule out before a program runs.
