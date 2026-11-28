# 7. モジュールとパッケージ

ここまでの例は 1 ファイルに収まっていました。実際のプログラムは目的ごとにモジュールへ分割し、他の人が公開したコードも利用します。この章ではその両方を扱います。ファイルがどのようにモジュールになるか、モジュールが外から見えるものをどう制御するか、そしてプロジェクトが他のプロジェクトにどう依存するかを見ていきます。

## 1 ファイル、1 モジュール

1 つのファイルが 1 つのモジュールです。モジュール名は `src/` 以下のパスから決まります。パスの各要素が `snake_case` から `PascalCase` に変換され、`.` で連結されます。

```text
shop/
├── Pome.toml
└── src/
    ├── main.emel               # エントリポイント `main`
    ├── email.emel              # モジュール Email
    └── billing/
        └── invoice.emel        # モジュール Billing.Invoice
```

ファイルの中に自分のモジュール名を書く宣言はなく、1 ファイルに 2 つのモジュールを入れることもできません。ファイル名とディレクトリ名は小文字で書きます。`json_codec.emel` は `JsonCodec` モジュールになります。2 つのモジュールは直接的にも間接的にも互いにインポートできません。

`src/email.emel`：

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

`src/billing/invoice.emel`：

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

`src/main.emel`：

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

`import Billing.Invoice` はそのモジュールを末尾の名前 `Invoice` で使えるようにし、その公開名は `Invoice.total(invoices)` のように前に付けて書きます。後ろに波括弧で列挙した名前は単独でも書けます。`import Billing.Invoice.{Invoice}` の後では、`Invoice` 型をプレフィックスなしで書け、さらに `Invoice.total` もそのまま使えます。

- モジュールのすべての名前を一括でインポートする構文はありません。プレフィックスなしで使う名前は 1 つずつ列挙します。
- エイリアス（別名）はありません。モジュールは常に名前の最後の部分で参照します。
- `import` はファイルの先頭、あらゆる宣言より前に書きます。
- モジュール、型、関数は別々の名前空間にあるため、`Email` モジュールと `Email` 型は衝突しません。どちらの意味になるかは書かれた場所によって決まります。`Email.parse` はモジュールであり、`to: Email` は型です。`import Email` だけではモジュールだけが可視になるため、型は `Email.Email` と書くか、`import Email.{Email}` のように名前付きでインポートします。
- `List`、`String`、`Option`、`Int` のようにコンパイラが直接知っている標準ライブラリのモジュールはインポート不要です。`Std` 以下のモジュールはインポートが必要です（第 6 章）。

メソッド呼び出し（第 3 章）にもインポートは不要です。`summary` の中の `invoice.to.to_string()` は、`invoice.to` の型である `Email` を定義しているモジュールから `to_string` を見つけます。

## pub

モジュールが宣言するものは、`pub` が付いていない限りそのモジュールに対して非公開（private）です。他のモジュールが `Invoice.format_cents` を呼び出すとコンパイルできません。

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

`pub` はトップレベルの宣言（関数、`type`、`enum`、`error`、`const`、および第 II 部のエフェクト・ハンドラ・レイヤ・トレイト）に付けます。公開型のフィールドや公開 enum のバリアントも一緒に公開されます。一部のフィールドや一部のバリアントだけを隠す方法はありません。モジュールがインポートした名前は再公開されません。`Billing.Invoice` をインポートしても、`Email` がインポート元に見えるようにはなりません。

公開関数はモジュールが他のモジュールに対して提供する約束の一部なので、戻り値の型・`fails`・`uses` まで含めてシグネチャをすべて書くのが一般的です。省略して推論させることもでき（第 2 章）、言語サーバーは推論されたシグネチャを書き込む機能を提供しています。

## opaque：内部表現を隠す

`pub opaque type Email(value: String)` は、型を公開しつつその中身を非公開にします。`Email` モジュールの外側からは、`Email` 型の値を構築したり、`match` で分解したり、フィールドを読み取ったりできません。`main` で `forged = Email.Email(value: "nobody")` のように書くとコンパイルエラーになります。

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

外側から `Email` を手に入れる唯一の方法は、まずテキストを検査する `Email.parse` です。したがってプログラムのどこにあっても、`Email` 型の値は `@` を含む文字列であることが保証されます。型そのものが検査済みの事実を運ぶため、`Email` を受け取る関数がもう一度検査する必要はありません。`Invoice` が `Email` を保持しているのはそのためです。

`opaque` は enum にも使えます（`pub opaque enum`）。値が守るべき不変条件があるときや、他のモジュールを壊さずに内部表現を自由に変更できるようにしたいときに使ってください。

## Pome：プロジェクトと依存関係

*Pome* とは、`Pome.toml` を持つディレクトリ、すなわちプロジェクトのことです。他のプロジェクトが依存する単位でもあります。`emela new shop` は `main` を持つプロジェクトを作成し、`emela new --lib parser` はルートモジュールが `src/lib.emel` のライブラリを作成します。

```toml
[pome]
name = "github.com/your-name/shop"
version = "0.1.0"
emela = "0.29.0"
```

- `name` はその Pome が置かれている場所、すなわち `https://` を除いた Git リポジトリのパスです。中央レジストリはなく、アカウントを作成する必要もありません。Pome は `v0.1.0` のような Git タグをプッシュすることで公開します。
- `version` はこの Pome のバージョンで、`emela` はこれをビルドできる最古のコンパイラバージョンです。

### Pome に依存する

`emela pome add` はリポジトリのパスを指定して依存関係を追加します。

```console
$ emela pome add --dev github:emela-lang/brix
```

これにより `Pome.toml` に依存関係が書き込まれ、必要なすべての正確なバージョンが `Pome.lock` に記録されます。

```toml
[dependencies]
"github.com/acme/money" = "v1.2.0"

[dev-dependencies]
"github.com/emela-lang/brix" = "v0.2.0"
```

- `Pome.toml` のバージョンは「このバージョン以降」を意味します。`Pome.lock` はプロジェクトが直接・間接に必要とするすべての Pome について、実際に使われるバージョンをコミットと内容のチェックサム付きで記録します。これは `emela pome` コマンドが書き込むもので、手で編集するものではありません。`Pome.lock` がないと `emela run` などのコマンドは停止して `emela pome install` を案内するため、プロジェクトと一緒に管理します。
- 依存関係はプロキシ経由で取得され、その内容がチェックサムデータベースと照合されるため、公開後にタグが付け替えられた場合も検知されます（`[module.pome]`）。
- `[dev-dependencies]` はテストでのみ使われます（第 8 章）。アサーションのヘルパー集である `brix` が最初に追加される定番の依存先です。
- `emela pome update`、`remove`、`list`、`verify` はそれぞれの名前のとおりに動作します（`[cmd.pome]`）。

依存先のモジュールはそのルート名の下に現れます。デフォルトではパスの最後の部分を `PascalCase` にした名前です。たとえば `github.com/acme/money` なら `Money` になり、その `src/lib.emel` は `import Money`、`src/convert.emel` は `import Money.Convert` でインポートします。プロジェクトから見えるのは自身が列挙した Pome だけで、依存先がさらに依存している Pome は見えません。

外部関数（第 16 章）を通じてホストを直接呼び出す Pome は、通常の Emela コードにはできないことを実行できます。そのような依存先はプロジェクトの `Pome.toml` の `[external]` で名前を指定して許可する必要があり、`emela pome add` は許可されていないものの追加を拒否します（`--allow-external` を付けると許可できます）。

### 自分の Pome のバージョン

ライブラリの新しいバージョンにタグを打つ前に、`emela pome publish --dry-run` を実行すると、公開関数や公開型を前のバージョンと比較します。公開されているものが削除または変更されている場合、バージョンをより大きく上げる必要があることを警告します（`[cmd.pome]`）。公開 enum にバリアントを追加する変更も、利用側の `match` がすべてのケースを網羅できなくなるため破壊的変更として扱われます。

## まとめ

- 1 ファイルが 1 モジュールになり、パスから名前が決まります（`src/billing/invoice.emel` は `Billing.Invoice`）。
- `import A.B` は `B` をプレフィックスとして使えるようにし、`import A.B.{x, Y}` はさらに `x` と `Y` を単独でも使えるようにします。ワイルドカードインポートもエイリアスもありません。
- 宣言は `pub` を付けない限り非公開です。`pub opaque` は型を公開しつつ中身を非公開にし、モジュールが値に関するルールを保証できるようにします。
- Pome は `Pome.toml` を持つプロジェクトです。Git リポジトリで識別され、Git タグで公開します。依存関係は `[dependencies]` と `[dev-dependencies]` に書き、`Pome.lock` がバージョンを固定します。

第 8 章ではテストを書きます。`@test` を付けた関数、アサーション、そしてテスト対象コードのエフェクトをテスト側から制御するハンドラを学びます。
