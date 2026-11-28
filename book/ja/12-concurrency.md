# 12. 中断と構造化並行性

第 6 章では、`clock.sleep(100)` が処理を進める前に待機するにもかかわらず、他のどの呼び出しとも同じように書けることを見ました。この章では、中断（suspension）がどのように機能するか、関数や呼び出しに印を付けずにコードが中断し得るかどうかを型システムがどう追跡するか、そして構造化並行性がハンドラの上にタスク・コンビネータ・同期プリミティブをどう構築しているかを解説します（`[effect.decl]`、`[effect.async]`、`[type.effect-bound]`、`[task]`、`[dyn.suspend]`）。

## `suspend fn`、`resume`、`Async`、`Immediate`

中断し得ることを示すのは、`effect` の中の操作宣言だけです。`fn` の代わりに `suspend fn` と書きます（`[effect.decl]`）。

```emel
import Std.Clock.{Clock}

effect Delay {
  suspend fn pause(ms: Int) -> ()
}

handler InstantDelay: Delay {
  suspend fn pause(ms, resume) {
    resume(())
  }
}

handler RealDelay: Delay {
  suspend fn pause(ms, resume) {
    clock <- Clock
    clock.sleep(ms)
    resume(())
  }
}

fn step(n: Int) -> Int uses { Delay } {
  delay <- Delay
  delay.pause(50)
  n + 1
}

fn run_immediate[R: Immediate](f: fn() -> Int uses { R }) -> Int uses { R } {
  f()
}

fn main() {
  result = run_immediate(|| with InstantDelay { step(41) })
  dbg(result)
  ()
}
```

```console
$ emela run
42
```

- `step` は通常の呼び出し構文で `delay.pause(50)` を呼び出しており、シグネチャにも `uses { Delay }` としか書いていません。関数が中断し得るかどうかは、その `R` に含まれるエフェクトから決まります。
- `suspend fn` の実装は最後の追加パラメータとして `resume: fn(T) -> () fails {} uses {}`（`T` は操作の戻り値の型）を受け取り、実装本体自身の型は `()` でなければなりません（`[effect.handler]`）。`resume(v)` を呼ぶと、中断していた呼び出し元が呼び出しの値として `v` を受け取って再開します。継続を再開できるのは*最大 1 回*までで、2 回目の呼び出しは欠陥になります（`[dyn.suspend]`）。
- 実際に中断するかどうかはハンドラが決めます。`InstantDelay` はその場で直接 `resume(())` を呼ぶため、`Async` を要件とし*ません*（`[effect.async]`）。`with InstantDelay { step(41) }` が `Delay` を提供すると、`R` から `Delay` が取り除かれ、`Immediate` でないエフェクトは何も加わらないため、式全体が `Immediate` なエフェクト集合を持ち、`run_immediate` を通ります。対照的に、`RealDelay` は `clock.sleep(ms)` を呼ぶため要件に `Clock` が含まれ、`with RealDelay { step(41) }` は `run_immediate` に渡せません。
- `Async` は「後で再開する可能性がある」ことを表すホストエフェクトです（`[effect.async]`、`[effect.host]`）。ユーザーは `Async` のハンドラを書いたり `use` で取得したりできず、ホストだけが（`main`、テスト、エクスポートへ）提供します。`suspend fn` の実装が直接呼び出し以外の方法で `resume` を使う（変数に格納する、クロージャにキャプチャする、ホストのコールバックに渡す）か、`@external suspend fn` を呼び出すと、コンパイラはそのハンドラの要件に `Async` を追加します。
- `suspend fn` の実装内で、`resume` がすでに呼ばれた可能性がある箇所より後で中断し得る呼び出しを評価するコードはコンパイルできません（`[effect.handler]`）。呼び出し元を再開した後に続けたい処理は、代わりに `Task.Background` で開始します。
- 通常の `fn` 操作は中断できません（`[effect.decl]`）。本体は `Immediate` な `R` を持つか、中断し得る呼び出しを一切含まない必要があります。

### `Immediate` を強制する

ホスト関数に渡すコールバック（第 16 章）など、中断してはならない場所があります。言語のあらゆる関数や呼び出しに印を付ける代わりに、Emela は組み込みのエフェクト集合制約 `R: Immediate` で「中断してはならない」ことを表現します（`[type.effect-bound]`）。

- エフェクトが `Immediate` であるのは、`suspend fn` も `suspend handler` メンバも持たず、かつ `Async` ではないときです。
- エフェクト集合が `Immediate` であるのは、その要素がすべて `Immediate` であるときです（空集合は `Immediate` です）。

`sleep` が `suspend fn` である `Clock` を使う関数を `run_immediate` に渡すと、コンパイルエラーになります。

```emel
import Std.Clock.{Clock, SystemClock}

fn wait_and_add(n: Int) -> Int uses { Clock } {
  clock <- Clock
  clock.sleep(10)
  n + 1
}

fn run_immediate[R: Immediate](f: fn() -> Int uses { R }) -> Int uses { R } {
  f()
}

fn main() {
  with SystemClock {
    dbg(run_immediate(|| wait_and_add(41)))
    ()
  }
}
```

```console
$ emela check
error[E4302]: effect `Clock` may suspend, but `R` of `run_immediate` must be `Immediate`
    ╭─[ src/main.emel:15:9 ]
    │
 15 │     dbg(run_immediate(|| wait_and_add(41)))
    │         ──────┬──────
    │               ╰────────
────╯

1 error
```

操作の本体が `resume` を呼ぶ*前*に失敗した場合、そのエラーは操作の `fails` として呼び出し元へ返ります（`[dyn.suspend]`）。`resume` を呼んだ*後*に失敗または欠陥が起きた場合、呼び出し元はすでに先へ進んでいるため、プログラムやテストのトップレベルにおける欠陥になります。`main` やテストが終了した後に `resume` を呼んでも何も起きません。`suspend fn` の本体が `resume` を一度も呼ばずに返った場合、待機していた継続は*割り込まれた*ものとしてアンワインドします（抜ける途中で各 `release` が実行されます）。

## タスクとファイバ

並行な処理は*ファイバ*上の*タスク*として実行されます。ファイバは単一スレッド上で協調的にスケジューリングされる軽量な実行単位です（`[task.basic]`）。0.20 系には OS スレッドも、ロックも、並列実行もありません（`[task.parallel]`）。タスクがプログラムを高速化するのは、純粋な CPU 計算を分割するときではなく、待機（ネットワークリクエスト、タイマー、I/O）を重ね合わせるときです。

ファイバは協調的に切り替わるため、切り替えが起きるのは*中断点*だけです。

- `Async` を要件とする `suspend fn` 操作の呼び出し（または `suspend handler` を開くときや解放するとき）。
- 公平性のためにランタイムが挿入するイールド（制御の譲り渡し）。これは中断し得る呼び出しの回数が一定に達したときに挿入されます。ランタイムは経過時間ではなく呼び出し回数を数えるため、制御を譲る箇所は毎回同じになります。

中断し得る呼び出しを含まない一連のコードは、他のタスクに対して*アトミック*に実行されます。

- `st.set(st.get() + 1)` の途中に別のタスクが割り込むことはありません。
- `Immediate` な関数は分割不可能な 1 ステップとして実行され、その中に公平性のためのイールドが挿入されることはありません。
- `x = st.get(); helper(); st.set(x + 1)` では、`helper` の `R` が `Immediate` でない限り、`helper()` の実行中に別のタスクが動く可能性があります。

## `Task.Group`

すべてのタスクは、そのライフタイムを区切る `with` 内のハンドラに属します（`[task.group]`）。

```emel
import Std.Clock.{Clock, TestClock}
import Std.Console.{Console, SystemConsole}
import Std.Instant
import Std.State.{State}
import Std.Task

fn fetch(name: String, ms: Int) -> String uses { Clock, Console } {
  clock <- Clock
  console <- Console
  clock.sleep(ms)
  console.print_line("done #{name}")
  name
}

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  with SystemConsole {
    (pair, end) = State.run(start, || {
      with TestClock {
        with tasks = Task.Group {
          a = tasks.start(|| fetch("user", 100))
          b = tasks.start(|| fetch("orders", 40))
          (a.join(), b.join())
        }
      }
    })
    console <- Console
    console.print_line("pair: #{pair}, elapsed: #{Instant.to_unix_milliseconds(end)}ms")
  }
}
```

```console
$ emela run
done orders
done user
pair: ("user", "orders"), elapsed: 100ms
```

2 つの取得処理が重なって実行され、`orders` は 40ms で、`user` は 100ms で終わり、合計の仮想時間は 140ms ではなく 100ms になっています。

- `tasks.start(f)` は `f` をキューに積み、中断せずに即座に `Task[A]` を返します。新しいタスクは、現在のタスクが次に中断したときに実行を始めます。`a.join()` は `a` の完了を待ってその値を返します（2 回目以降の `join` は記憶されている値を即座に返します）。
- タスクは自身のグループのケイパビリティを通じて兄弟タスクを開始できます。
- `with tasks = Task.Group` の本体が値を返すと、まだ実行中のタスクはすべて割り込まれ、グループの `release` はすべてのタスクのアンワインド（各タスク自身の `release` を含む）が終わるまで待ちます。
- いずれかのタスクが（`fail` または欠陥で）失敗すると、兄弟タスクがすべて割り込まれ、失敗は本体の次の中断点で届けられます。すべてのタスクがアンワインドした後、`with` はその失敗で終了します。複数のタスクが失敗した場合は、最初のものが結果になり、残りはノートとして表示されます。
- グループは、本体が終了したとき、またはタスクが失敗したときに閉じます。閉じたグループで `start` を呼ぶと欠陥になります。
- `Task[A]` は `@external pub type Task[A] uses TaskGroup` という*ケイパビリティ束縛型*として宣言されています（`[type.rules]`）。型の中にグループのケイパビリティを保持するため、`Task[A]` は自身のグループの `with` の外へ出られません。
- 呼び出し側のグループでタスクを開始するヘルパー関数は、集合型パラメータ `E` と `R` を持つケイパビリティパラメータを受け取り、自身のタスクが必要とするエフェクトを列挙します（`fn crawl[E, R](tasks: use TaskGroup[E, { Http, Async, R }], url: String) -> ()`）。より大きな `E` や `R` を持つグループは、より小さな集合の要件を満たします（`[task.group]`）。
- `Task.Group` の `E` と `R` はその中で開始されるタスクから推論され、グループの `with` の箇所で確定します（`[task.group]`）。グループの*内側*で提供したエフェクトは、そのタスクには届きません。

```emel
import Std.Console.{Console, SystemConsole}
import Std.Task

fn main() {
  with tasks = Task.Group {
    with SystemConsole {
      tasks.start(|| {
        console <- Console
        console.print_line("hi")
      })
    }
    ()
  }
}
```

```console
$ emela check
error[E4301]: effect `Console` is not provided in `main`
    ╭─[ src/main.emel:7:7 ]
    │
  7 │ ╭─▶       tasks.start(|| {
    ┆ ┆
 10 │ ├─▶       })
    │ │
    │ ╰───────────────
────╯
   = note: tasks get their handlers from outside the `with` of their group, so `with` inside it does not reach them
   = help: provide it with `with` outside the group

1 error
```

ハンドラがグループ内のすべてのタスクより長く生きるように、`SystemConsole` は `Task.Group` の外側（または同じ `with` 内の `tasks = Task.Group` より前）へ移動します。

## コンビネータ

日常的なコードのほとんどは `Task.Group` を直接開く必要がありません。`Std.Task` は `Task.Group` と `Deferred` の上に、素の Emela で 5 つのコンビネータを構築しています（`[task.combinators]`）。

| 関数 | 振る舞い |
|---|---|
| `Task.both(f, g)` | 型が異なっていてもよい 2 つの関数を並行に実行し、`(A, B)` を返す |
| `Task.all(fs)` | パラメータのない関数の `List` を並行に実行し、結果を元の順で集める |
| `Task.map(xs, f)` | 各要素に `f` を並行に適用し、結果を元の順で集める |
| `Task.race(fs)` | 最初に完了した結果（またはエラー）を返し、残りに割り込む |
| `Task.timeout(ms, f)` | `Clock` 上で `ms` ミリ秒以内に `f` が終われば `Some(a)` を返し、そうでなければ `f` に割り込んで `None` を返す |

```emel
import Std.Clock.{Clock, TestClock}
import Std.Instant
import Std.State.{State}
import Std.Task

fn slow(label: String, ms: Int) -> String uses { Clock } {
  clock <- Clock
  clock.sleep(ms)
  label
}

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  (summary, _) = State.run(start, || {
    with TestClock {
      pair = Task.both(|| slow("a", 50), || slow("b", 30))
      mapped = Task.map([10, 30, 20], |ms| slow("#{ms}ms", ms))
      winner = Task.race([|| slow("slow", 100), || slow("fast", 20)])
      timed = Task.timeout(50, || slow("late", 200))
      (pair, mapped, winner, timed)
    }
  })
  dbg(summary)
  ()
}
```

```console
$ emela run
(("a", "b"), ["10ms", "30ms", "20ms"], "fast", None)
```

`Task.timeout` は `Clock` エフェクトを使うため、`TestClock` の下では仮想時間で動作し一瞬で完了します。

## `Task.Background`、`Exit.attempt`、`Cause`

監査ログの記録やバックグラウンドの後始末のように、結果を待たない処理を起動したい場合は `Task.Background` を使います（`[task.background]`）。

- そのケイパビリティの型は `use Spawn[E, R]` で、`bg.start(f)` は `()` を返す関数を取ります。
- `Task.Group` と異なり、`Background` は決して閉じません。`release` が待機している間でも `bg.start(f)` を呼ぶことができ、欠陥になりません。
- 本体が（正常に、あるいはエラーで）終わっても、`Background` はタスクに割り込み*ません*。`release` は、待機中に新しく開始されたタスクも含めてすべてが完了するまで待ちます。
- バックグラウンドタスクが失敗しても、兄弟タスクや本体はそのまま動き続けます。代わりに、そのタスクがアンワインドした後にそのタスク内で `on_failure: fn(Cause[E]) -> () uses { R }` が呼ばれます。`on_failure` の `E` は空でなければならず、`on_failure` 内の欠陥は `release` 内の欠陥と同じように扱われます。
- バックグラウンドタスクが割り込まれるのは、`Task.Background` の `with` 自体が外側から割り込まれた場合だけです（その場合、`on_failure` は呼ばれません）。

タスク内や境界における失敗は、`Exit[A, E]`（`Success(A)` または `Failure(Cause[E])`）と `opaque` 型 `Cause[E]` で表されます（`[task.cause]`）。

- 原因の種類（`Cause.kind`）は `Fail`（`fail` によるもので `E` で追跡される）か `Die`（欠陥によるもの）の 2 つです。割り込みは原因では*なく*、値として受け取ることは決してありません。
- `Exit.attempt(f)` は（新しいタスクを開始せず）その場で `f` を実行し、`Fail` と `Die` の両方を `Failure(cause)` として捕捉します。欠陥を値として観測できる境界はここですが、自身のタスクへの割り込みは捕捉せず、アンワインド中の `Die` も捕捉しません。
- Emela には「`E` のいずれか 1 つ」を表す合併値型がないため、`Cause[E]` からエラー値を直接取り出すことはできません。`Cause.kind` と `Cause.to_string` で調べるか、`Cause.raise(cause)` で再送出して具体的なエラーを `escape` で処理します。

```emel
import Std.Cause
import Std.Console.{Console, SystemConsole}
import Std.Exit
import Std.Exit.{Failure, Success}
import Std.Task

error Missing(key: String) derive Show

fn lookup(key: String) -> String fails Missing {
  if key == "ok" { "found" } else { fail Missing(key:) }
}

fn main() {
  with SystemConsole {
    console <- Console
    with bg = Task.Background(
      on_failure: |cause| console.print_line("bg failed: #{Cause.to_string(cause)}"),
    ) {
      bg.start(|| {
        _ = lookup("nope")
        ()
      })
      console.print_line("main body done")
    }

    Exit.attempt(|| lookup("missing")) match {
      Success(v) -> console.print_line("got #{v}")
      Failure(cause) -> {
        console.print_line("kind=#{Cause.kind(cause)} msg=#{Cause.to_string(cause)}")
        recovered = Cause.raise(cause) escape {
          Missing(key:) -> "fallback for #{key}"
        }
        console.print_line(recovered)
      }
    }

    Exit.attempt(|| panic("bug")) match {
      Success(_) -> ()
      Failure(cause) -> console.print_line(
        "kind=#{Cause.kind(cause)} msg=#{Cause.to_string(cause)}",
      )
    }
  }
}
```

```console
$ emela run
main body done
bg failed: error: Missing(key: "nope")
kind=Fail msg=error: Missing(key: "missing")
fallback for missing
kind=Die msg=defect: bug
```

バックグラウンドタスクが始まる前に `"main body done"` が出力されている点に注目してください。`bg.start` はタスクをキューに積んだだけで、実際に動き出したのは `with bg = Task.Background` が `release` に入って待機したタイミングです。

## 割り込み

ハンドルを指定して単一のタスクに割り込む関数はありません（`[task.interrupt]`）。割り込みは常に構造から発生します。`Task.Group` の本体が終了したとき、`Task.Group` 内の兄弟タスクが失敗したとき、あるいは外側の `with` が割り込まれたときです。特定のタスクを任意のタイミングで止めたい場合は、そのタスク専用の `Task.Group` に置くか、停止シグナル用の `Deferred` を渡します。

- 割り込みは、対象タスクの次の中断点で効力を発揮します。
- その地点からタスクはアンワインドし、内側から外側へすべての `release` を実行します。
- `release` の実行中にタスクが割り込まれることはありません。保留中の割り込みは、その `release` が終わった時点で効力を発揮します。
- `escape` も `Exit.attempt` も割り込みを握りつぶすことはできません。アンワインドが `escape` を通過したとしても、次の中断点で割り込みが再開されます。

## `Deferred`、`Semaphore`、`Queue`

タスク間を協調させるための 3 つの組み込みハンドラがあり、いずれも `Task.Group` と同じく `with` にスコープされます（`[task.deferred]`、`[task.semaphore]`、`[task.queue]`）。

- `Deferred[A]` は最大 1 回だけ値を埋められる器です。`complete(v)` は初回のみ `v` を格納し（初回は `True`、以降は `False` を返す）、`await()` は値が埋まるまで中断して待ち（埋まるとすべての待機者を起こす）、`poll()` は中断せずに確認します。
- `Semaphore(permits: n)` は並行数を `n` 個の許可に制限し、待機を始めた順に割り当てます。許可はハンドラメンバ（`Permits(n:)` は利用可能になるまで中断して待ち、`TryPermits(n:)` は `n` 個が空いていて待機中のタスクがいないときだけ取得し、そうでなければ `NoPermits` で失敗する）なので、正常終了・エラー・欠陥・割り込みのいずれでも `release` によって自動的に返却されます。利便性のために `with` を包んだ `Semaphore.with_permit` / `with_permits` と `try_with_permit` / `try_with_permits` も用意されています。`available()` はその瞬間の空き許可数を返しますが、後続の取得操作とはアトミックではありません。
- `Queue[A](capacity: c)` は有界な FIFO チャネルです。`offer(v)` は満杯の間待機し、`take()` は空の間待機し、`try_offer` と `poll` は中断しません。`close()` でキューを閉じると、残っている要素はすべて取り出せますが、以降の `offer`（および空になった後の `take`）は `QueueClosed` で失敗します。待機中の送信側も受信側も FIFO 順で処理され、割り込まれた `offer` は要素を追加しません。

```emel
import Std.Clock.{Clock, TestClock}
import Std.Console.{Console, SystemConsole}
import Std.Deferred.{Deferred}
import Std.Instant
import Std.Queue.{Queue, QueueClosed}
import Std.Semaphore
import Std.Semaphore.{Semaphore}
import Std.State.{State}
import Std.Task

fn main() {
  start = Instant.from_unix_milliseconds(Int64.from_int(0))
  with SystemConsole {
    console <- Console
    (_, _) = State.run(start, || {
      with TestClock {
        # Deferred：1 回限りのシグナル
        with ready = Deferred, tasks = Task.Group {
          _ = tasks.start(|| {
            clock <- Clock
            clock.sleep(10)
            _ = ready.complete(8080)
            ()
          })
          port = ready.await()
          console.print_line("listening on #{port}")
        }

        # Semaphore：並行数の制限
        with limit = Semaphore(permits: 2) {
          results = Task.map([1, 2, 3, 4], |id| {
            Semaphore.with_permit(limit, || {
              clock <- Clock
              clock.sleep(10)
              id * 10
            })
          })
          console.print_line("results: #{results}")
        }

        # Queue：クローズ付き有界 FIFO
        with jobs = Queue(capacity: 2), tasks = Task.Group {
          _ = tasks.start(|| {
            List.each(["a", "b", "c"], |item| jobs.offer(item))
            jobs.close()
          })
          fn drain() {
            item = jobs.take()
            console.print_line("job: #{item}")
            drain()
          }
          drain() escape {
            QueueClosed -> console.print_line("queue closed")
          }
        }
      }
    })
  }
}
```

```console
$ emela run
listening on 8080
results: [10, 20, 30, 40]
job: a
job: b
job: c
queue closed
```

`Deferred` と `Queue` の型引数はケイパビリティの使い方から推論されますが、明示的に書くこともできます（`Deferred[Int]`、`Queue[String](capacity: 2)`）。`Deferred`、`Semaphore`、`Queue` はそれぞれの `with` のケイパビリティです。そのため、これらを使うタスクは必ずその `with` の内側にネストされた `Task.Group` や `Task.Background`、あるいは同じ `with` 内でこれらの後ろに書かれた `Task.Group` や `Task.Background` の中に置く必要があり、スコープ終了後に使われることはありません。

## まとめ

- 中断を記すのは `suspend fn` 宣言だけで、関数や呼び出しに印は付きません。ハンドラは 1 回限りの `resume` を受け取り、その場で再開する（`Immediate`）か後で再開する（`Async`）かを選びます。
- `R: Immediate` は、ホストコールバックや `fn` 操作など中断してはならない場所から中断を静的に排除します。
- タスクは単一スレッド上のファイバで協調的に動き、中断点でのみ切り替わります。中断し得る呼び出しを含まない区間はアトミックに実行され、公平性のためのイールドは決定的です。
- すべてのタスクは `Task.Group`（終了時には残りのタスクに、失敗時には兄弟タスクに割り込み、終了・失敗で閉じ、`Task[A]` をスコープ内に閉じ込める）か `Task.Background`（タスクの完了を待ち、閉じることがなく、失敗を `on_failure` へ渡す）の中に属します。
- `Task.both`、`all`、`map`、`race`、`timeout` は素の Emela で一般的な並行パターンをカバーします。
- `Exit.attempt` は割り込みを捕捉することなく、`Fail` と `Die` を `opaque` な `Cause[E]` を持つ `Exit[A, E]` の値へ変換します。
- `Deferred`、`Semaphore`、`Queue` は、1 回限りの受け渡し・並行数制限・有界ワークキューのためのスコープ付きハンドラです。

第 13 章では、データ、`opaque`、トレイト、エフェクト、ケイパビリティをすべて組み合わせ、合成によってプログラムをどう設計するかを見ていきます。
