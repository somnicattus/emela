# 7. Modules and packages

The examples so far fit in one file. A real program is split into modules, each with a purpose, and uses code that other people have published. This chapter covers both: how files become modules, how a module decides what others can see, and how a project depends on other projects.

## One file, one module

A module is a file. Its name comes from its path under `src/`: each part of the path is turned from `snake_case` into `PascalCase`, and the parts are joined with `.`.

```text
shop/
├── Pome.toml
└── src/
    ├── main.emel               # the entry point, `main`
    ├── email.emel              # module Email
    └── billing/
        └── invoice.emel        # module Billing.Invoice
```

There is no declaration inside a file that names its module, and no way to put two modules in one file. File and directory names are lower case; `json_codec.emel` is the module `JsonCodec`. Two modules cannot import each other, directly or through others.

`src/email.emel`:

```emel
pub opaque type Email(value: String)

pub error InvalidEmail(input: String) derive Show

pub fn parse(input: String) -> Email fails InvalidEmail {
  if String.contains(input, "@") { Email(value: input) } else { fail InvalidEmail(input:) }
}

pub fn to_string(email: Email) -> String {
  email.value
}
```

`src/billing/invoice.emel`:

```emel
import Email.{Email}

pub type Invoice(to: Email, cents: Int)

pub fn total(invoices: List[Invoice]) -> Int {
  List.fold(invoices, from: 0, f: |sum, invoice| sum + invoice.cents)
}

pub fn summary(invoice: Invoice) -> String {
  "#{invoice.to.to_string()}: #{format_cents(invoice.cents)}"
}

fn format_cents(cents: Int) -> String {
  rest = cents % 100
  padding = if rest < 10 { "0" } else { "" }
  "#{cents / 100}.#{padding}#{rest}"
}
```

`src/main.emel`:

```emel
import Billing.Invoice.{Invoice}
import Email.{InvalidEmail}

fn main() {
  ada = Email.parse("ada@example.com")
  invoices = [Invoice(to: ada, cents: 1250), Invoice(to: ada, cents: 305)]
  dbg(Invoice.total(invoices))
  dbg(invoices |> List.map(Invoice.summary))
  checked = Some(Email.parse("not an email")) escape {
    InvalidEmail(_) -> None
  }
  dbg(checked)
  ()
}
```

```console
$ emela run
1555
["ada@example.com: 12.50", "ada@example.com: 3.05"]
None
```

## import

`import Billing.Invoice` makes the module available under its last name, `Invoice`, and its public names are written with that name in front: `Invoice.total(invoices)`. The names listed in braces after it can also be written alone: after `import Billing.Invoice.{Invoice}`, the type `Invoice` is written without a prefix, and `Invoice.total` still works.

- There is no form that imports every name of a module. Each name used without a prefix is listed.
- There are no aliases. A module is known by the last part of its name.
- Imports come first in a file, before any declaration.
- A module, a type and a function live in separate namespaces, so the module `Email` and the type `Email` do not clash. Which one a name means depends on where it is written: `Email.parse` is the module, and `to: Email` is the type. `import Email` alone makes only the module visible; the type is then written `Email.Email`, or imported by name as `import Email.{Email}`.
- The modules of the standard library that the compiler knows directly, such as `List`, `String`, `Option` and `Int`, need no import. The modules under `Std` do (chapter 6).

A method call (chapter 3) needs no import either. `invoice.to.to_string()` in `summary` finds `to_string` in the module that defines `Email`, because that is where the type of `invoice.to` comes from.

## pub

Everything a module declares is private to it unless it is marked `pub`. Another module that calls `Invoice.format_cents` does not compile:

```console
$ emela check
error[E2104]: `format_cents` is private to module `Billing.Invoice`
   ╭─[ src/main.emel:4:15 ]
   │
 4 │   dbg(Invoice.format_cents(5))
   │               ──────┬─────
   │                     ╰───────
───╯

1 error
```

`pub` goes on top-level declarations: functions, types, enums, errors, constants, and the effects, handlers, layers and traits of Part II. The fields of a public type and the variants of a public enum are public with it; there is no way to hide one field or one variant. Names a module imports are not passed on: importing `Billing.Invoice` does not make `Email` visible to the importer.

A public function is part of what the module promises to other modules, so its signature is usually written out in full, with its result type, `fails` and `uses`. They can be inferred (chapter 2), and the language server offers to write the inferred ones in.

## opaque: hiding a representation

`pub opaque type Email(value: String)` makes the type public and its contents private. Outside the module `Email`, a value of type `Email` cannot be constructed, taken apart with `match`, or have its field read. A `main` that tries `forged = Email.Email(value: "nobody")` does not compile:

```console
$ emela check
error[E2105]: cannot construct opaque `Email` outside module `Email`
   ╭─[ src/main.emel:4:18 ]
   │
 4 │   forged = Email.Email(value: "nobody")
   │                  ──┬──
   │                    ╰────
───╯
   = help: `Email` is opaque: use the functions of module `Email` to make one

1 error
```

The only way to get an `Email` from outside is `Email.parse`, which checks the text first. So a value of type `Email` anywhere in the program is a string that contains `@`: the type carries the result of the check, and no function that receives an `Email` has to check it again. `Invoice` holds an `Email` for that reason.

`opaque` works on enums too (`pub opaque enum`). Use it whenever a type has a rule its values must keep, or when the representation should be free to change without breaking other modules.

## Pome: projects and dependencies

A *Pome* is a project: a directory with a `Pome.toml`. It is also the unit that other projects depend on. `emela new shop` creates one with a `main`; `emela new --lib parser` creates a library, whose root module is `src/lib.emel`.

```toml
[pome]
name = "github.com/your-name/shop"
version = "0.1.0"
emela = "0.29.0"
```

- `name` is where the Pome lives: the path of its Git repository, without `https://`. There is no central registry and no account to create. A Pome is published by pushing a Git tag such as `v0.1.0`.
- `version` is the version of this Pome, and `emela` is the oldest compiler that can build it.

### Depending on a Pome

`emela pome add` adds a dependency, by the path of its repository:

```console
$ emela pome add --dev github:emela-lang/brix
```

It writes the dependency into `Pome.toml` and the exact versions of everything it needs into `Pome.lock`:

```toml
[dependencies]
"github.com/acme/money" = "v1.2.0"

[dev-dependencies]
"github.com/emela-lang/brix" = "v0.2.0"
```

- A version in `Pome.toml` means "this version or later". `Pome.lock` records the version actually used of every Pome the project needs, directly or not, with its commit and a checksum of its contents. The `emela pome` commands write it, and it is not edited by hand. Without it, `emela run` and the other commands stop and suggest `emela pome install`, so it belongs with the project.
- A dependency is fetched through a proxy, and its contents are checked against a checksum database, so a tag that is moved after publishing is noticed (`[pome.overview]`).
- `[dev-dependencies]` are used only by tests (chapter 8). `brix`, a set of helpers for assertions, is the usual first one.
- `emela pome update`, `remove`, `list` and `verify` do what their names say (`[cmd.pome]`).

The modules of a dependency appear under its root name, by default the last part of its path in `PascalCase`: `github.com/acme/money` gives `Money`, its `src/lib.emel` is `import Money`, and its `src/convert.emel` is `import Money.Convert`. A project can see only the Pomes it lists itself; the dependencies of a dependency are not visible to it.

A Pome that calls into the host directly, through external functions (chapter 16), can do things that ordinary Emela code cannot. A project has to allow each such dependency by name, in `[external]` of its `Pome.toml`, and `emela pome add` refuses to add one that is not allowed (`--allow-external` allows it).

### Versions of your own Pome

Before a new version of a library is tagged, `emela pome publish --dry-run` compares its public functions and types with the previous version. If something public was removed or changed, it warns that the version needs a larger step (`[cmd.pome]`). Adding a variant to a public enum counts as a change that breaks users, because their `match` on it no longer covers every case.

## Summary

- A file is a module, named after its path: `src/billing/invoice.emel` is `Billing.Invoice`.
- `import A.B` makes `B` available as a prefix; `import A.B.{x, Y}` also makes `x` and `Y` usable alone. There are no wildcard imports and no aliases.
- Declarations are private unless marked `pub`. `pub opaque` makes a type public and its contents private, so its module can guarantee a rule about its values.
- A Pome is a project with a `Pome.toml`. It is identified by its Git repository and published with Git tags. Dependencies go in `[dependencies]` and `[dev-dependencies]`, and `Pome.lock` fixes their versions.

Chapter 8 writes tests: functions marked `@test`, assertions, and handlers that let a test control the effects of the code it tests.
