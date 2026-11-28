# The Emela Programming Language

A book that teaches Emela from the first program to the boundary with the host. It describes the 0.20 series of the language, as specified in [`spec/`](../spec/README.md).

The book is informative. Where it and the specification disagree, the specification is right, and the book is to be fixed.

## How the book is organized

- **One path through the language.** The chapters build on each other in order. Readers who already know some of the ideas skip ahead with the reading paths below, instead of reading a separate book.
- **The two levels of the language are the first two parts.** Part I covers what people who write applications use; Part II covers what people who write libraries and runtime boundaries use (`[intro.layers]`). Part I does not depend on Part II.
- **Effects and handlers are the hub.** Resources, structured concurrency, designing with interfaces, and external functions are all built from handlers, so they come after the chapter that defines them.

## Contents

Chapters without a link are not written yet.

[Introduction](00-introduction.md): the three pillars and the fixed premises, the two levels, installing Emela, a first program run on JS and WASM and tested, and how to read the book. (`[intro]`)

### Part I. Everyday Emela

1. [Data](01-data.md): basic types, `type`, `enum`, tuples, type parameters, `const`, using `derive`. (`[data]`, `[trait.derive]`)
2. [Functions and expressions](02-functions.md): `fn`, argument names, type parameters, blocks and bindings, operators, `if`, `match` and patterns. (`[fn.decl]`, `[fn.args]`, `[fn.block]`, `[fn.operator]`, `[fn.if]`, `[fn.match]`)
3. [Functions as values](03-functions-as-values.md): passing functions, anonymous functions and what they capture, function types, the pipe `|>`, method calls `x.f(a)`, local functions. (`[fn.decl]`, `[type.fn]`, `[fn.pipe]`, `[type.rules]`, `[fn.local]`)
4. [Repetition without changing values](04-repetition.md): the functions that go over a list, `List.fold`, recursion, the guarantee for tail calls, building a list, and immutability. (`[fn.loop]`, `[api.list]`, `[dyn.memory]`)
5. [Errors](05-errors.md): `error`, `fail`, the `fails` clause, `escape`, errors of functions passed as arguments, choosing between an Option and an error, errors that reach `main`, and defects. (`[error]`, `[dyn.error]`, `[lib.api-rules]`)
6. [Using effects](06-using-effects.md): effect aliases `x <- E` and operations, `uses`, `with`, the handlers and the layers `Platform` and `Process` of the standard library, replacing a handler, effects of function values, and calls that wait. (`[effect.use]`, `[effect.with]`, `[lib.layers]`)
7. [Modules and packages](07-modules.md): files as modules, `import`, `pub`, `opaque`, Pomes, dependencies and the lock, `@external` permissions, and checking the versions of a library. (`[module]`, `[pome.overview]`, `[cmd.pome]`)
8. [Testing](08-testing.md): `@test`, `assert` and its report, testing failures, test handlers in `State.run`, virtual time, the test root, and `emela test`. (`[test]`, `[cmd.test]`, `[api.test-handlers]`)

### Part II. The foundation

9. [Traits](09-traits.md): `trait`, `impl`, bounds, and the details of `derive`. (`[trait]`)
10. [Defining effects and handlers](10-effects-and-handlers.md): `effect`, `handler` with fields and state, `layer`, named `with` items, handler members, labels, State, capability types `use E`, and why capabilities cannot leak. (`[effect]`, `[effect.layer]`)
11. [The lifetime of resources](11-resources.md): `init` and `release`, the scope of a `with`, release on failure, and the order of release. (`[effect.handler]`, `[dyn.with]`, `[dyn.memory]`)
12. [Suspension and structured concurrency](12-concurrency.md): `suspend fn` operations and why calls carry no mark, Async and Immediate, a handler that resumes now or later, tasks on fibers that switch only at suspension points, `Task.Group` and `Task.Background`, the combinators, interruption, causes of failure and `Exit.attempt`, Deferred, Semaphore and Queue. (`[effect.decl]`, `[effect.async]`, `[type.effect-bound]`, `[task]`, `[dyn.suspend]`)
13. [Designing with composition](13-composition.md): hiding representation with `opaque`, traits as interfaces, effects as interfaces with handlers as their implementations, and delegating to a capability held in a field.
14. [What the types rule out](14-what-types-rule-out.md): what E and R record, the containment of E and R, Immediate, capabilities that cannot leak, and exhaustiveness. (`[type]`)

### Part III. Hosts and targets

15. One program, three targets: what the specification makes the same on JS, WASM and native, what is implementation-defined, and what each target does not support yet. (`[target]`)
16. External functions: `@external`, host types, boundary types, the External effect, `@export` and entry adapters. (`[ext]`, `[effect.external]`)
17. How Emela is compiled: specialization and monomorphization, state machines and generators, reference counting. This chapter describes the current implementation, not a guarantee of the language. (`[target.impl-hint]`)

### Part IV. Tools

18. The commands, Pome, the formatter, the test runner and the language server, as a reference. Each tool is first introduced in Part I where it is first used. (`[cmd]`, `[pome.overview]`, `[fmt]`, `[cmd.test]`, `[lsp]`)

### Appendices

- A. Coming from another language: Java; Haskell; Rust; Gleam; ZIO and Effect-TS; Koka, OCaml 5 and Effekt; languages with `async` and `await`; C, WASM and JS as targets; languages with similar aims. Each guide covers data, errors, effects, and suspension and concurrency.
- B. What Emela does not have, and why. (`[intro.premises]`, `[error.not]`, `[type.not]`, `[trait.not]`)
- C. Index of the sections of the specification that each chapter relies on.

## Reading paths

| If you know | Already familiar | New in Emela | Read |
|---|---|---|---|
| Java | Interfaces, dependency injection, checked exceptions | Data defined by its cases, no loops, no classes or inheritance | Introduction, all of Part I, then 12, 13 |
| Haskell | Algebraic data types, closures, type classes | Effects in place of monads, no higher-kinded types, no Result type | Introduction, 5, 6, Part II, Appendix B |
| Rust | Enums with data, `match`, traits and `derive`, Option | No Result type or `?`, no ownership or borrowing, no mutable variables, effects | Introduction, 4 to 6, 9 to 12, 17 |
| Gleam | Immutable values, no loops, the pipe, custom types, a JS target | Errors outside the return type, effects and handlers, traits | Introduction, 5, 6, 9, 10, 12 |
| JavaScript, TypeScript, Python, C#, Kotlin, Swift | `async` and `await` or `suspend fun`, promises, cancellation | Suspension without marks on functions or calls, typed errors, effects | Introduction, all of Part I, then 10, 12 |
| ZIO, Effect-TS | Typed error and requirement channels, layers, fibers | Direct style, handlers as part of the language | Introduction, 5, 6, 10 to 12 |
| Koka, OCaml 5, Effekt | Effect handlers | One-shot continuations only, errors apart from effects, `init` and `release`, capabilities that cannot leak | Introduction, 5, 10 to 12, 17 |
| C, WASM, JS as targets | Output formats, host interfaces | How effects meet the host | Introduction, 1 to 6, Part III |

### Suspension and concurrency

Most readers arrive with a model of waiting and concurrency that differs from Emela's more than anything else does, so the guides of Appendix A compare it on its own. What every guide starts from:

- There is no syntax for asynchrony. Only the declaration of an effect operation says that it may suspend (`suspend fn`); function signatures and calls carry no mark. Whether a function may suspend shows in its R, which is inferred.
- The handler decides. The same function suspends under a handler that resumes later, and does not under a handler that resumes on the spot; then Async does not appear in the type of the `with` either. Where suspending would be wrong, such as in a callback given to the host, the type asks for Immediate.
- Tasks run on fibers that switch cooperatively on one thread, only at calls that can suspend. The runtime yields for fairness after a count of such calls, not after an amount of time, so its yields fall at the same places on every run. A stretch with no such call runs as one step. There is no parallelism, and no threads or locks.
- A task is started only in a `Task.Group` or `Task.Background`, and ends before the `with` that started it does. There is no function that interrupts a single task.
- A continuation is resumed at most once.

| If you know | What you use there | What to look for in Emela |
|---|---|---|
| Java | Threads, virtual threads, executors, locks | One thread and no locks; tasks bounded by a `with` |
| Haskell | `IO`, `forkIO`, `async`, STM | Waiting as an effect operation, not a monad; atomic stretches between suspension points instead of transactions |
| Rust | `async fn`, `.await`, futures, an executor, `Send` | No coloring of functions; the runtime is part of the program; nothing is shared between threads |
| Gleam | Processes and messages on the BEAM | Tasks joined for their results instead of processes with mailboxes |
| ZIO, Effect-TS | Fibers, `fork`, interruption, scopes, Deferred, Semaphore, Queue | The same tools in direct style; no `fork` outside a group, no function to interrupt one fiber |
| Koka, OCaml 5, Effekt | Schedulers and async written as handlers | Async is provided only by the host; a user handler of a `suspend fn` chooses to resume now or later |
| `async` and `await` languages | Promises or tasks, `await` at each call, cancellation tokens | Calls without `await`; interruption at suspension points, with releases run on the way out |

## Writing the book

- Each chapter states the rules in its own terms. Comparisons with other languages go in Appendix A, not in the chapters.
- Section names of the specification (`[effect.handler]`) are given where a chapter relies on a rule, so that a reader can look up the exact wording.
- Code examples match the output of `emela fmt` and are checked to compile with the current compiler. The output shown is the output of `emela run` on the JS target unless the text says otherwise.
- The current implementation is kept apart from the language. A sentence about what the compiler does today says so.
