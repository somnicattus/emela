# 5. エラー

操作によっては、必ず結果を返せるとは限りません。ユーザーが存在しないかもしれませんし、文字列が数値ではないかもしれません。Emela はこのような結果を*エラー*と呼びます。エラーは関数が提供する約束の一部であり、関数の型に現れ、すべての呼び出し側がそれを対処していることをコンパイラが検査します。また、エラーを対処するコードは結果を計算するコードとは分けて書かれます。

ゼロ除算のようなプログラム自身の誤りは、この意味でのエラーではありません。それはどの型にも現れずプログラムを停止させる*欠陥*（defect）です。この章では両者を扱い、ある状況でどちらを選ぶべきかも解説します。

道具は 2 つあります。`fail` でエラーを発生させ、`escape` でエラーを処理します。その間には何も書く必要がありません。関数が処理しないエラーは、そのまま呼び出し元へ伝わります。

## エラーを宣言する

エラーは型と同じ形で `error` を使って宣言します。

```emel
error NotFound(id: Int) derive Show

error Banned(name: String) derive Show

type User(id: Int, name: String, banned: Bool)

const USERS = [User(id: 1, name: "Ada", banned: False), User(id: 2, name: "Bob", banned: True)]

fn find_user(id: Int) -> User fails NotFound {
  USERS |> List.find(|u| u.id == id) |> Option.or_fail(NotFound(id:))
}

fn active_user(id: Int) -> User fails NotFound | Banned {
  user = find_user(id)
  if user.banned { fail Banned(name: user.name) }
  user
}
```

- `error NotFound(id: Int)` は、`type` と同じく型とコンストラクタを宣言します。`NotFound(id: 1)` は他の値と同じように束縛したり渡したり格納したりできる値です。
- フィールドを持たないエラーも宣言できます（`error Unauthorized`）。
- エラーは他のエラーと名前、すなわち*タグ*によって区別されます。`NotFound(id: 1)` と `NotFound(id: 2)` は中身が異なるだけで同じエラーです。
- `derive Show` を付けるとエラーを表示できるようになり、エラーが `main` に到達したときに役立ちます（後述）。

### エラーを発生させる

`fail` はエラーをもって計算を終了します。`active_user` では、`fail Banned(name: user.name)` によって関数が終了し、呼び出し元は結果の代わりにそのエラーを受け取ります。

`fail ...` の型は第 1 章で紹介した値を持たない型 `Never` です。この式が値を返すことは決してないため、どの型の値が期待される場所にも収まります。`if user.banned { fail ... }` に `else` が不要なのも、一方の分岐が値を返す `if` の他方の分岐に `fail` を置けるのもそのためです。

`Option.or_fail(opt, err)` は `Option` からの橋渡しをする関数で、`Some` のときは中身を返し、`None` のときは `err` で失敗します。

### 関数のエラー

関数のエラーはその型の一部です。`find_user` は `-> User fails NotFound` と書かれており、`User` を返すか `NotFound` で失敗することを表します。エラーは `fails NotFound | Banned` のように `|` で区切ったタグの集合であり、順序は問いません。

`active_user` は `find_user` を呼び出していますが、`NotFound` に対して何もしていません。エラーは自動的に `active_user` の呼び出し元へ伝わり、`NotFound` は `active_user` の `fails` にも含まれます。呼び出し箇所に特別な演算子を書く必要はありません。どの呼び出しもエラーをそのまま伝播させます。

エラーがどの程度厳密に検査されるかはシグネチャによって決まります（`[error.fails]`）。

| 記述 | 意味 |
|---|---|
| 戻り値の型なし | エラーは戻り値の型と一緒に推論される |
| `-> T` | 関数は失敗しない。処理されないエラーがあるとコンパイルエラー |
| `-> T fails A \| B` | 関数はちょうどこれらのエラーで失敗する |

したがって、`active_user` に `fails` を付けず `-> User` とだけ書くとコンパイルエラーになります。

```console
$ emela check
error[E3201]: unhandled error `NotFound`
    ╭─[ src/main.emel:14:10 ]
    │
 14 │   user = find_user(id)
    │          ──────┬──────
    │                ╰────────
────╯
   = help: handle it with `escape`, or declare it with `fails`

error[E3201]: unhandled error `Banned`
    ╭─[ src/main.emel:15:20 ]
    │
 15 │   if user.banned { fail Banned(name: user.name) }
    │                    ──────────────┬─────────────
    │                                  ╰───────────────
────╯
   = help: handle it with `escape`, or declare it with `fails`

2 errors
```

シグネチャが許可していないエラーごとに、コンパイラはそれを発生させる最初の式を指し示します。

## エラーを処理する

`escape` は直前の式のエラーを処理します。式の後ろに、`match` と同じ形でアームを並べて書きます。以下の関数は先ほどの宣言を使います。

```emel
error Forbidden(reason: String) derive Show

fn describe(id: Int) -> String {
  active_user(id).name escape {
    NotFound(_)   -> "nobody"
    Banned(name:) -> "#{name} (banned)"
  }
}

fn name_or_guest(id: Int) -> String fails Banned {
  active_user(id).name escape {
    NotFound(_) -> "guest"
  }
}

fn checked_name(id: Int) -> String fails NotFound | Forbidden {
  active_user(id).name escape {
    Banned(name:) -> fail Forbidden(reason: "#{name} is banned")
  }
}

fn main() {
  dbg(describe(1))
  dbg(describe(2))
  dbg(describe(3))
  dbg(name_or_guest(3))
  dbg(checked_name(1))
  refused = checked_name(2) escape {
    _ -> "refused"
  }
  dbg(refused)
  ()
}
```

```console
$ emela run
"Ada"
"Bob (banned)"
"nobody"
"guest"
"Ada"
"refused"
```

式が正常に値を返したときはその値がそのまま通過し、アームは使われません。失敗したときは、そのエラーに対応するアームが代わりに値を与えます。各関数がそれぞれの使い方を示しています。

- `describe` はすべてのエラーから回復します。アームが `NotFound` と `Banned` の両方を処理しているため `describe` は失敗せず、シグネチャの `-> String` がそれを表しています。
- `name_or_guest` は `NotFound` だけを処理します。`escape` はすべてのエラーを処理する必要はありません。残りのエラーは呼び出し元へ伝わり、`Banned` が `fails` に残ります。
- `checked_name` はあるエラーを別のエラーに変換します。アームの中でも `fail` でき、そのエラーが関数のエラー集合に加わります。
- `_` はすべてのエラーを処理します。`refused` のように、エラーの内容自体が重要でないときに使います。

`escape` のアームはエラーをコンストラクタ（フィールドに対するパターン付き）または `_` でマッチします。エラー自体を変数に束縛することはできません。1 つの `escape` が処理するエラーは互いに異なる型になり得るため、「`NotFound` か `Banned` のどちらか」を表す型が存在しないからです。`NotFound(id: 1)` のようにあるエラーの一部の値だけにマッチするアームはそれらの値だけを処理し、他の値は引き続き伝播するため、そのエラーは関数のエラー集合に残ります。式が発生させ得ないエラーに対するアームを書くと警告（W3201）が出ます。

アームは元の式と同じ型の値を返すか、あるいは失敗します。そのためエラーを情報として保持したい場合は、先に式の値を変換します。たとえば `Some(find_user(id)) escape { NotFound(_) -> None }` とするとエラーを `Option` に変換できます。

### 引数として渡す関数のエラー

関数を受け取る関数は、その関数のエラーをそのまま伝播させます（第 3 章）。

```emel
error Negative(n: Int) derive Show

fn check(n: Int) -> Int fails Negative {
  if n < 0 { fail Negative(n:) }
  n
}

fn main() {
  dbg(List.map([1, 2], check))
  fixed = List.map([1, -2, 3], check) escape {
    Negative(n:) -> [n]
  }
  dbg(fixed)
  ()
}
```

```console
$ emela run
[1, 2]
[-2]
```

`List.map(xs, check)` は `check` が失敗したときに `Negative` で失敗し、その要素で処理を止めます。失敗しない関数だけを受け取りたい関数は、パラメータの型に `f: fn(Int) -> Int fails {}` と書きます。そこに `check` を渡すとコンパイルエラーになります。

## Option かエラーか

`Option` もエラーも、結果が得られない可能性があることを表します。標準ライブラリは次のように使い分けており、自分の関数でも同じ基準が使えます（`[lib.api-rules]`）。

- 検索しても見つからない場合や、位置が範囲外になり得る場合は `Option` を返します（`List.get`、`List.find`）。結果がないことは通常の答えの 1 つであり、多くの場合、呼び出し側がその場で次に何をするか判断します。
- 入力の形式が不正な場合はエラーで失敗します（`Int.parse(s)` は `ParseError` で失敗します）。呼び出し側が結果の代わりになる妥当な値を持っていることはめったにありません。
- ファイルが読めないなど、プログラムの外側で問題が起きた場合は、エフェクトの操作がエラーで失敗します（第 6 章）。
- 呼び出し側が守るべき前提条件を破った場合は、欠陥にします。

どちらの方向にも短く変換できます。`Option` からエラーへは `Option.or_fail(opt, err)`、エラーから `Option` へは `Some(f(x)) escape { _ -> None }` と書きます。

## エラーが main に到達したとき

どの関数にも処理されなかったエラーが `main` に到達すると、プログラムは終了します。

```emel
error NotFound(id: Int) derive Show

fn find(id: Int) -> String fails NotFound {
  if id == 1 { "Ada" } else { fail NotFound(id:) }
}

fn main() {
  dbg(find(1))
  dbg(find(9))
  ()
}
```

```console
$ emela run
"Ada"
error: NotFound(id: 9)
$ echo $?
1
```

`error: ...` の行が標準エラーに出力され、終了コードは 1 になります。エラーは `Show` を使って表示されます。`Show` を持たないエラーは `error: NotFound` のように名前だけが表示されます。

## 欠陥

欠陥（defect）は、プログラムが回復することを期待できない誤りです。欠陥はどの型にも現れず、`escape` でも処理できません。

```emel
fn average(xs: List[Int]) -> Int {
  List.fold(xs, from: 0, f: |sum, x| sum + x) / List.length(xs)
}

fn main() {
  dbg(average([1, 2, 3]))
  dbg(average([]))
  ()
}
```

```console
$ emela run
2
defect: division by zero
$ echo $?
101
```

欠陥が発生するのは次の場合です。

- コード上あり得ない状況に対する `panic(message)` や、未実装のコードに対する `todo()`。
- 整数のゼロ除算、および `NaN` を `Int` に変換するなど、受け入れ範囲外の入力を渡された一部の操作。
- テスト内で失敗した `assert`（第 8 章）。
- ホストの関数が投げた例外（第 16 章）。

欠陥がプログラムのトップに到達すると、標準エラーに `defect: ...` を出力し、終了コード 101 で終了するため、エラー（終了コード 1）と区別できます。終了する前にも、プログラムが保持しているリソースの解放処理は実行されます（第 11 章）。

エラーを欠陥に変えるには、アームの中で `panic` を使います（`load_config(path) escape { _ -> panic("the bundled config must load") }`）。通常のコードの中で欠陥をエラーに戻すことはできません。欠陥を観測できるのは境界だけです。すなわちプログラムのトップ、テストランナー、あるいは `Exit.attempt`（第 12 章）です。

`average([])` が失敗するのは、リストが空かどうかを確認せずに長さで割っているからです。これを欠陥にすべきかエラーにすべきかは `average` の設計判断です。空リストが呼び出し側の誤りなら欠陥が適切ですし、想定すべき入力なら `average` が検査して `Option` を返すか独自のエラーで失敗すべきです。

## まとめ

- エラーは `error` で宣言し、`fail` で発生させ、`escape` で処理します。処理されないエラーは呼び出し箇所に何も書かなくても呼び出し元へ伝わります。
- 関数のエラーは `fails A | B` として型に含まれます。戻り値の型を書いて `fails` を書かない関数は失敗しません。
- `escape` は式のエラーをすべて処理することも、一部だけ処理することも、別のエラーや値に変換することもできます。
- エラーはタグによって区別されます。アームはタグまたは `_` にマッチします。
- 値が存在しない可能性がある場合は `Option`、不正な入力にはエラー、プログラムの誤りには欠陥を使います。
- `main` に到達したエラーは終了コード 1 で、欠陥は終了コード 101 でプログラムを終了します。

第 6 章では、関数の型に含まれるもう 1 つの要素である「エフェクト」を扱います。
