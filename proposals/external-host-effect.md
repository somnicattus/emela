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

An `@external` fn with `uses` omitted can be used only inside a handler, and `[ext.call]` says that it cannot be carried out of a handler as a function value either. Nothing in the specification enforces that second rule: a closure that calls such a function, or the function value itself, can be returned from an operation and called from `main`. This proposal adds a built-in effect, `Host`, that is the R of these external functions and that is removed only from the R of the bodies of a handler. A value that carries a host side effect then carries `Host` in its type, and calling it outside a handler is a compile error by the existing rules on R. `Host` can be written only in the signatures of fns that are not pub and in bodies, so it never appears in the type of a pub fn or of an effect operation.

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

No new syntax. `Host` is a new name, a built-in effect, that can be written in some uses clauses (see "Where Host can be written").

```emel
fn run_twice(step: fn() -> () uses { Host }) uses { Host } {
  step()
  step()
}

effect Leak {
  fn get() -> fn(String) -> () uses { Host } # E0444
}
```

### Meaning

- `Host` is a built-in effect that represents the side effects that external functions have on the host. It has no operations. Nothing provides it: no handler can be written for it, and `with` does not remove it.
- The R of an `@external` fn with `uses` omitted is `{ Host }`. The R of an `@external suspend fn` contains `Host` whether or not `uses {}` is written. An `@external` fn that is not `suspend` and has an explicit uses clause keeps that R (`uses {}` still declares that it is pure).
- `Host` is removed from the R of the init, the release and the operation bodies of a handler. It is not removed from the types of the anonymous functions and local fns written inside them: a closure that calls an external function has `Host` in its R wherever it goes.
- The R of main, of a test function, of an `@export` fn and of the value of a const MUST be empty except for Async (`[effect.with]`), so a call whose R contains `Host` outside the bodies of a handler is a compile error (E0402). Holding such a value is not an error; calling it is.
- The rules of `[ext.call]` on references stay: naming an external function with `uses` omitted outside a handler is still E0407. A function outside a handler can get `Host` in its R only by calling a function value that was made inside a handler.
- The code users write first does not change: a handler that calls external functions directly, or through closures passed to `List.map`, `List.fold` and other functions whose parameter types omit the clauses, type-checks as before. Only code that takes the host side effect out of the handler, or that passes it to a parameter whose uses clause is written without `Host`, changes.

### Where Host can be written

`Host` is a detail of the code that implements a handler. It is kept out of the types that other modules see, so that a dependency shows the host to its dependents only as named effects (`[module.pome]`).

- `Host` MAY be written in a uses clause of the signature of a fn that is not pub (its own uses clause, and those of the function types in its parameters and return type), and in a uses clause inside a body (the type annotation of a binding or of an anonymous function's parameter, the signature of a local fn).
- Writing `Host` anywhere else is a compile error (E0444): the signature of a pub fn, an effect declaration (the types of its operations), the effect named by a handler declaration, the fields of a type, enum or error, a Trait function, a const, and a set type argument of a named type (`Handler[E, Host]`).
- The type of a pub fn after inference MUST NOT contain `Host`, neither in its R nor in a function type in its parameters or return type (E0445). A hidden set variable of a parameter is not `Host`; it becomes `Host` only at a call, in the R of the caller.

These rules keep the interface of a module free of `Host`, and with them a value whose type contains `Host` cannot leave the bodies of a handler at all. A value leaves the bodies of a handler only in these ways, and none of them can carry `Host`:

- as the result of an operation or the argument of `resume`, whose types come from the effect declaration. Omitted clauses there are empty (`[type.fn]`), `Host` cannot be written there, and the type arguments of the effect are written as concrete types in the handler declaration (`[effect.decl]`: there are no generic handlers), where `Host` cannot be written either (E0444);
- as the fields of an error, which are declarations; an error has no type parameters (`[error.decl]`);
- as the value of release, which is discarded, or as the handler value made by init, which is not seen outside the with (`[effect.handler]`).

So the operation `get` of the Summary is rejected at the handler (E0403), and the value of a const, or the result of any function that is not a body of a handler, never has `Host` in its type. E0402 is a second line: without E0444, an operation declared with `uses { Host }` could return the closure, and calling it in `main` would be E0402.

### Typing rules

- `Host` is Immediate (`[type.effect-bound]`): it has no suspend fn and is not Async. A closure that calls an external function can still be passed to a callback parameter of an external function.
- requires(H) (`[effect.handler]`, `[type.rules]`) never contains `Host`, because it is the union of the Rs of the bodies after `Host` is removed.
- A function type written in an effect declaration is pure when its clauses are omitted (`[type.fn]`), and `Host` cannot be written there, so an operation cannot return a closure that performs host side effects. The operation `get` of the Summary is E0403 at the anonymous function.
- A parameter function type with the uses clause omitted has a hidden set variable, which accepts `Host`. A function that only passes on or calls such a closure, written with the clauses omitted, needs no `Host` in its signature:

```emel
fn next_line(step: fn() -> Int, text: fn() -> String) -> Option[String]
  fails InvalidInput | StdinFailure {
  step() match {
    0 -> Some(text())
    _ -> next_line(step, text)
  }
}
```

### Trade-offs

- **Inference instead of a written signature.** A pub fn that takes a closure with `Host` in its type must leave the uses clauses of that parameter and of itself to inference (hidden set variables and an inferred R), because it cannot name `Host`. A fn whose R is inferred is type-checked with the group of definitions that refer to each other (`[type.inference]`), not from its signature alone. It costs the type checker and the LSP more work, an error in it is reported at the expression that needs the effect rather than against a written clause, and a reader learns its effects from the LSP rather than from the source. A fn that is not pub can write `Host` and keep a closed signature.
- **No host closures through operations.** A handler cannot hand a closure that performs host side effects to another handler through an operation, because the types of operations cannot contain `Host`. The effect has to expose the operation that the closure performs, or the other handler has to call its own external functions.
- **A name that is seen more than written.** `Host` appears in the types that the LSP displays and in the messages of E0402 and E0403, also where it cannot be written. The fix for such an error is to keep the host side effect inside the handler, not to write `Host`, and the messages say so.
- **Two new codes**, E0444 and E0445, and one more effect in the Rs that the type checker carries through handler bodies and closures. The removal of `Host` happens once per body of a handler, and E0445 is checked once per pub fn.

## Specification changes

### `[effect.host]` (new)

Added to Chapter 8 after `[effect.label]`, as 8.10.

````markdown
## 8.10 Host `[effect.host]`

- Host is a built-in effect that represents the side effects that external functions have on the host (Chapter 12). It has no operations. `handler H: Host` is a compile error (E0413), and `use Host` is a compile error (E0414).
- The R of an `@external` fn with uses omitted is `{ Host }`. The R of an `@external suspend fn` contains Host, whether or not a uses clause is written (`[ext.call]`).
- Nothing provides Host, and with does not remove it. Host is removed from the R of the init, the release and the operation bodies of a handler (`[effect.handler]`). It is not removed from the types of the anonymous functions and local fns written inside them.
- Therefore a function value that performs a host side effect has Host in its type, and it can be called only inside the bodies of a handler. The R of main, a test function, an `@export` fn and the value of a const cannot contain Host (`[effect.with]`, E0402).
- Host MAY be written in a uses clause of the signature of a fn that is not pub (its own, and those of the function types in its parameters and return type), and in a uses clause inside a body (the type annotation of a binding or of an anonymous function's parameter, and the signature of a local fn). Writing Host anywhere else is a compile error (E0444): the signature of a pub fn, an effect declaration, the effect named by a handler declaration, the fields of a type, enum or error, a Trait function, a const, and a set type argument of a named type.
- The type of a pub fn after inference MUST NOT contain Host, in its R or in a function type in its parameters or return type (E0445). Host is a detail of the code that implements a handler, and a module shows the host to other modules only as named effects.
- A value whose type contains Host cannot leave the bodies of a handler: the result of an operation and the argument of resume have the types of the effect declaration, the fields of an error are declarations, the value of release is discarded, and the handler value is not seen outside the with. Host is written in none of these.
- Host has no suspend fn and is not Async, so it is Immediate (`[type.effect-bound]`).

```emel
fn run_twice(step: fn() -> () uses { Host }) uses { Host } {
  step()
  step()
}

effect Leak {
  fn get() -> fn(String) -> () uses { Host } # E0444
}
```
````

### `[ext.call]`

Replace the table row and the bullets that begin "The restriction on where a function can be called" and "The host side effects of an external function" with:

```markdown
| `uses` omitted | Has host side effects. Its R is `{ Host }` | Named only inside the init, the release and the operation bodies of a handler |

- The restriction on where an external function with `uses` omitted can be named applies to references, not only to calls (E0407). The inside of an anonymous function inside the init, the release or an operation body of a handler counts as inside.
- A function value that refers to such a function, and a closure that calls it, has Host in its R (`[effect.host]`). Host is removed only from the R of the bodies of a handler, so the value can be called only inside the bodies of a handler. Calling it elsewhere is a compile error (E0402, or E0403 where it is passed to a function type whose uses clause does not contain Host). The types of operations and of pub fns cannot contain Host (E0444, E0445), so such a value cannot be handed out through them.
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

Add:

```markdown
| E0444 | `Host` written outside the signature of a fn that is not pub and outside a body (`[effect.host]`) | ``effect `Host` cannot be written in an effect declaration`` (the place is named: ``in the signature of pub function `f` ``, ``in a field``, and so on). `= help: Host stays inside the code that implements a handler` |
| E0445 | The type of a pub fn contains `Host` after inference (`[effect.host]`) | ``public function `f` performs host side effects`` with a label at the first expression that requires `Host`. `= help: call it from a handler, or make it not pub` |
```

Change:

- E0402: append to the help: ``When the missing effect is `Host`, the help is ``= help: `Host` comes from external function `host_log`; call it inside a handler`` ``.
- E0403: append: ``When the found set contains `Host` and the expected type cannot contain it, the help is ``= help: a host side effect cannot leave the bodies of a handler`` ``.
- E0413: add ``handler `Runtime` cannot implement `Host`: only the bodies of a handler discharge it`` to the messages, and "8.10" to the sections.
- E0414: extend to `use Host`, with the message `` `Host` has no operations and cannot be used with `use` ``.

### `[glossary.terms]`

Add `Host` to "Used as they are", after `Async`.

## Diagnostics

- [ ] E0444 (new): `Host` written outside the signature of a fn that is not pub and outside a body
- [ ] E0445 (new): the inferred type of a pub fn contains `Host`
- [ ] E0402: a help for `Host` that names the external function the effect came from
- [ ] E0403: a help when a host side effect would leave the bodies of a handler
- [ ] E0413: also reported for `handler H: Host`
- [ ] E0414: also reported for `use Host`

## Compatibility

Programs that compile now and stop compiling:

- A program that returns, or stores in State or a field, a function value that performs a host side effect and calls it outside a handler, or returns it from an operation. This is the hole being closed; it was already against `[ext.call]`.
- A program that passes a closure calling an external function to a parameter whose uses clause is written without `Host`. The bundled std has two such functions, `next_line` and `rest_text` in `Std.Process`, which are not pub. Their four `uses { Async }` clauses are removed, so that `Async` and `Host` are inferred. With the clauses removed and no other change, every project under `examples/` and `tests/` type-checks with `8b41f27`, and `SystemStdin` reads lines and the rest of the input as before on the JS target.

A survey of the bundled std, by instrumenting the check of E0407 and running `emela check` on every project under `examples/` and `tests/`: 44 references to external functions with `uses` omitted or `suspend fn` inside handlers. 35 are direct calls in a body, 9 are calls inside anonymous functions (`SystemStdin` passes them to `next_line` and `rest_text`; `SystemHttp` passes them to `List.fold` and `List.map`), and 0 are references as values. Only the two functions above need a change. No pub fn of std is expected to get `Host` in its type (E0445), because none can name an external function with `uses` omitted; this is to be confirmed by the implementation. Pomes outside this repository (`brix` and others) were not surveyed.

Programs that keep compiling do not change meaning. `Host` has no run-time representation.

A module that defines its own `Host` shadows the built-in name with W0202 (`[module.resolution]`), as for any name added to the Prelude.

## Implementation

- `emela-resolve`: add `Host` to `PRELUDE_EFFECTS` (`builtins.rs`).
- `emela-types`:
  - the R of an external fn with `uses` omitted, and of an external `suspend fn`, contains `Host` (`signature.rs`)
  - remove `Host` from the R of init, release and operation bodies before requires(H) is computed (`body.rs`, `body/effects.rs`)
  - E0444 where uses clauses and set type arguments are lowered, E0445 after the generalization of each pub fn
  - E0413 and E0414 for `Host`, in the same places as for `Async`
  - the E0402 and E0403 helps
- `emela-core`: remove the `uses { Async }` clauses of `next_line` and `rest_text` in `std/process.emel`.
- `emela-driver`: messages, and tests for the leaks of the Motivation (a closure, the function value, a closure stored in `State`), for each place of E0444, for E0445 with an inferred R and an inferred return type, and for the std forms that keep compiling.
- No change to codegen: `Host` has no operations and no handler, and it is removed before the R of a handler reaches a with.

No prototype of the change has been written. The leak in the Motivation was reproduced with `8b41f27`, and the change to `next_line` and `rest_text` was tried on its own.

## Alternatives

- **Delete rule 2 of `[ext.call]`.** The implementation stays as it is, and handing out a closure that performs host side effects becomes the handler author's responsibility, as the correctness of `uses {}` is the declarer's. Rejected: a type that says `uses {}` would not mean pure, test handlers cannot replace such a closure, and `[module.pome]` loses its guarantee that dependencies reach the host only through effects.
- **Make the rule on references stricter.** An external function with `uses` omitted could be named only as the callee of a direct call in a body, not as a value and not inside an anonymous function; a call runs on the spot, so nothing can leave. It needs no change to R. Rejected: it forbids closures that never leave the handler, and the bundled std uses them in `SystemStdin` and `SystemHttp` (9 references); they would have to be rewritten as recursion.
- **Track external functions in capture sets (`[effect.escape]`).** A closure that refers to an external function would be treated as holding a capability of the handler and could not leave it. Rejected: it forbids holding rather than calling; capture sets are not part of function types, so a function such as `next_line` could not say that it calls such a closure; and an external function is not tied to one with, as a capability is.
- **Check at run time.** Calling an external function outside the dynamic extent of a handler body would be a defect. Rejected: every call of an external function would pay for the check, and the error would be found when the code runs, not when it compiles.
- **`Host` can be written in any uses clause.** `Host` would be written wherever `Async` can, with no E0444 and E0445. A handler could hand host closures to another handler through an operation whose type says `uses { Host }`, and no signature would have to be left to inference. Rejected: `Host` would appear in the types of pub fns and effect operations, which are the interface of a module, and a dependency could show its dependents a path to the host that is not a named effect (a pub fn whose R contains `Host`, callable from their handlers).
- **`Host` cannot be written at all.** `Host` would exist only in inferred types, with E0444 for every place and no need for E0445's list of places. Rejected: every fn that calls or passes on a closure with `Host` in its type would have to leave its clauses to inference, including private helpers such as `run_twice` in the Syntax section; inference would become mandatory in the code that implements handlers, with the costs listed in Trade-offs, while writing `Host` in a private signature shows nothing to other modules.
- **Make the rule on references redundant.** With `Host` in R, E0407 could be dropped for external functions with `uses` omitted: a helper fn that is not pub and calls one directly would get `Host` in its R and could be called only from handlers, and E0445 would keep it out of pub fns. Not proposed, because it changes more than the hole needs; it is an open question below.

## Prior art

- A built-in effect for interaction with the outside world, which no user handler provides and which only the top level, or the runtime, discharges: the `io` family of effects in Koka, the `IO` ability in Unison, the IO monad in Haskell. Here the discharge happens at the bodies of a handler instead of at the top level, so that the outside world stays behind named effects.
- Foreign functions given an effect by their declaration: `extern` declarations with an effect in Koka.

## Open questions

- The name: `Host`, or another (`Extern`, `Io`, `Foreign`). The proposal uses `Host`, the word that Chapter 12 already uses for the other side of an external function. `Io` would suggest one effect for all input and output, which the named effects of the platform layer are meant to replace.
- Whether to drop E0407 for external functions with `uses` omitted and rely on `Host` and E0445 (see Alternatives). The proposal keeps it.
- Whether an `@external suspend fn` declared with `uses {}` should contain `Host`. The proposal says yes, matching the current rule that it can be named only inside a handler.
- Whether a set type argument (`Handler[E, Host]`) may be written in the signature of a fn that is not pub, as a uses clause may. The proposal forbids it, to keep the rule short.

## Decision
