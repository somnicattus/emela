---
title: The Host effect, which keeps external functions inside handlers
authors: [somnicattus]
status: draft
pr: 1
issue:
decided:
touches: [effect.host (new), ext.call, effect.handler, diag.list, glossary.terms]
requires: []
related: []
---

# The Host effect, which keeps external functions inside handlers

## Summary

An `@external` fn with `uses` omitted can be used only inside a handler, and `[ext.call]` says that it cannot be carried out of a handler as a function value either. Nothing in the specification enforces that second rule: a closure that calls such a function, or the function value itself, can be returned from an operation and called from `main`. This proposal adds a built-in effect, `Host`, that is the R of these external functions and that is removed only from the R of the bodies of a handler. A value that carries a host side effect then carries `Host` in its type, and calling it outside a handler is a compile error by the existing rules on R.

```emel
effect Leak {
  fn get() -> fn(String) -> () fails {} uses {}
}

handler Leaker: Leak {
  fn get() { |s| host_log(s) } # E0403: expected `uses {}`, found `uses { Host }`
}
```

## Motivation

`[ext.call]` has three rules for an external function with `uses` omitted:

1. The restriction on where it can be called applies to references, not only to calls.
2. It cannot be carried out of a handler as a function value either.
3. The inside of an anonymous function inside an operation body of a handler counts as inside.

Rules 1 and 3 are rules on where the name is written, and the implementation checks exactly them (E0407). Rule 2 is a rule on where a value flows, and no section says how it is checked. Rule 2 was presumably meant to follow from rule 1, but it does not: a reference written inside an operation body can be returned from the operation. The R of these functions is empty (`[ext.call]`: "The host side effects of an external function are not put in R"), so the type of the value is `fn(String) -> () uses {}`, a function that looks pure.

This program compiles with 0.27.1 (`8b41f27`), and running it on the JS target performs the host side effect twice from `main`, outside any handler:

```emel
@external(js, "./log.mjs", "hostLog")
fn host_log(s: String) -> ()

effect Leak {
  fn get() -> fn(String) -> () fails {} uses {}
}

handler Leaker: Leak {
  fn get() { |s| host_log(s) }
}

fn main() {
  f = with Leaker { (use Leak).get() }
  f("called from main, outside any handler")
}
```

Returning `host_log` itself (`fn get() { host_log }`) behaves the same. `emela check` reports nothing for either.

This breaks the promise at the opening of Chapter 12, that every side effect goes through a named effect:

- A function whose type says `uses {}` performs a host side effect. A test handler cannot replace it, and code that relies on its purity is wrong.
- `[module.pome]` relies on rule 2: an `@external` fn in a dependency cannot be pub (E0237), and "an external function that omits use can only be used inside a handler in the first place (12.2), so the only path to the host visible to dependents is effects" (`spec/history.md`). A dependency can hand out such a closure through any operation, so the host is reachable without an effect.

The workaround in the current specification is to not write such code. The compiler gives no help.

## Proposal

### Syntax

No new syntax. `Host` is a new name, a built-in effect, written in uses clauses like `Async`.

```emel
fn next_line(step: fn() -> Int uses { Async, Host }, text: fn() -> String) -> Option[String]
  fails InvalidInput | StdinFailure
  uses { Async, Host } {
  step() match {
    0 -> Some(text())
    _ -> next_line(step, text)
  }
}
```

### Meaning

- `Host` is a built-in effect that represents the side effects that external functions have on the host. It has no operations. Nothing provides it: no handler can be written for it, and `with` does not remove it.
- The R of an `@external` fn with `uses` omitted is `{ Host }`. The R of an `@external suspend fn` contains `Host` whether or not `uses {}` is written. An `@external` fn that is not `suspend` and has an explicit uses clause keeps that R (`uses {}` still declares that it is pure).
- `Host` is removed from the R of the init, the release and the operation bodies of a handler. It is not removed from the types of the anonymous functions and local fns written inside them: a closure that calls an external function has `Host` in its R wherever it goes.
- The R of main, of a test function, of an `@export` fn and of the value of a const MUST be empty except for Async (`[effect.with]`), so a call whose R contains `Host` outside the bodies of a handler is a compile error (E0402). Holding such a value is not an error; calling it is.
- The rules of `[ext.call]` on references stay: naming an external function with `uses` omitted outside a handler is still E0407. A function outside a handler can get `Host` in its R only by calling a function value that was made inside a handler, so the only path to the host that a dependency shows to its dependents is still effects. A pub fn whose R contains `Host` can be called only from the bodies of a handler.
- The code users write first does not change: a handler that calls external functions directly, or through closures passed to `List.map`, `List.fold` and other functions whose parameter types omit the clauses, type-checks as before. Only code that takes the host side effect out of the handler, or that passes it to a parameter whose uses clause is written without `Host`, changes.

### Typing rules

- `Host` is Immediate (`[type.effect-bound]`): it has no suspend fn and is not Async. A closure that calls an external function can still be passed to a callback parameter of an external function.
- requires(H) (`[effect.handler]`, `[type.rules]`) never contains `Host`, because it is the union of the Rs of the bodies after `Host` is removed.
- A function type written in an effect declaration is pure when its clauses are omitted (`[type.fn]`), so an operation that returns a closure made with an external function MUST write `uses { Host }` in its return type. The caller can call the closure only inside the bodies of a handler.

```emel
effect Leak {
  fn get() -> fn(String) -> () fails {} uses { Host }
}

handler Leaker: Leak {
  fn get() { |s| host_log(s) }
}

handler Relay: Console {
  fn print_line(text) {
    f = with Leaker { (use Leak).get() }
    f(text)
  }
  fn print_error_line(text) { () }
}

fn main() {
  f = with Leaker { (use Leak).get() }
  f("outside") # E0402: Host is not allowed in the R of main
}
```

## Specification changes

### `[effect.host]` (new)

Added to Chapter 8 after `[effect.label]`, as 8.10.

```markdown
## 8.10 Host `[effect.host]`

- Host is a built-in effect that represents the side effects that external functions have on the host (Chapter 12). It has no operations. `handler H: Host` is a compile error (E0413), and `use Host` is a compile error (E0414). Host can be written in a uses clause (`fn f() uses { Host }`).
- The R of an `@external` fn with uses omitted is `{ Host }`. The R of an `@external suspend fn` contains Host, whether or not a uses clause is written (`[ext.call]`).
- Nothing provides Host, and with does not remove it. Host is removed from the R of the init, the release and the operation bodies of a handler (`[effect.handler]`). It is not removed from the types of the anonymous functions and local fns written inside them.
- Therefore a function value that performs a host side effect has Host in its type, and it can be called only inside the bodies of a handler. The R of main, a test function, an `@export` fn and the value of a const cannot contain Host (`[effect.with]`, E0402).
- Host has no suspend fn and is not Async, so it is Immediate (`[type.effect-bound]`).
```

### `[ext.call]`

Replace the table row and the bullets that begin "The restriction on where a function can be called" and "The host side effects of an external function" with:

```markdown
| `uses` omitted | Has host side effects. Its R is `{ Host }` | Named only inside the init, the release and the operation bodies of a handler |

- The restriction on where an external function with `uses` omitted can be named applies to references, not only to calls (E0407). The inside of an anonymous function inside the init, the release or an operation body of a handler counts as inside.
- A function value that refers to such a function, and a closure that calls it, can leave the handler, but its R contains Host (`[effect.host]`). Host is removed only from the R of the bodies of a handler, so the value can be called only inside the bodies of a handler. Calling it elsewhere is a compile error (E0402, or E0403 where it is passed to a function type whose uses clause does not contain Host).
- The host side effects of an external function are put in R as Host. An external function with an explicit uses clause that is not a `suspend fn` has the R that is written, and `uses {}` declares that it is pure.
```

and replace "An external function declared as `suspend fn` can be used only inside a handler, even when `uses {}` is written (E0407)." with:

```markdown
- An external function declared as `suspend fn` can be named only inside a handler, even when `uses {}` is written (E0407). Its R contains Async and Host.
```

### `[effect.handler]`

Replace "requires(H) (9.4) is the union of the Rs of init, release, and the operation bodies." with:

```markdown
- requires(H) (9.4) is the union of the Rs of init, release, and the operation bodies, after Host is removed from each (`[effect.host]`).
```

### `[diag.list]`

- E0402: append to the help: ``When the missing effect is `Host`, the help is ``= help: `Host` comes from external function `host_log`; call it inside a handler`` ``.
- E0403: append: ``When the missing effect is `Host`, the note is ``= note: `Host` comes from external function `host_log` `` ``.
- E0413: add ``handler `Runtime` cannot implement `Host`: only the bodies of a handler discharge it`` to the messages, and "8.10" to the sections.
- E0414: extend to `use Host`, with the message `` `Host` has no operations and cannot be used with `use` ``.

### `[glossary.terms]`

Add `Host` to "Used as they are", after `Async`.

## Diagnostics

- [ ] E0402: a help for `Host` that names the external function the effect came from
- [ ] E0403: a note for `Host` that names the external function the effect came from
- [ ] E0413: also reported for `handler H: Host`
- [ ] E0414: also reported for `use Host`

No codes are added or removed.

## Compatibility

Programs that compile now and stop compiling:

- A program that returns, or stores in State or a field, a function value that performs a host side effect and calls it outside a handler. This is the hole being closed; it was already against `[ext.call]`.
- A program that passes a closure calling an external function to a parameter whose uses clause is written without `Host`. The bundled std has two such functions, `next_line` and `rest_text` in `Std.Process` (`uses { Async }` becomes `uses { Async, Host }`). Neither is pub.

A survey of the bundled std, by instrumenting the check of E0407 and running `emela check` on every project under `examples/` and `tests/`: 44 references to external functions with `uses` omitted or `suspend fn` inside handlers. 35 are direct calls in a body, 9 are calls inside anonymous functions (`SystemStdin` passes them to `next_line` and `rest_text`; `SystemHttp` passes them to `List.fold` and `List.map`), and 0 are references as values. Only the two functions above need a change. Pomes outside this repository (`brix` and others) were not surveyed.

Programs that keep compiling do not change meaning. `Host` has no run-time representation.

A module that defines its own `Host` shadows the built-in name with W0202 (`[module.resolution]`), as for any name added to the Prelude.

## Implementation

- `emela-resolve`: add `Host` to `PRELUDE_EFFECTS` (`builtins.rs`).
- `emela-types`:
  - the R of an external fn with `uses` omitted, and of an external `suspend fn`, contains `Host` (`signature.rs`)
  - remove `Host` from the R of init, release and operation bodies before requires(H) is computed (`body.rs`, `body/effects.rs`)
  - E0413 and E0414 for `Host`, in the same places as for `Async`
  - the E0402 help and the E0403 note
- `emela-core`: `next_line` and `rest_text` in `std/process.emel` take `uses { Async, Host }`.
- `emela-driver`: messages, and tests for the leaks of the Motivation (a closure, the function value, a closure stored in `State`), each reported at the call in `main`, and for the std forms that keep compiling.
- No change to codegen: `Host` has no operations and no handler, and it is removed before the R of a handler reaches a with.

No prototype of the change has been written. The leak in the Motivation was reproduced with `8b41f27`.

## Alternatives

- **Delete rule 2 of `[ext.call]`.** The implementation stays as it is, and handing out a closure that performs host side effects becomes the handler author's responsibility, as the correctness of `uses {}` is the declarer's. Rejected: a type that says `uses {}` would not mean pure, test handlers cannot replace such a closure, and `[module.pome]` loses its guarantee that dependencies reach the host only through effects.
- **Make the rule on references stricter.** An external function with `uses` omitted could be named only as the callee of a direct call in a body, not as a value and not inside an anonymous function; a call runs on the spot, so nothing can leave. It needs no change to R. Rejected: it forbids closures that never leave the handler, and the bundled std uses them in `SystemStdin` and `SystemHttp` (9 references); they would have to be rewritten as recursion.
- **Track external functions in capture sets (`[effect.escape]`).** A closure that refers to an external function would be treated as holding a capability of the handler and could not leave it. Rejected: it forbids holding rather than calling, so a closure could not be passed from one handler to another; capture sets are not part of function types, so a function such as `next_line` could not say that it calls such a closure; and an external function is not tied to one with, as a capability is.
- **Check at run time.** Calling an external function outside the dynamic extent of a handler body would be a defect. Rejected: every call of an external function would pay for the check, and the error would be found when the code runs, not when it compiles.
- **Make the rule on references redundant.** With `Host` in R, E0407 could be dropped for external functions with `uses` omitted: a helper fn that calls one directly would get `Host` in its R and could be called only from handlers. This is kept for an open question below, not proposed, because it lets a dependency's pub fn give dependents a path to the host that is not an effect.

## Prior art

- A built-in effect for interaction with the outside world, which no user handler provides and which only the top level, or the runtime, discharges: the `io` family of effects in Koka, the `IO` ability in Unison, the IO monad in Haskell. Here the discharge happens at the bodies of a handler instead of at the top level, so that the outside world stays behind named effects.
- Foreign functions given an effect by their declaration: `extern` declarations with an effect in Koka.

## Open questions

- The name: `Host`, or another (`Extern`, `Io`, `Foreign`).
- Whether to drop E0407 for external functions with `uses` omitted and rely on `Host` alone (see Alternatives). The proposal keeps it.
- Whether `Host` can be written in the uses clause of any declaration, as `Async` can, or only in function types of parameters and of effect operations. The proposal allows any uses clause; the std change above needs it on `next_line` itself.
- Whether an `@external suspend fn` declared with `uses {}` should contain `Host`. The proposal says yes, matching the current rule that it can be named only inside a handler.
- Whether the E0402 and E0403 reports for `Host` deserve a code of their own instead of a help and a note.

## Decision
