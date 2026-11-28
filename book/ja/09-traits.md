# 9. トレイト

第 1 章から登場している `derive Eq, Ord, Show` は、`==` や `<` による値の比較や、`#{...}` での値の整形を可能にしてきました。これら 3 つの名前は*トレイト*（trait）です。トレイトは型が提供できる一連の関数に名前を付け、ジェネリック関数が型パラメータに対してそれらの関数を要求できるようにします（`[trait]`）。

トレイトは型に属する振る舞いを表します。2 つの値をどう比較するか、値をどうテキストに変換するか、JSON からどうデコードするかといった振る舞いです。出力やデータベースとの通信のように外側の世界に依存する振る舞いは、トレイトではなくエフェクトです（第 10 章）。

## トレイトを宣言する

`trait` は、実装する型が提供する関数のシグネチャを列挙します（`[trait.decl]`）。

```emel
trait Describe {
  fn describe(self) -> String
}

type User(id: Int, name: String)

enum Role {
  Admin
  Member(team: String)
}

impl User: Describe {
  fn describe(self) {
    "#{self.name} (##{self.id})"
  }
}

impl Role: Describe {
  fn describe(self) {
    self match {
      Admin         -> "admin"
      Member(team:) -> "member of #{team}"
    }
  }
}

fn announce[A: Describe](value: A) -> String {
  "-> #{value.describe()}"
}

fn main() {
  ada = User(id: 1, name: "Ada")
  dbg(Describe.describe(ada))
  dbg(ada.describe())
  dbg(announce(ada))
  dbg(announce(Member(team: "core")))
  ()
}
```

```console
$ emela run
"Ada (#1)"
"Ada (#1)"
"-> Ada (#1)"
"-> member of core"
```

- `self` は最初のパラメータであり、その型は実装対象の型を表す `Self` です。パラメータとして `self` を取れるのはトレイトと `impl` の関数だけです（ハンドラも本体の中で `self` と `Self` を使います。第 10 章）。
- 戻り値の型を省略したトレイト関数は `()` を返します。
- `self` を省略し、戻り値の型だけで `Self` を使う関数も書けます（後述）。

### スーパートレイトとデフォルト実装

トレイトはコロンの後ろに別のトレイト（*スーパートレイト*）を要求できます（`trait A: B`、複数の場合は `trait A: B + C`）。また、トレイト内の関数に本体を書くこともでき、これは `impl` で省略できる*デフォルト実装*になります。

```emel
trait Priority: Eq {
  fn compare_priority(self, other: Self) -> Ordering

  fn higher(self, other: Self) -> Self {
    Priority.compare_priority(self, other) match {
      Less -> other
      _    -> self
    }
  }
}

enum Ticket {
  Low
  Normal
  Urgent
}
  derive Eq

fn rank(ticket: Ticket) -> Int {
  ticket match {
    Low    -> 0
    Normal -> 1
    Urgent -> 2
  }
}

impl Ticket: Priority {
  fn compare_priority(self, other) {
    Ord.compare(rank(self), rank(other))
  }
}

fn main() {
  first = Normal
  second = Urgent
  dbg(first.higher(second))
  dbg(Priority.higher(Urgent, Low))
  ()
}
```

```console
$ emela run
Urgent
Urgent
```

`Priority` は `Eq` をスーパートレイトに持つため、`impl Ticket: Priority` を書いた時点で `Ticket` が `Eq` を実装しているかが検査され、実装していなければコンパイルエラーになります。`Ticket` は `compare_priority` だけを実装し、`higher` はそのまま引き継ぎます。スーパートレイトで循環を作ることはできません（`trait A: B` と `trait B: A` はコンパイルできません）。

### トレイト関数のエラーとエフェクト

トレイト関数はデフォルトで純粋です（`[trait.effects]`）。トレイト関数が失敗し得る場合やエフェクトを使う場合は、宣言に `fails` や `uses` を書きます。

```emel
error BadFormat(input: String) derive Show

trait FromText {
  fn from_text(input: String) -> Self fails BadFormat
}

type Port(value: Int)

impl Port: FromText {
  fn from_text(input) {
    n = Int.parse(input) escape {
      _ -> fail BadFormat(input:)
    }
    if n < 1 || n > 65535 { fail BadFormat(input:) }
    Port(value: n)
  }
}

fn main() {
  port: Port = FromText.from_text("8080")
  dbg(port)
  ()
}
```

```console
$ emela run
Port(value: 8080)
```

実装が発生させ得るエラーや使用できるエフェクトは、トレイトが宣言した範囲内に限られます。そのため、`[A: FromText]` を持つジェネリック関数は、`A` が具体的にどの型になるかに関わらず、トレイト宣言だけから自身のエラーとエフェクトを把握できます。

## トレイトを実装する

実装は `impl Type: Trait` と書きます（`[trait.impl]`）。コロンの意味は `[A: Ord]` や `trait Ord: Eq` と同じで、左側が右側を満たすことを表します。

- パラメータの型と戻り値の型はトレイト宣言で固定されているため、`impl` では省略します。デフォルト実装のない関数はすべて書く必要があります。
- 型パラメータを持つ型に対する `impl` では、ヘッドに型パラメータの境界を書きます。

```emel
type Pair[A, B](first: A, second: B)

impl Pair[A: Eq, B: Eq]: Eq {
  fn eq(self, other) {
    self.first == other.first && self.second == other.second
  }
}

impl Pair[A: Show, B: Show]: Show {
  fn show(self) {
    "(#{self.first}, #{self.second})"
  }
}

fn largest[A: Ord](xs: List[A]) -> Option[A] {
  xs match {
    [] -> None
    [first, ..rest] -> Some(
      List.fold(rest, from: first, f: |best, x| if x > best { x } else { best }),
    )
  }
}

fn same_largest[A: Ord](xs: List[A], ys: List[A]) -> Bool {
  largest(xs) == largest(ys)
}

fn main() {
  p = Pair(first: 1, second: "one")
  dbg(p == Pair(first: 1, second: "one"))
  dbg("pair: #{p}")
  dbg(same_largest([3, 1, 4], [4, 2]))
  ()
}
```

```console
$ emela run
True
"pair: (1, one)"
True
```

3 つのルールによって、どの呼び出しがどの実装に到達するかに曖昧さが一切生じないことが保証されています。

- `impl` のヘッドに書けるのは型名と互いに異なる型パラメータだけです。`impl List[Int]: Show` や `impl Pair[A, A]: Eq` はコンパイルできず、プログラム内のどこであっても同じ `(Type, Trait)` の組に対する `impl` を 2 つ書くとコンパイルエラーになります。実装どうしが重複することはありません。
- `impl` を書けるのは、その型を定義したモジュールか、そのトレイトを定義したモジュールだけです。どちらも自分のモジュールでない場合は、1 フィールドの独自の `type` で型をラップし（実行時コストはありません）、そのラッパーに対してトレイトを実装します。

  ```emel
  type MyDate(inner: ThirdParty.Date)

  impl MyDate: Encode { ... }
  ```

- スーパートレイトを持つトレイトを `impl` で実装する場合、スーパートレイト側の `impl` が要求する境界がヘッドに書いた境界から導かれなければなりません。`impl Pair[A: Ord, B: Ord]: Ord` は、`A: Ord` と `B: Ord` から `A: Eq` と `B: Eq` が導かれるため受け入れられます。

## 呼び出しと境界

トレイト関数は 3 通りの方法で呼び出せます（`[trait.call]`）。

- トレイト名で修飾する：`Describe.describe(ada)` または `ada |> Describe.describe`。
- メソッド呼び出しとして呼ぶ：`ada.describe()`。関数が `self` を取り、呼び出し元のモジュールでそのトレイトが名前で可視である場合に使えます（Prelude のトレイト、同じモジュールで定義されたトレイト、または波括弧でインポートされたトレイト。`[type.rules]`）。`[A: Describe]` のような境界付き型パラメータを型に持つ値に対しては、`announce` の `value.describe()` のように境界の関数が直接見つかります。
- `FromText.from_text("8080")` のように `self` を持たず `Self` が戻り値の型だけに現れる関数では、コンパイラは期待される戻り値の型から `impl` を選びます。その型は型注釈（`port: Port = ...`）や結果の使い方（`FromText.from_text(s) + 1` は `Int` を選ぶ）から決まります。関数内のどこからも型が決まらない場合、呼び出しはコンパイルできません。

```emel
error BadFormat(input: String) derive Show

trait FromText {
  fn from_text(input: String) -> Self fails BadFormat
}

fn main() {
  x = FromText.from_text("8080")
  dbg(x)
  ()
}
```

```console
$ emela check
error[E3307]: cannot infer which impl of `FromText` to use: add a type annotation
   ╭─[ src/main.emel:8:7 ]
   │
 8 │   x = FromText.from_text("8080")
   │       ─────────┬────────
   │                ╰──────────
───╯

1 error
```

ジェネリック関数は `[A: Ord]`、複数のトレイトなら `[A: Ord + Show]` のように型パラメータを制約します。境界にはそのスーパートレイトも含まれます。先ほどの `same_largest` は `[A: Ord]` しか書いていませんが、`Ord` が `Eq` を含意するため `Option[A]` に `==` を使えています。境界は関数を参照するすべての箇所で検査され、呼び出さずに値として渡す場合も例外ではありません。

## 組み込みトレイト

コンパイラには 5 つのトレイトが組み込まれています（`[trait.builtin]`）。

| トレイト | 使用箇所 | ユーザーによる `impl` |
|---|---|---|
| `Eq` | `==`、`!=` | 可能 |
| `Ord: Eq` | `<`、`<=`、`>`、`>=` | 可能 |
| `Show` | `#{...}`、`inspect` | 可能 |
| `Immediate` | 中断しないエフェクト集合（第 12 章） | コンパイラが決定 |
| `Capability` | ケイパビリティ型 `use E`（第 10 章） | コンパイラが決定 |

`Immediate` や `Capability` に対して `impl` を書くとコンパイルエラーになります。

### `Ord` と `Float`

`Ord.compare` は、`Equal` が `==` と一致する全順序を約束します。`Float` には自分自身とすら等しくない `NaN` があるためこの約束を守れず、`Float` は `Eq` と `Show` は実装しますが `Ord` は実装しません。

`Float` に直接適用する `<`、`<=`、`>`、`>=` 演算子は組み込みの IEEE 754 比較です（`NaN` との比較はすべて `False` になります）。ジェネリック関数内の比較は `Ord` を通るため、`List[Float]` を `largest` に渡すとコンパイルエラーになります。

```console
$ emela check
error[E3302]: type `Float` does not implement `Ord`
    ╭─[ src/main.emel:11:7 ]
    │
 11 │   dbg(largest([1.5, 2.5]))
    │       ───┬───
    │          ╰─────
────╯

1 error
```

浮動小数点数に全順序が必要なときは `Float.compare` を明示的に渡します。`Float.compare` は `NaN` を最大として扱い、2 つの `NaN` どうし、および `0.0` と `-0.0` を `Equal` として扱います。

```emel
fn main() {
  dbg(List.sort_by([3.5, 1.2, 2.8], Float.compare))
  ()
}
```

```console
$ emela run
[1.2, 2.8, 3.5]
```

### `Show`、`inspect`、`dbg`

自動導出された `Show` は、ソースコードで値を構築するときの形式で値を整形します（`User(id: 1, name: "Ada")`）。文字列補間では、トップレベルの `String` は引用符なしで埋め込まれ、リストやコンストラクタの中の `String` は引用符付きで表示されます。`inspect[A: Show](value: A) -> String` は `Show` を要求し、トップレベルの `String` にも引用符を付けます。

```emel
type Label(text: String) derive Show

fn main() {
  text = "hello"
  dbg("#{text}")
  dbg(inspect(text))
  dbg("#{Label(text:)}")
  ()
}
```

```console
$ emela run
"hello"
"\"hello\""
"Label(text: \"hello\")"
```

`dbg` は `Show` を要求せず、任意の型の値を表示します。`Show` を持たない型、`opaque` 型（中身を表示）、関数（`<fn>`）、ホストオブジェクト（`<external Name>`）も含まれます。出力先は標準エラーのみで、`String` として取得することはできません。失敗した `assert` の表示（第 8 章）や `main` に到達したエラーの表示（第 5 章）も `Show` を必須とはしませんが、`Show` がある場合はより詳しい情報が表示されます。関数型は `Eq`、`Ord`、`Show` のいずれも満たしません。

## `derive` の詳細

`derive` は、型のフィールドやバリアントから通常の `impl` を生成するようコンパイラに指示します（`[trait.derive]`）。

- 各フィールドの処理は、そのフィールド自身の型の `impl` を通じて行われます。たとえば `User` が手書きの `Show` を持つ独自の `Email` を保持している場合、`User` の自動導出された `Show` はその `Show` を呼び出します。いずれかのフィールドの型が対象トレイトを実装していなければ `derive` はコンパイルできません（そのため `Float` フィールドを持つ型は `Ord` を derive できません）。
- 型パラメータを持つ型に対する `derive` は、たとえフィールドに現れないパラメータであっても、すべての型パラメータに境界を付けます（`enum Tree[A] { ... } derive Eq` は `impl Tree[A: Eq]: Eq` を生成します）。
- スーパートレイトは暗黙には derive されません。`derive Ord` を書くには、`Eq` も derive または実装されている必要があります。`derive Show` と手書きの `impl T: Show` を併用するとコンパイルエラーになります。
- ユーザーが宣言した型は、`derive` または `impl` を書かない限り `Eq`、`Ord`、`Show` のいずれも満たしません。暗黙に derive されるものは何もありません。
- 非公開の型において、プログラム内のどこからも要求されないトレイトを derive すると警告が出ます（例外は `error` に対する `Show` で、エラーが `main` やテストに到達したときに使われます）。

コンパイラは 7 つのトレイトについて導出ルールを持っています。Prelude の `Eq`・`Ord`・`Show`、`Std.Dynamic` の `Encode`・`Decode`・`ObjectKey`、そして `Std.Shape` の `Shaped` です。それ以外のトレイトを `derive` に書くとコンパイルエラーになります。ユーザー定義の導出ルールは将来のバージョンに持ち越されています。

シリアライズ用のトレイトは、フィールドの `@key` やバリアントの `@tag` と組み合わせてシリアライズ時の名前を制御します。

```emel
import Std.Dynamic.{Decode, Encode}
import Std.Json

type Point(@key("x_pos") x: Int, @key("y_pos") y: Int)
  derive Eq, Encode, Decode

fn main() {
  raw = Json.encode(Point(x: 3, y: 4))
  dbg(raw)
  decoded: Point = Json.decode(raw)
  dbg(decoded == Point(x: 3, y: 4))
  ()
}
```

```console
$ emela run
"{\"x_pos\":3,\"y_pos\":4}"
True
```

`derive` は、トレイトが値を読み取るものか構築するものかに応じて `opaque` 型（第 7 章）を次のように扱います。

- `Eq`、`Ord`、`Show`、`Encode`、`Shaped` は `opaque` 型にも derive できます。ただし自動導出された `Show` や `Encode` は隠蔽されたフィールドを表示してしまうため、中身を非公開のままにしたい場合は `impl` を手で書きます。
- `Decode` と `ObjectKey` は `opaque` 型に derive できません。自動導出されたデコーダは、モジュールの不変条件を検査する関数を迂回して値を直接構築してしまうからです。

## トレイトにない機能

Emela は意図的にトレイトを小さく保っています（`[trait.not]`）。関連型、高カインド型に対するトレイト、実行時の動的ディスパッチ、ユーザー定義の演算子トレイトはありません。トレイト関数の呼び出しはすべて、コンパイル時の型によって決まる単一の `impl` に解決されます。実行時に実装を差し替えたい場合や、インターフェースの背後にどの実装があるかを隠したい場合、Emela ではトレイトではなくエフェクトとハンドラを使います。

## まとめ

- `trait` は型が提供できる関数を宣言します。`Self` 型の `self`、任意のデフォルト実装、任意のスーパートレイト（`trait A: B`）を持てます。トレイト関数は `fails` や `uses` を書かない限り純粋です。
- `impl Type: Trait` は型のモジュールまたはトレイトのモジュールでトレイトを実装します。プログラム全体で `(Type, Trait)` の組ごとの `impl` は最大 1 つであり、ヘッドで型引数を特殊化することはできません。
- ジェネリック関数は `[A: Trait]` や `[A: T1 + T2]` で型パラメータを制約します。境界にはスーパートレイトが含まれます。戻り値の型にのみ `Self` が現れる関数は、期待される型から `impl` を選びます。
- `Eq`、`Ord`、`Show` は組み込みです。`Float` は `NaN` が全順序を破るため `Ord` を持ちません。`dbg` は `Show` を要求しません。
- `derive` は `Eq`、`Ord`、`Show`、`Encode`、`Decode`、`ObjectKey`、`Shaped` について通常の `impl` を生成します。`Decode` と `ObjectKey` は `opaque` 型に derive できません。

第 10 章では抽象化のもう一方の側、すなわち独自のエフェクト・ハンドラ・レイヤ・ラベル・ケイパビリティの定義に進みます。
