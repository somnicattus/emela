# 11. リソースのライフタイム

Emela ではメモリは自動的に管理され、値のメモリがいつ解放されるかは規定されていません（`[dyn.memory]`）。一方で、データベース接続、開いたファイル、ネットワークソケット、セマフォの許可といった外部リソースは、メモリ回収を待つわけにはいきません。明確な時点で取得し、何かが失敗した場合も含めて速やかに解放する必要があります。

Emela はリソースのライフタイムを値ではなく `with` に結び付けます。データ型に紐づくデストラクタはありません。代わりにリソースはハンドラとして表現され、`with` に入るときに `init` がリソースを取得し、`with` を出るときに `release` が後始末を行います（`[effect.handler]`、`[dyn.with]`）。

## `init` と `release`

どのハンドラも `init` ブロックと `release` ブロックを宣言できます。

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

2 つ目のブロックでは `s.query("BAD")` が `Broken` で失敗し、`with` の外側にある `escape` がエラーを捕捉する*前*に `FakeSession` の `release` が実行されています。`init` と `release` の対応関係は構造的に保証されます。

- ハンドラ値は自身の `init` の中でしか構築できないため、`with` に入らずにハンドラを手に入れる方法はありません。
- ケイパビリティは自分を作った `with` の外へ出られないため（第 10 章）、`release` が実行された後にリソースへ触れるコードは書けません。
- 本体がどのように終わっても――正常な値、`fail`、欠陥（`panic` や失敗した `assert`）、タスクの割り込み（第 12 章）――`with` を抜けるときに `release` は必ず 1 回実行されます。
- 複数のエフェクトを実装するハンドラ（`handler H: A + B`）は、`init` を 1 回、`release` を 1 回だけ実行します。

### `release` は失敗できない

`release` のエラー集合 `E` は空でなければなりません（`[effect.handler]`）。`with` を抜ける時点ですでに別のエラーや欠陥をアンワインドしている可能性があるため、後始末が独自の型付きエラーを持ち込むことはできません。

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

後始末の中で失敗し得る処理を呼ぶ場合は、`release` の中で `escape` を使ってエラーを処理する（たとえばログを出して続行する）か、`panic` で欠陥に変えます。`release` の戻り値は捨てられるため、型は何でも構いません。通常の `fn` 操作とは異なり、`release` は `suspend fn` 操作を呼び出すことができ（第 12 章）、実行中に割り込まれることはありません。

## 初期化と解放の順序

複数の項目を持つ `with`（`with H1, ..., Hn { body }`）は 4 つのステップで実行されます（`[dyn.with]`）。

1. どの `init` も実行される前に、`H(arguments)` の引数を `with` の外側で左から右へ評価します。この `with` の項目が提供するエフェクトは引数のスコープには入りません。無名項目が同じエフェクトを提供する後ろの項目によって上書きされる場合、その引数は型検査こそされますが評価はされず、`init` も実行されません（`[effect.with]`）。
2. 項目を要件に従って並べ替え、各 `init` を 1 回ずつ実行します。同じ `with` 内の別項目の `requires(H)` を満たす無名ハンドラが先に初期化され、要件による制約がない部分では記述順が保たれます（名前付き項目は名前による提供を行わないため、要件を満たしません）。ハンドラメンバ項目（`x = E.M(...)` や `x = c.M(...)`）の順序は `E` または `c` だけで決まり、内部ハンドラの要件は考慮されません。
3. 本体を評価します。
4. 初期化の逆順で `release` を実行し、その後、本体の結果（値、エラー、欠陥、または割り込み）を返します。

`release` が初期化の逆順で実行されるため、ハンドラの `release` は `requires(H)` に含まれるどのエフェクトも安全に使えます。それらのエフェクトを提供するハンドラはまだ生きており、解放されるのはその後だからです。

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

`LiveDb` は `EnvConfig` より前に書かれていますが、`LiveDb` が `Config` を要求するため、`EnvConfig` が先に初期化され、最後に解放されます。

## `init` と `release` の最中の失敗

`release` は、対応する `init` が値を返した場合に限り実行されます（`[dyn.with]`）。

- `H(args)` の引数の評価中に失敗した場合、どの `init` もまだ実行されていないため何も解放されず、`with` はそのエラーで終了します。
- `init` が失敗した場合（先ほどの `LiveDb(should_fail: True)` のように）、本体はスキップされます。`LiveDb` の `init` は値を返していないため `LiveDb` の `release` は実行*されず*、`init` を完了していたすべてのハンドラ（`EnvConfig`）が逆順で解放されます。

`release` の中で欠陥が起きた場合は、どの後始末もスキップされないよう一定の優先順位に従います（`[dyn.defect]`）。

- 本体が正常な値、エラー、または割り込みで終わっていた場合、`release` 内の欠陥がその `with` の結果になります。
- 本体がすでに欠陥で終わっていた場合、最初の欠陥がそのまま結果として残り、各 `release` の欠陥はノート（`note: the release of handler <name> also failed: <message>`）として追記されます。
- いずれの場合も、残りのすべてのハンドラの `release` は引き続き実行されます。

## `init` の中でリソースを開く

ハンドラの `init` は、リポジトリハンドラが `Pool` から接続を借りるように、別のエフェクトのハンドラメンバからサブリソースを取得したいことがよくあります。`init` の中では、ハンドラメンバを式として直接呼び出して（`Pool.Lease(...)` または `pool.Lease(...)`）、開いたハンドラのケイパビリティをフィールドに格納できます（`[effect.handler]`、`[dyn.with]`）。

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

`Lease` のライフタイムは `RepoHandler` に結び付けられています。

- `RepoHandler` の `release` が*先*に（`self.conn` がまだ生きている間に）実行され、その後に `init` の中で開かれたすべてのハンドラが開いた順の逆順で解放されます。
- `RepoHandler` の中で `self.conn.exec(...)` を呼んでも、`Conn` は `requires(RepoHandler)` に加わりません。`conn` は `RepoHandler` と同じ期間生き続けるため、`RepoHandler` が要求するのは `Pool`（と `Console`）だけです（`[effect.use]`）。
- `RepoHandler` の `init` がいくつかのサブハンドラを開いた後に途中で失敗した場合、エラーが伝播する前にそれらのサブハンドラが逆順で解放されます。
- `init` の中で内側の `with` を使って開いたサブハンドラはその内側の `with` より長く生きられず、`Self(...)` に格納することもできません。その内側の `with` が終わるときに解放されます。

## `release` が実行されない場合

`release` はすべての正常終了、型付きエラー、欠陥、割り込みで実行されます。仕様では、`release` が実行されないか途中で打ち切られ得る状況を次の 3 つだけに限定しています（`[dyn.with]`）。

- プログラムが必要なメモリを確保できない場合。このとき解放処理が実行されるかどうかは実装定義です（`[dyn.memory]`）。
- イベントループが空になった場合（待機中のホスト処理もなく、前進できるファイバもない状態）。中断している `release` は打ち切られます。ただし `Task.Group` や `Task.Background` の `release` は例外で、割り込まれたタスクのアンワインド完了を待ちます（`[dyn.suspend]`）。
- プロセス自体が突然終了した場合（Ctrl-C などの OS シグナル、外部関数による `abort` の呼び出し、あるいは同じスレッド上で別の実行（launch）がすでに進行中の状態でもう一度 `@export` 関数を呼んだ場合）。それ以降の Emela コードは実行されません。

それ以外のすべてのケースでは、`with` に入ることと出ることは厳密に対応します。

## まとめ

- リソースは値のデストラクタではなく、`with` にスコープされたハンドラ（`init` と `release`）によって管理されます。
- `init` が値を返したすべてのハンドラについて、本体が正常に返る・失敗する・欠陥で停止する・割り込まれるのいずれであっても、`release` が 1 回実行されます。
- `release` のエラー集合 `E` は空でなければならず、中断し得る操作を呼び出すことができ、実行中に割り込まれません。
- 1 つの `with` の中では、引数が `with` の外側で左から右へ評価され、ハンドラが要件順に初期化され、解放は初期化の逆順で実行されます。
- ハンドラの `init` はハンドラメンバを直接開いて（`Pool.Lease(...)`）そのケイパビリティをフィールドに格納できます。それらのサブハンドラは所有ハンドラの `release` の*後*に逆順で解放されます。
- `release` の欠陥によって残りの `release` がスキップされることはありません。本体がすでに欠陥で停止していた場合、`release` の欠陥はノートとして記録されます。

第 12 章では、エフェクト・ハンドラ・`release` の土台の上に、中断と構造化並行性を構築します。
