# 8. テスト

テストは `@test` を付けた関数であり、その中の検査は `assert` で書きます。エフェクトを使うコードは、第 6 章でコンソールを差し替えたのと同じ仕組みで、テスト側が制御するハンドラの下で実行してテストします。それ以外に特別なものは不要です。セットアップすべきフレームワークも、モックオブジェクトの仕組みもありません。

## テストを書く

この章の例は、打ち上げ前にカウントダウンする小さなプロジェクト `launch` です。

`src/countdown.emel`：

```emel
import Std.Clock.{Clock}
import Std.Console.{Console}

pub fn countdown(from: Int) -> () uses { Clock, Console } {
  clock <- Clock
  console <- Console
  List.each(List.range(0, from), |i| {
    console.print_line(label(from - i))
    clock.sleep(1000)
  })
  console.print_line("liftoff")
}

fn label(n: Int) -> String {
  if n == 1 { "1 second" } else { "#{n} seconds" }
}

@test
fn label_of_one_is_singular() {
  assert label(1) == "1 second"
}

@test
fn label_of_three() {
  assert label(3) == "3 seconds"
}
```

`src/main.emel`：

```emel
import Countdown
import Std.Platform.{Platform}

fn main() {
  with Platform {
    Countdown.countdown(3)
  }
}
```

```console
$ emela test --reporter plain
running 4 tests
test Countdown.label_of_one_is_singular ... ok
test Countdown.label_of_three ... ok
test CountdownTest.prints_each_second ... ok
test CountdownTest.takes_three_seconds ... ok

test result: ok. 4 passed; 0 failed; 0 filtered out
```

（`CountdownTest` の 2 つのテストは次の節にあります。）

- `@test` はパラメータを持たないトップレベル関数に付けます。`emela test` はプロジェクト内のそのような関数をすべて 1 つずつ実行します。
- ここで行っているように、テスト対象コードと同じファイルにテストを書くこともできます。その場合、`label` のような非公開関数も呼び出せます。
- テスト名は関数名になり、エントリモジュール以外にある場合はモジュール名が先頭に付きます（`Countdown.label_of_three`）。
- テスト関数は通常のビルドからは除外され、他のテストからしか呼び出せません。

`emela test` は結果の表示方法を状況に応じて選びます。端末では人間が読みやすいレイアウトで表示しますが、これはバージョンによって変わることがあります。`--reporter plain` は本書で使っている行単位のフォーマットを出力し、結果が同じであれば出力形式も一定に保たれます（`[cmd.test.reporter]`）。

一部のテストだけを実行するには、パスの後にフィルタを指定します。`emela test . label` は名前に `label` を含むテストだけを実行します。

### アサーションが失敗したとき

`assert` は `Bool` を受け取り、それが `False` のときにテストを失敗させます。もし `label_of_three` が `"3 second"` を期待していたら、次のようになります。

```console
$ emela test --reporter plain . label_of_three
running 1 test
test Countdown.label_of_three ... FAILED

failures:

---- Countdown.label_of_three ----
 --> src/countdown.emel:25:3
defect: assertion failed: label(3) == "3 second"
  left: "3 seconds"
 right: "3 second"

failures:
    Countdown.label_of_three

test result: FAILED. 0 passed; 1 failed; 3 filtered out
$ echo $?
1
```

レポートには `assert` の位置、ソースコードの式、そして比較式であれば両辺の値が表示されます。両辺の値が表示されるのは両辺が `Show` を持つ場合で、そうでなければソーステキストだけが表示されます。失敗した `assert` は欠陥です（第 5 章）。そのテストは終了しますが、残りのテストは引き続き実行されます。`assert` はテストの外でも使え、失敗したときにプログラムを停止させるチェックとして機能します。

テストは、処理されないエラーがテストから外へ出た場合や、他の欠陥で停止した場合にも失敗します。ある処理が失敗することを確かめるには、エラーを値に変換して比較します（第 5 章）。

```emel
error Empty

fn first(xs: List[Int]) -> Int fails Empty {
  xs match {
    [x, ..] -> x
    []      -> fail Empty
  }
}

@test
fn first_of_empty_fails() {
  result = Some(first([])) escape {
    Empty -> None
  }
  assert result == None
}
```

## エフェクトを使うコードをテストする

`countdown` は出力と待機を行います。テストから直接呼び出すことはできません。`main` と同じく、テストで使うエフェクトはすべてテストの中で提供する必要があるからです。`with` なしで `countdown(1)` を呼び出すテスト `needs_handlers` はコンパイルエラーになります。

```console
$ emela test
error[E4301]: effects `Console`, `Clock` are not provided in test function `needs_handlers`
    ╭─[ src/countdown.emel:30:3 ]
    │
 30 │   countdown(1)
    │   ──────┬─────
    │         ╰───────
────╯
   = help: provide it with `with`

1 error
```

`SystemConsole` と `SystemClock` の下で `countdown` を実行しても動きはしますが、端末に出力されてしまい、3 秒かかり、比較できる結果も手に入りません。標準ライブラリにはいくつかのエフェクト向けに、先頭に `Test` が付いたテスト用ハンドラが用意されています（`[lib.layers]`）。

- `TestConsole` は出力された行を、`State` で保持するリストに追加します。
- `TestClock` は仮想時刻を `State` で保持します。`sleep` はプログラム内で他に実行できるものがなくなるまでだけ待ち、その時点で時計が最も早いスリープの時刻まで進んでそれを起こします。1 時間スリープするテストも一瞬で終わります。
- `TestStdin`、`TestStdout`、`TestStderr` はプロセスの標準ストリームの代わりになります。

これらのハンドラは記録する内容を `State` エフェクトに保持するため、それぞれ `State.run` の中で使います。`State.run` は `State` を提供し、結果と並べて最終状態を返します（第 6 章）。2 つ使う場合は、`State.run` を 2 重に入れ子にします。次のテストは `test/countdown_test.emel` に置きます。

```emel
import Countdown
import Std.Clock.{TestClock}
import Std.Console.{TestConsole}
import Std.Instant
import Std.State.{State}

@test
fn prints_each_second() {
  start = Instant.from_unix_milliseconds(0)
  ((_, lines), _) = State.run(start, || {
    State.run([], || {
      with TestClock, TestConsole {
        Countdown.countdown(3)
      }
    })
  })
  assert lines == ["3 seconds", "2 seconds", "1 second", "liftoff"]
}

@test
fn takes_three_seconds() {
  start = Instant.from_unix_milliseconds(0)
  (_, end) = State.run(start, || {
    State.run([], || {
      with TestClock, TestConsole {
        Countdown.countdown(3)
      }
    })
  })
  assert Instant.to_unix_milliseconds(end) == 3000
}
```

外側の `State.run` が `TestClock` 用の時刻を保持し、内側の `State.run` が `TestConsole` 用の出力行を保持します。それぞれのテストは `countdown` を一切変更せず、何も引数として渡さずに、`countdown` が何をしたか（出力した行と費やした時間）を観測しています。2 つのテストを合わせても 1 秒よりずっと短い時間で終わります。

同じパターンは独自のエフェクトにもそのままスケールします。たとえばデータベース用の独自エフェクトを宣言するアプリケーションでは、テスト用のハンドラも宣言して通常はレイヤにまとめ、テストでは `with AppTest { ... }` の下でコードを実行します（第 10 章）。

## テストルート

プロジェクトはテストをソースコードとは分け、*テストルート*である `test/` 以下に置くことができます。

```text
launch/
├── Pome.toml
├── src/
│   ├── countdown.emel
│   └── main.emel
└── test/
    └── countdown_test.emel     # モジュール CountdownTest
```

- `test/` 以下のすべてのモジュールはテスト専用です。`@test` が付いていない宣言も通常のビルドからは除外されるため、テストデータを作る関数など、他のテスト向けのヘルパーもここに置けます。
- `test/` 以下のモジュールは `src/` 以下のモジュールをインポートできますが、見えるのは公開名だけです。また `[dev-dependencies]` の Pome もインポートできます（第 7 章）。
- `src/` 以下のモジュールは `test/` 以下のモジュールをインポートできません。

アサーションのヘルパー、テストデータジェネレータ、プロパティベーステストは標準ライブラリには含まれていません。それらは Pome `github.com/emela-lang/brix` にあり、プロジェクトにテスト依存関係として追加して使います（`emela pome add --dev github:emela-lang/brix`）。

## まとめ

- テストは `@test` が付いたパラメータなしの関数で、`emela test` がすべてを実行します。`False` になる `assert` は両辺の値を表示してテストを失敗させます。
- テストは、失敗した `assert`、処理されないエラー、その他の欠陥のいずれかによって失敗します。残りのテストは引き続き実行されます。
- テストはテスト対象コードのエフェクトをすべて提供します。`TestConsole` や `TestClock` などのテストハンドラはコードの振る舞いを `State` に記録し、`State.run` で取り出せます。
- テストはコードと同じ場所に置くことも（非公開関数が見える）、`test/` 以下に置くことも（公開名だけが見える）できます。

## 第 I 部の終わり

第 I 部では、アプリケーションを書くために必要なすべてを扱いました。データ、関数、繰り返し、エラー、エフェクト、モジュール、そしてテストです。ここで使ったのはすべて標準ライブラリのエフェクトとハンドラだけでした。

第 II 部では、それらの土台となっている基盤を扱います。ライブラリやプログラムの境界を書くための知識、すなわちトレイト、独自のエフェクトとハンドラ、適切なタイミングで解放されるリソース、並行タスク、そしてこれらを組み合わせて大きなプログラムを構成する方法です。第 9 章では、第 1 章から `derive Eq, Ord, Show` で使ってきた仕組みに名前を与える「トレイト」から始めます。
