# はじめに

Emela は、型付きエラーと型付きエフェクトを備えた関数型プログラミング言語です。JavaScript、WebAssembly、C にコンパイルでき、同じプログラムが 3 つすべてで同じ結果を返します。

この章では、Emela がどのような言語か、インストール方法、そして小さなプログラムがどのように見えるかを紹介します。本書の残りの章では、そのプログラムを構成する要素を 1 つずつ解説していきます。

## 3 つの柱

Emela は 3 つの事柄をシンプルに保ち、言語のすべての機能はそのいずれかに属します（`[intro.pillars]`）。どれにも属さない機能は追加しません。

- **データの定義**（`type`、`enum`、`error`）。まずデータを、取り得るケースと各ケースが持つフィールドとして定義します。
- **データを扱う関数**（`fn`、`match`、パイプ `|>`）。関数は値をその形によって分解し、新しい値を組み立てます。
- **副作用の明示**（`effect`、`use`、`with`）。出力する、ファイルを読む、待機するといった外部世界に対する関数の振る舞いはその型の一部となり、それをどう実行するかは呼び出し側が決めます。

これらから次のいくつかの決定が導かれ、変わることはありません（`[intro.premises]`）。

- 値は決して変化せず、ループ構文もありません。繰り返しはコレクションに対する関数、または自分自身を呼び出す関数で書きます。
- null はありません。値が存在しない可能性がある場合は `Option` 型を使います。
- エラーはエフェクトではありません。関数のエラーはエフェクトとは別に型に含まれます。また、ゼロ除算のようなプログラム自身の誤りは第 3 のもの、すなわちどの型にも現れない*欠陥*（defect）です。
- 型間の暗黙の変換はなく、ユーザーが定義する演算子もありません。

## 2 つのレベル

Emela には 2 つのレベルがあります（`[intro.layers]`）。アプリケーションを書く人が使うのは言語のごく一部です。データ、関数、`match`、エラー、そして標準ライブラリが提供するエフェクトとハンドラです。ライブラリやプログラムの境界を書く人は、これらに加えて独自のエフェクトやハンドラ、トレイト、ホストの関数を宣言します。アプリケーションを書くために第 2 レベルが必要になることはありません。

本書も同じ構成をとります。第 I 部ではアプリケーションを書くために必要なすべてを扱います。第 II 部は第 I 部が土台としている基盤を扱います。第 III 部ではプログラムが動作するホストとターゲットを扱い、第 IV 部はツールのリファレンスです。

## Emela のインストール

macOS（Apple Silicon）、Linux（x86_64）、Windows（x86_64）向けにビルド済みバイナリが公開されています。macOS と Linux では次のようにインストールします。

```console
$ curl -fsSL https://emela.dev/install.sh | sh
$ emela --version
```

これにより `emela` が `$HOME/.emela/bin` にインストールされます。Windows では、リポジトリ `github.com/emela-lang/emela` のリリースから `emela-<version>-x86_64-pc-windows-msvc.zip` をダウンロードし、`emela.exe` を `PATH` の通った場所に配置してください。

プログラムを実行するには、ターゲットに応じたホストが必要です。

| ターゲット | 必要なもの |
|---|---|
| JavaScript（デフォルト） | Node.js 22 以降、または Bun 1.4.3 以降 |
| WebAssembly（`--target wasm`） | Wasmtime 49 以降 |
| ネイティブ（`--target native`） | `cc` として使える C11 コンパイラ |

`emela doctor` を実行すると、これらのうちどれがインストールされているかを確認できます。ネイティブターゲットはまだ Windows では動作しません。

リポジトリの `editors/` 以下には、いくつかのエディタ向けのシンタックスハイライトや言語サーバーの設定も用意されています。言語サーバーはすべての関数について推論された型・エラー・エフェクトを表示してくれるため、第 1 章から役立ちます。

## 最初のプログラム

カフェの注文金額を計算する小さなプログラムを見てみましょう。これは `emela new cafe` で作成したプロジェクトにおける `src/main.emel` の全体です。

```emel
import Std.Console.{Console, TestConsole}
import Std.Platform.{Platform}
import Std.State.{State}

# データ：閉じたケースの集合で、それぞれが独自のフィールドを持つ。
enum Item {
  Coffee(shots: Int)
  Tea
  Cake(slices: Int)
}

# エラーもデータと同じように宣言し、関数の型の一部になる。
error UnknownItem(name: String) derive Show

fn parse(name: String) -> Item fails UnknownItem {
  name match {
    "espresso" -> Coffee(shots: 1)
    "doppio"   -> Coffee(shots: 2)
    "tea"      -> Tea
    "cake"     -> Cake(slices: 1)
    _          -> fail UnknownItem(name:)
  }
}

fn price(item: Item) -> Int {
  item match {
    Coffee(shots:) -> 250 + 100 * shots
    Tea            -> 300
    Cake(slices:)  -> 450 * slices
  }
}

# 出力はエフェクト：型に含まれ、外側から提供される。
fn receipt(order: List[String]) -> () fails UnknownItem uses { Console } {
  console <- Console
  total = order |> List.map(parse) |> List.map(price) |> List.fold(from: 0, f: |sum, p| sum + p)
  console.print_line("#{List.length(order)} items: #{total} yen")
}

fn main() {
  with Platform {
    receipt(["doppio", "cake", "tea"])
    console <- Console
    receipt(["espresso", "scone"]) escape {
      UnknownItem(name:) -> console.print_line("sorry, no #{name}")
    }
  }
}

# テストは、出力内容を記録するハンドラを使って同じコードを実行する。
@test
fn prints_the_total() {
  (_, lines) = State.run([], || {
    with TestConsole {
      receipt(["tea", "tea"])
    }
  })
  assert lines == ["2 items: 600 yen"]
}
```

```console
$ emela run
3 items: 1200 yen
sorry, no scone
$ emela test --reporter plain
running 1 test
test prints_the_total ... ok

test result: ok. 1 passed; 0 failed; 0 filtered out
```

`--reporter plain` は、本書で掲載する行単位のテスト結果フォーマットを選びます（第 8 章）。

上から順に読むと、このプログラムは第 I 部の大半を使っています。

- `Item` は `enum` です。商品は、ショット数を持つコーヒー、紅茶、スライス数を持つケーキのいずれかであり、それ以外はありません（第 1 章）。
- `parse` と `price` は `match` で値を分解します。`match` はすべてのケースを処理しなければなりません。新しい種類の商品を追加した場合、`price` がそれを処理するまでコンパイルは通りません（第 2 章）。
- `receipt` はパイプ `|>` で処理をつなぎ、`List.map` に関数を渡し、ループの代わりに `List.fold` で金額を合計します（第 3 章、第 4 章）。
- `parse` は `UnknownItem` で失敗する可能性があり、それを型に書いています。`receipt` はそのエラーを処理しないため、エラーは `List.map` を通り抜けて呼び出し元へ伝わり、`receipt` も `fails UnknownItem` と書きます。`main` はこれを `escape` で処理します（第 5 章）。
- `receipt` は出力を行うため、型に `uses { Console }` と書きます。どのように出力するかは自分では決めません。`main` は `with Platform` でプラットフォームのハンドラを提供し、テストは代わりに行をリストに集める `TestConsole` を提供します（第 6 章、第 8 章）。

同じプログラムが変更なしで WebAssembly 上でも動作します。

```console
$ emela run --target wasm
3 items: 1200 yen
sorry, no scone
```

`emela run --target native` は C コンパイラで実行ファイルをビルドします。仕様は 3 つのターゲットすべてで同じ結果になることを定めています。その範囲と、現時点でターゲット間に残っている違いについては第 15 章で説明します。

## 本書の読み方

各章は順に積み上がる構成になっています。他の言語からいくつかの概念をすでに知っている場合は、[README](README.md) の読書ガイドでどこから読み始め、どこを飛ばせるかを確認できます。本書では各章の中で他言語との比較は行いません。比較は付録 A に言語ごとにまとめています。

本書は `spec/` の仕様が定める言語を解説します。ルールが重要になる箇所には `[intro.pillars]` のように角括弧でセクション名を示していますので、正確な文言を仕様で確認できます。本書と仕様が食い違う場合は仕様が正しいものです。

特記がない限り、記載する出力は JS ターゲットのものです。`emela run` は最初に実行コマンド（`running` で始まる行）を標準エラーに出力しますが、本書ではその行を省略しています。

第 1 章ではデータから始めます。
