# 10. エフェクトとハンドラを定義する

第 6 章では標準ライブラリのエフェクトとハンドラを使いました。この章では、自分でそれらを宣言する方法を解説します。`effect` は操作のインターフェースを定義し、`handler` はそれらを実装し、`layer` は複数のハンドラを束ね、ラベルは同じエフェクトの複数用途を区別し、ケイパビリティ値（`use E`）は特定のハンドラをその `with` の外へ漏らさずに明示的に渡せるようにします（`[effect]`）。

## エフェクトとハンドラを宣言する

`effect` は呼び出し側が実行できる操作を列挙し（`[effect.decl]`）、`handler` は 1 つ以上のエフェクトのすべての操作を実装します（`[effect.handler]`）。

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

- `PrefixLogger(prefix: String)` はフィールド `prefix` を宣言しています。フィールドを持つハンドラは必ず `init` ブロックを宣言しなければなりません。`init` は `with` に入るときに 1 回実行され、`Self(...)`（または `PrefixLogger(...)`）でハンドラ値を構築します。フィールドを持たないハンドラに `init` は必須ではありませんが、初期化処理がある場合は（`Self` または自身の名前を返す形で）書くことができます。ハンドラ値は自身の `init` の中でしか構築できないため、`init` と `release` がスキップされることは決してありません。
- `init(prefix: String)` は型を明示したパラメータを取り、呼び出し側は `with` の箇所で `PrefixLogger(prefix: "app")` のように引数を渡します。引数はどの `init` も実行される前に、`with` の外側で評価されます（`[effect.with]`）。
- 操作のパラメータの型と戻り値の型は `effect Logger` から決まるため、ハンドラでは省略します。操作の本体や `release` の中では、`self` がハンドラ値を指し、フィールドを読み取れます（`self.prefix`）。また `self` はそのハンドラのケイパビリティでもあり、`use Logger` 型のパラメータへ渡すことができます。
- `PrefixLogger` は `info` の中で `Console` を使っています。操作に `uses` を書くことはできず、ハンドラのヘッダにも要件は書きません。`init`、`release`、およびすべての操作本体で使われるエフェクトの和集合がそのハンドラの*要件*（`requires(H)`）になり、ハンドラを `with` に置いた場所で提供されなければなりません。要件はハンドラ側に留まるため、`PrefixLogger` を別のハンドラに差し替えても `work` の `uses` は一切変わりません。
- `effect` 内の操作は `fails E` を宣言できます。その実装は `E` の範囲内のエラーしか発生させられないため、ハンドラを差し替えても呼び出し側の `E` が変わることはありません。
- 操作名が衝突しない限り、1 つのハンドラが 1 つの `self` を共有して複数のエフェクトを実装できます（`handler SystemTx(conn: Connection): Query + Execute`）。エフェクトもハンドラも通常の型パラメータを取れます（`effect State[S]`、`handler Memory[S](value: S): State[S]`）。ハンドラのすべての型パラメータはそのエフェクト、フィールドの型、または `init` パラメータの型のいずれかに現れる必要があり、型引数は各 `with`・`layer`・`by` の箇所で 1 回決まります（`[effect.with]`）。

ソースコードで宣言するエフェクトには、少なくとも 1 つの操作またはハンドラメンバが必要です。どちらも持たないのは組み込みのホストエフェクト（`Async`、`ProcessHost`、`PageHost`）と `External` だけです。

## `State` と状態を持つハンドラ

Emela の値は決して変化せず、ハンドラのフィールドも `init` の後は不変です。インメモリストアや呼び出しを記録するテストハンドラのように、ハンドラが可変な状態を必要とする場合は、標準ライブラリの `State[S]` エフェクト（`Std.State`、`[effect.state]`）を使います。

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

- `State[S]` は `get() -> S`、`set(value: S) -> ()`、`update(f: fn(S) -> S) -> ()` を提供します。`update` の `f` は純粋であるため、複数のタスクが状態を共有している場合でも `update` はアトミックに実行されます（第 12 章）。
- `State.run(initial, body)` は `body` に `State[S]` を提供して `(result, final_state)` を返し、`R` から `State[S]` を取り除きます。`body` がエラーで失敗すると状態は破棄され、エラーだけが伝播します。失敗時にも状態を保持したい場合は、`body` の中で `escape` します。
- 型引数が異なるエフェクトは `R` の中で別物として区別されます。`State[Int]` と `State[String]` は衝突しません。同じ基底型を持つ 2 つの状態を区別したい場合は、別々の 1 フィールド型でラップする（`type Hits(n: Int)`、`type Misses(n: Int)`）か、ラベル（後述）を使います。

## `layer` でハンドラを束ねる

実際のアプリケーションでは多数のハンドラを配線し、ハンドラどうしが互いのエフェクトに依存し合うことがよくあります。`layer` は、複数のハンドラや他のレイヤをまとめた束に名前を付けます（`[effect.layer]`）。

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

- `layer` 内の項目の並び順に意味はありません。`AppLive` では、`AppLogger` の `init` が `Config` を使っているにもかかわらず `AppLogger` を `StaticConfig` より先に書いています。コンパイラが `requires(H)` から初期化順を決定し、その逆順で `release` を実行します（`[dyn.with]`、第 11 章）。
- `layer` 内の項目には引数を渡せますが（`StaticConfig(name: "billing")`）、レイヤ項目には評価順序がないため、その引数は純粋な式（`E` と `R` が空）でなければなりません。エフェクトを伴う引数は代わりに `with` の箇所で渡します。
- 1 つの `layer` 内で同じエフェクトを提供する項目を 2 つ書くとコンパイルエラーになります。レイヤ間やハンドラ要件の循環もコンパイルできません。
- レイヤは一部の要件を未解決のまま残し、外側の `with` から提供させることができます。`with` でレイヤの後ろにハンドラを並べる（`with AppLive, StaticConfig(name: "staging")`）とそのハンドラだけが差し替わり、上書きされた側のハンドラは初期化すらされません（`[effect.with]`）。

## 名前付き `with` 項目とケイパビリティ型 `use E`

`with` の無名項目はそのエフェクトを*名前で*提供するため、ブロック内のどのコードもエフェクトエイリアス（`log <- Logger`）を通じて利用できます。同じエフェクトのインスタンスを同時に 2 つ使いたい場合や、特定のハンドラを関数へ明示的に手渡したい場合は、項目に名前を束縛します（`[effect.with]`、`[effect.use]`）。

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

- `app` と `audit` は*ケイパビリティ*（または*ケイパビリティ値*）です。初期化済みのハンドラを表す値であり、型は `use Logger` です。操作はケイパビリティに対して直接呼び出します（`primary.info(msg)`）。
- 名前付き項目は名前による提供を行いません。`with app = PrefixLogger(...)` は `app` という束縛を通じてのみハンドラを利用可能にし、`log <- Logger` や同じ `with` 内の他項目の要件に対して `Logger` を提供しません。また、名前付き項目が後続の項目で上書きされることもありません。
- `layer` を名前付き項目として束縛することはできず（`with x = AppLive` はコンパイルできません）、新しいハンドラを作らないラベル項目も同様です（後述）。
- `log <- Logger` は*エフェクトエイリアス*であり、値ではありません。`log.op(...)` や `log.op` としてしか使えず、`use Logger` 型のパラメータに渡すことはできません。
- `use E` は型であり、パラメータ、フィールド、タプル、リスト、戻り値の型に書けます。複数のエフェクトを実装するハンドラは `use E1 + E2` 型の値を生成し、これは `use E1` が期待される場所へ渡せます（`E1` の操作だけに狭められます）。任意のケイパビリティを受け入れるジェネリック関数は `[C: Capability]` と書きます（`[trait.builtin]`）。`use E` は他のどのトレイトも満たさないため、2 つのケイパビリティを `==` で比較することはできません。

## ハンドラメンバ

データベースプールが接続を開く、ファイルシステムがファイルを開く、セマフォが許可を付与するといったように、エフェクト自身がスコープ付きリソースをオンデマンドで開くファクトリとして振る舞うことがあります。そのために、エフェクトは*ハンドラメンバ*を宣言できます（`[effect.decl]`、`[effect.handler]`）。

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

- `Vault` のすべてのハンドラは、同じ `init` パラメータと `fails` 節を持つ内部 `handler` として `Open` を実装しなければなりません。内部ハンドラを外側のハンドラの外から名指しすることはできず、外側のハンドラの `self` を参照することもできません。また、内部ハンドラ自身の要件は外側のハンドラ（`ConsoleVault`）の要件に加わります。
- ハンドラメンバを開けるのは、エフェクト名経由（`with bucket = Vault.Open(...)`）またはケイパビリティ経由（`with bucket = vault.Open(...)`）の名前付き `with` 項目とする場合か、別のハンドラの `init` の中（第 11 章）だけです。そのエフェクトは束縛されたケイパビリティ経由でのみ使われ、その `with` が終わるときに `release` が実行されます。
- 開くときや解放するときに中断し得るハンドラメンバは、`suspend handler` として宣言します（第 12 章の `Semaphore.Permits` がその例です）。`suspend` がない場合、その `init` と `release` は中断できません。

## ラベル

監査ログとデバッグログがどちらも `Console` を使う場合のように、あらゆる関数へケイパビリティパラメータを引き回すことなく、同じエフェクトを 2 つの異なる役割で使いたいことがあります。*ラベル*は、別のエフェクトの操作を共有する独立したエフェクトを宣言します（`[effect.label]`）。

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

- `effect AuditLog of Console` は、`Console` と同じ操作とハンドラメンバを持ちつつ `R` の中では別のエフェクトとして扱われます。エフェクトエイリアス（`audit <- AuditLog`）で操作を直接呼ぶことも、`with AuditLog { ... }` で `AuditLog` から `Console` を提供して `emit` のような通常の関数を `AuditLog` へ書き込ませることもできます。
- 基底となるエフェクトはユーザーエフェクトでなければなりません（別のラベルや、`Async`・`TaskGroup` のような組み込みエフェクトは不可）。`handler H: AuditLog` と書くことはできず、代わりに `AuditLog by H` で任意の `Console` ハンドラを接続するか、`AuditLog by Console` で `Console` へ合流させます。
- `PrefixConsole` は `Console` を実装しつつ、`print_line` の中で `Console` も使っています。ハンドラ自身が実装するエフェクトに対する要件は、自分自身ではなく外側の `Console`（ここでは同じ `with` 内の `SystemConsole`）によって満たされるため、自分自身を呼び出してしまうことはありません（`[effect.with]`）。
- `effect DebugLog of Console default` は、明示的な `DebugLog by P` で上書きされない限り、`Console` を提供する任意の無名項目（ここでは `SystemConsole`）を含む `with` の本体へ自動的に提供されます。`default` ラベルは同じ `with` 内の他ハンドラの要件は満たさず、名前付き項目からも提供されません。`default` のないラベルは明示的に提供しなければならないため、監査ログが暗黙に標準出力へフォールバックしてしまうことはありません。
- ラベルは `effect Hits of State[Int]` のように、具体的な型引数を持つパラメータ化エフェクトに対しても使えます。

## ケイパビリティが漏洩できない理由

ケイパビリティが有効なのは、そのハンドラを初期化した `with` が生きている間だけです。その `with` が終わればハンドラの `release` が実行され、リソースはもう存在しません。そのため型システムは、ケイパビリティが（直接であっても、データ構造の中やエラーの中であっても、クロージャにキャプチャされた形であっても）自分を作った `with` の外へ出ることを禁止します（`[effect.escape]`）。

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

クロージャ `|| log.info("hi")` は `log` をキャプチャしているため、その `R` にケイパビリティ `log` が含まれます。これを `with log = ConsoleLogger` から返してしまうと、`ConsoleLogger` がすでに終わった後で `log.info("hi")` を呼び出せてしまいます。同じクロージャを `with` の*内側*で `List.each` に渡すのであればスコープを出ないため問題ありません。

エフェクトエイリアス（`log <- Logger`）の振る舞いは異なります。これは特定の `with` に紐づくケイパビリティではありません。`log.info("hi")` を呼ぶクロージャの `R` に含まれるのは `Logger`（エフェクト名）なので、自由に返すことができ、最終的に呼び出された場所で有効な `Logger` ハンドラが使われます。

## まとめ

- `effect` は操作（任意で `fails` を持ち、`uses` は持てない）と任意のハンドラメンバを宣言します。`handler Name(fields): Effect` がそれらを実装し、その要件は `init`、`release`、および操作本体から推論されます。
- 可変な状態は `State[S]` に置き、`State.run(initial, body)` で実行します。状態を持つハンドラやテストハンドラは `State[S]` を要件とします。
- `layer` はハンドラや他のレイヤを束ねます。初期化順はハンドラの要件に従い、`with` の後ろに並べた項目はレイヤ内のハンドラを上書きします。
- 名前付き `with` 項目（`with x = H`）は `use E` 型のケイパビリティ値を束縛し、`E` を名前では提供しません。ケイパビリティは関数に渡したり、狭めたり（`use E1 + E2` から `use E1`）、ハンドラメンバ経由で開いたり（`x = E.M(...)`）できます。
- ラベル（`effect L of E` および `effect L of E default`）は同じエフェクトの役割を区別し、`L` や `L by P` で配線します。
- ケイパビリティは、直接・データ構造内・エラー内・クロージャのキャプチャのいずれであっても、自分を作った `with` の外へ漏洩できません。

第 11 章ではハンドラのライフサイクル全体を掘り下げ、`init` と `release` があらゆる終了経路で確定的なリソース解放をどのように保証するかを見ていきます。
