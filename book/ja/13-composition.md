# 13. 合成による設計

Emela にはクラスも、継承も、実行時の動的ディスパッチもありません。大きなプログラムは、第 I 部と第 II 部で学んだ 3 つの小さな仕組みを合成して構築します。不変条件を守る `opaque` 型、データ型を抽象化するトレイト、そして振る舞いとリソースを抽象化するエフェクトとハンドラです。

この章では、それぞれの道具をいつ使い分けるかを比較し、振る舞いを重ねるための主要なパターン――ハンドラのフィールドに保持したケイパビリティへ委譲するパターン――を紹介します。

## `opaque` で内部表現を隠す

メールアドレスは `@` を含む、金額は負にならない、リストはソート済みである、といったように値が満たすべきルールがある型は、専用のモジュールで `pub opaque` として宣言します（第 7 章）。

`src/money.emel`：

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

`src/main.emel`：

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

- `Money` モジュールの外側から値を構築できるのは `Money.from_cents` と `Money.add` だけなので、プログラムのどこにあっても `Money` が非負であることが保証されます。
- 呼び出し側は `cents` を読んだり `Money(...)` にマッチしたりできないため、モジュールは後で呼び出し側を壊さずに内部表現を変更（たとえば `Int64` へ変更）できます。
- `opaque` 型に対して `Decode` と `ObjectKey` は derive できません（第 9 章）。自動導出されたデコーダは `from_cents` を通らずに値を構築してしまうからです。
- `Money` が `Show` を derive せず手書きで `impl Money: Show` を書いている点にも注目してください。自動導出された `Show` は `Money(cents: 1375)` と出力して隠蔽されたフィールドを見せてしまいます。

`opaque` が解決するのはデータの整合性と内部表現の隠蔽です。実行時に異なる実装を切り替えるためのものではありません。

## データに対するインターフェースとしてのトレイト

ある操作が値の型だけに依存し、型ごとに 1 つの標準的な意味しか持たない場合は `trait` を使います。等価性（`Eq`）、順序（`Ord`）、整形（`Show`）、シリアライズ（`Encode`、`Decode`、`Shaped`）、第 9 章の `FromText` のようなドメイン変換がこれにあたります。

- 一貫性（コヒーレンス）により、プログラム全体で `impl Type: Trait` はただ 1 つに決まります（`[trait.impl]`）。実装を引き回す必要も、2 つのモジュールが同じ型に対して別の順序を選んでしまう心配もありません。
- すべての呼び出しはコンパイル時にその `impl` へ解決され、実行時の間接呼び出しは起きません。
- トレイト関数はデフォルトで純粋です（`[trait.effects]`）。開いた接続を保持したり、時計を読んだり、本番とテストで振る舞いを差し替えたりはしません。

もし 1 つの型に対して同じトレイトの 2 通りの `impl` が欲しくなった場合（本番用とテスト用、あるいはファイル書き出し用とインメモリ用など）、そのインターフェースはトレイトではなく `effect` として定義すべきです。

## 振る舞いに対するインターフェースとしてのエフェクト

あるコンポーネントが外の世界に対して何を行うかをインターフェースとして記述し、それをどう実現するかから呼び出し側を切り離したい場合は `effect` を使います。

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

- `sign_up` は、`Accounts` が SQL データベースと通信するのかインメモリの `Map` なのか、あるいは `Notifier` がメールを送るのかコンソールに出力するのかを知りません。
- `sign_up` はハンドラも設定レコードも引数として受け取りません。シグネチャには必要なエフェクト（`uses { Accounts, Notifier }`）だけを書き、配線は境界（`layer App`、あるいは `test/` 以下のテスト用 `layer`）で行います。
- 各ハンドラ自身の要件（`State[Map[String, Account]]` や `Console`）は `sign_up` からは隠蔽され、ハンドラを `with` に置いた箇所でのみ表に現れます。

各エフェクトを 1 つの役割に絞っておく（`Accounts` と `Notifier` を分ける）ことで、どの関数の `uses` を見てもその関数が触れられる範囲が正確に分かり、テストでも他を触らずに 1 つのハンドラだけを差し替えられます。

## フィールドに保持したケイパビリティへ委譲する

キャッシュの追加、読み込み回数のカウント、呼び出しのログ記録、リクエストのリトライなど、あるハンドラが*同じ*エフェクトの別実装をラップしたいことがあります。

もし `handler CountingStore: Store` が自身の `get` の中で `store <- Store` と書くと、外側から名前で提供された `Store` を要件として要求することになります。そうする代わりに、ケイパビリティ（`use Store`）をフィールドに保持すれば、委譲を明示的かつ合成可能に書けます（`[effect.use]`、`[effect.handler]`）。

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

- `self.inner.get(key)` を呼んでも、`Store` は `requires(CountingStore)` に加わり*ません*。`init` の引数は `with CountingStore(inner: raw)` の外側で評価されるため、コンパイラは `raw` が `CountingStore` より長く生きる外側の `with` に属していることをすでに把握しています（`[effect.use]`、`[effect.handler]`）。
- `raw` は名前付き項目なので名前による提供を行わず、本体の中で `Store` を名前で提供する無名項目は `CountingStore` だけになります。したがって `load_pair` は確実にカウンタを通ります。ラベルも曖昧さもなく、いくつものラッパーを積み重ねられます（`with a = MemoryStore { with b = CachingStore(inner: a) { with CountingStore(inner: b) { ... } } }`）。
- 中断の追跡もそのまま保たれます。`self.inner` が `suspend fn` 操作を持つ場合、操作本体からそれを呼べば `Immediate` の判定に計上され、`release` から呼べば `requires(H)` に `Async` が加わります。

ハンドラが自身のライフタイムにわたってサブリソースを所有する場合（第 11 章で接続を借りた `RepoHandler` のように）は、`init` でハンドラメンバ（`Pool.Lease(...)`）を開いてケイパビリティをフィールドに格納します。この場合も、`self.conn` の操作を呼ぶことで余計な要件が増えることはなく、サブリソースはラッパー自身の `release` の後に自動で解放されます。

## 適切な道具を選ぶ

| 目的 | 道具 |
|---|---|
| 値の不変条件を守る、またはフィールドを隠す | `pub opaque type` / `pub opaque enum` |
| データ型ごとに 1 つの標準的な操作を静的に検査する | `trait` と `impl` |
| 差し替え可能な振る舞い、I/O、テスト用ハンドラ | `effect` と `handler`（`layer` と `with` で配線） |
| 同じエフェクトの 2 つの役割を名前で配線する | ラベル（`effect L of E`） |
| 2 つのインスタンスを明示的に渡す、またはあるハンドラを別のハンドラでラップする | 名前付き `with` 項目 + ケイパビリティパラメータまたはフィールド（`use E`） |
| 確定的に取得・解放されるスコープ付きリソース | `init` と `release` を持つハンドラ、またはハンドラメンバ |

## まとめ

- `opaque` 型はモジュール境界で不変条件を強制し、内部表現を自由に変更できるようにします。
- トレイトはデータ型を抽象化し、`(Type, Trait)` ごとにプログラム全体で一貫し、静的に解決される 1 つの `impl` を提供します。
- エフェクトとハンドラは振る舞いと環境を抽象化します。`layer` がプログラムの境界でそれらを配線するため、ビジネスロジックは自身が名指しするエフェクトにしか依存しません。
- ハンドラは `init` で `inner: use E` を受け取ってフィールドに格納することで、要件に `E` を増やすことなく、かつ `with` でライフタイムを検証された状態で同じエフェクトの別ハンドラをラップできます。

第 14 章では一歩引いて型・エフェクトシステム全体を見渡し、プログラムを実行する前に Emela の型が排除するバグの範囲を整理します。
