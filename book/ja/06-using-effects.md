# 6. エフェクトを使う

1 行出力したり、時計を読んだり、乱数を引いたりする関数は、結果を計算する以上のことをしています。プログラムの外側の世界に働きかけたり、外側の世界に依存したりしているのです。Emela では、出力する・時計を読むといった関数の能力を*エフェクト*と呼びます。関数が使うエフェクトはエラーと並んで型の一部となり、外側から提供されます。関数は「出力する必要がある」と宣言し、それをどう実行するかは呼び出し側のコードが決めます。

この後半こそがエフェクトを有用にする仕組みです。同じ関数であっても、本番実行時には端末へ出力し、テスト実行時にはリストへ記録する、といったことが関数に手を加えることも手動で何かを渡すこともなく実現できます。

この章では標準ライブラリのエフェクトを使います。独自のエフェクトを宣言する方法は第 10 章で説明します。

## 再び Hello, world

`emela new` が生成するプログラム（第 1 章）は、この章の内容を凝縮した小さな例です。

```emel
import Std.Console.{Console, SystemConsole}

fn greet(name: String) {
  console <- Console
  console.print_line("Hello, #{name}!")
}

fn main() {
  with SystemConsole {
    greet("Ada")
    greet("Bob")
  }
}
```

```console
$ emela run
Hello, Ada!
Hello, Bob!
```

- `import Std.Console.{Console, SystemConsole}` は `Std.Console` モジュールの 2 つの名前、エフェクト `Console` とハンドラ `SystemConsole` を可視にします。プラットフォームのエフェクトとハンドラは `Std` 以下のモジュールにあります（モジュールとインポートの詳細は第 7 章）。
- `greet` の中の `console <- Console` は、ブロックの残りにおいて `Console` エフェクトを `console` という名前で使うことを宣言します。この名前を*エフェクトエイリアス*と呼びます。`console.print_line(...)` はそのエフェクトの*操作*を呼び出します。
- `main` の中の `with SystemConsole { ... }` は、ブロック内で実行されるすべてのコードに `Console` を提供します。`SystemConsole` は*ハンドラ*であり、プロセスの標準出力へ書き込むことで `Console` の操作を実装します。

`greet` はどのハンドラを使うかを指定しませんし、指定することもできません。要求するのは `Console` だけです。どのハンドラを使うかは、呼び出しを囲む `with` が決めます。

## 関数の型に現れるエフェクト

関数が使うエフェクトは型の一部であり、`uses` の後に書きます。

```emel
import Std.Console.{Console}
import Std.Platform.{Platform}

fn report(scores: List[(String, Int)]) -> () uses { Console } {
  console <- Console
  List.each(scores, |(name, score)| console.print_line("#{name}: #{score}"))
}

fn main() {
  with Platform {
    report([("Ada", 90), ("Bob", 72)])
  }
}
```

```console
$ emela run
Ada: 90
Bob: 72
```

`greet` のように `uses { Console }` は省略できます。`uses` を書かない場合、エフェクトは常に推論され（第 2 章）、エディタ上に表示されます。`greet` や `report` を呼び出す関数も同様に `Console` を使うことになり、`with` がそれを提供するまで外側へ伝わっていきます。

型の中のエフェクトはエラーと同じく集合です（`uses { Console, Clock }`）。型にエフェクトを持たない関数は、デバッグ出力の `dbg`（第 1 章）を除き外の世界に働きかけることはなく、その結果は引数のみによって決まります。

`List.each` は関数の作用のために各要素へ関数を呼び出します（第 4 章）。ここの無名関数は出力を行うため、`List.each` の呼び出しも `Console` を使います。引数として渡した関数のエフェクトは、エラーと同じく呼び出し側へそのまま伝わります（第 3 章）。

### 誰も提供していないとき

`main` はすべてのエフェクトを提供しなければならない場所です。`with` なしで `greet` を呼び出す `main` を持つプログラムはコンパイルできません。

```console
$ emela check
error[E4301]: effect `Console` is not provided in `main`
   ╭─[ src/main.emel:9:3 ]
   │
 9 │   greet("Ada")
   │   ──────┬─────
   │         ╰───────
───╯
   = help: provide it with `with`

1 error
```

`main` がホストに委ねたままにできるエフェクトは、待機できることなどホスト自身が提供するものだけです（`[effect.with]`、第 12 章）。

## エフェクトを提供する

`with` は 1 つ以上の項目をカンマ区切りで並べ、それらのエフェクトをブロックに提供します。`with` の値はブロックの値になります。

標準ライブラリには、プラットフォームの各エフェクトに対して、エフェクト名の前に `System` を付けたハンドラ（`SystemConsole`、`SystemClock`、`SystemRandom` など）が用意されています（`[lib.layers]`）。これらを束ねた 2 つの*レイヤ*があります。

- `Platform`（`Std.Platform` 内）：どのホストにも存在するハンドラ（Console、Clock、Random、Http）をまとめたもの。
- `Process`（`Std.Process` 内）：`Platform` に加え、プロセスを必要とするハンドラ（引数、標準入出力、環境変数、ファイル、TCP）をまとめたもの。

そのため `main` は通常 `with Platform { ... }` と書き、ファイルや引数を扱うときは `with Process { ... }` と書きます。レイヤはハンドラと同じように `with` の中で使えます。

プラットフォームのエフェクトの操作は失敗することがあります。たとえば標準入力から 1 行読む操作は、入力が有効なテキストでないときに `InvalidInput` で失敗します。失敗し得る操作はエフェクト宣言の中で `fails` を持ち、その呼び出しは失敗し得る関数の呼び出しと同じように失敗します（第 5 章）。

### ハンドラを差し替える

関数が指定するのはエフェクトだけなので、どの呼び出しの周りでもハンドラを差し替えられます。1 つの `with` の中の複数項目が同じエフェクトを提供する場合、後ろに書いたものが優先されます。

```emel
import Std.Clock.{Clock}
import Std.Console.{Console, TestConsole}
import Std.Platform.{Platform}
import Std.State.{State}

fn countdown(from: Int) {
  clock <- Clock
  console <- Console
  List.each(List.range(0, from), |i| {
    console.print_line("#{from - i}")
    clock.sleep(100)
  })
  console.print_line("go")
}

fn main() {
  (_, lines) = State.run([], || {
    with Platform, TestConsole {
      countdown(3)
    }
  })
  dbg(lines)
  ()
}
```

```console
$ emela run
["3", "2", "1", "go"]
```

`with Platform, TestConsole` は、`Clock` などは `Platform` から、`Console` は `TestConsole` から提供します。`TestConsole` はどこにも出力せず、出力された各行を `State` エフェクトで保持するリストへ追加します。`State.run([], || ...)` は空リストを初期値として関数を実行し、関数の結果と最終的なリストを返します。`countdown` 自体は一切変更しておらず、各行の間では本物の時計で待機しています。

このようにして、エフェクトを使うコードとそれをどう実行するか決めるコードが分離されます。テストも同じ仕組みを使って、コンソールや時計やデータベースを自分で制御できるハンドラに差し替えます（第 8 章）。手動で何かを渡す必要はありません。ハンドラは `with` の中で呼び出されるあらゆる関数へ、どれほど深い階層であっても届きます。

## エフェクトと値としての関数

関数値は `fn() -> () uses { Console }` のように自身の型にエフェクトを持ちます。そしてエフェクトが提供される必要があるのは関数を作った場所ではなく、関数を呼び出す場所です。出力を行う無名関数はどこで作成しても問題なく、`Console` が必要になるのはそれを呼び出すときです。

パラメータの関数型に `uses {}` と書くと、エフェクトを持たない関数だけを受け入れます。

```emel
fn evaluate(f: fn(Int) -> Int uses {}, x: Int) -> Int {
  f(x)
}
```

出力を行う無名関数を `evaluate` に渡すとコンパイルエラーになります。`List.map` や `List.each` のようにパラメータの関数型で `uses` を省略すると、任意のエフェクトを受け入れて呼び出し側へそのまま伝えます（第 3 章）。

## 待機する

操作の中には待機するものがあります。`countdown` の中の `clock.sleep(100)` は 0.1 秒待ってから処理を進めます。`Clock` の宣言では、`sleep` は呼び出し元のコードを中断し得る操作、すなわち `suspend fn` です。

呼び出し箇所に印は何も付きません。`clock.sleep(100)` も `countdown` の呼び出しも、他のあらゆる呼び出しと同じように書きます。呼び出しが待機し得るかどうかは呼び出し先の型にあるエフェクトから決まるため、呼び出しごとに書き直す必要はありません。待機によって何が可能になるか――複数のタスクを同時に動かすこと――については第 12 章で解説します。

## まとめ

- エフェクトは、出力する・時計を読むといった、関数が要求する能力です。関数は `name <- Effect` でエフェクトを使い、`name.operation(...)` でその操作を呼び出します。
- 関数のエフェクトは `uses { A, B }` として型に含まれ、`uses` を省略すると推論されます。`with` で提供されるまで、すべての呼び出し元へ伝播します。
- `with Handler { ... }` はハンドラのエフェクトをブロックに提供します。`main` はすべてを提供し、通常は `Platform` または `Process` レイヤを使います。
- 関数はハンドラを名指ししないため、`with` の後ろに別の項目を置いて差し替えられます（たとえば `SystemConsole` の代わりに `TestConsole`）。
- 引数として渡した関数はそのエフェクトを呼び出し側へ伝えます。パラメータの型に `uses {}` と書くとエフェクトのない関数だけを受け入れます。
- 待機し得る呼び出しも他の呼び出しと同じように書きます。

第 7 章では、プログラムをモジュールに分割する方法と、他のパッケージのコードを使う方法を学びます。
