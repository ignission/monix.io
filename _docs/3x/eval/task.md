---
layout: docs3x
title: Task
type_api: monix.eval.Task
type_source: monix-eval/shared/src/main/scala/monix/eval/Task.scala
description: |
  このデータ型は非同期の計算や、副作用の制御、不確定性の回避、コールバック地獄の回避に役立ちます。
---

## イントロダクション

`Task`は遅延する可能性のある、非同期計算を制御するためのデータ型です。副作用の制御や非決定論やコールバック地獄の回避に役立ちます。

importを整理します:

```scala mdoc:silent
// タスクを評価するためには、Schedulerが必要です
import monix.execution.Scheduler.Implicits.global

// キャンセル可能なFuture型
import monix.execution.CancelableFuture

// Taskはmonix.eval内にあります
import monix.eval.Task
import scala.util.{Success, Failure}
```

```scala mdoc:reset:invisible
import monix.execution.CancelableFuture
import monix.eval.Task
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

```scala mdoc:silent:nest
// sumを実行すると、(applyのセマンティクスにより)別のスレッドで発生します
// しかし、このインスタンスの構築時には何も起こりません
// この表現は純粋で、単なる仕様です。タスクはデフォルトでは遅延動作をします ;-)
val task = Task { 1 + 1 }

// タスクはrunToFutureでのみ評価されます
// コールバックスタイル:
val cancelable = task.runAsync { result =>
  result match {
    case Right(value) =>
      println(value)
    case Left(ex) =>
      System.out.println(s"ERROR: ${ex.getMessage}")
  }
}
//=> 2

// または、Futureに変換することもできます
val future: CancelableFuture[Int] =
  task.runToFuture

// 結果を非同期にプリントします
future.foreach(println)
//=> 2
```

### 設計の概要

Monixの`Task`を要約します:

- 遅延 &amp; 非同期評価
- 1つのプロデューサーが1つまたは複数のコンシューマーに1つの値だけをプッシュするモデル
- [実行モデル](../execution/scheduler.md#execution-model)をきめ細かく制御することができる
- `runToFuture`が呼ばれるまでは、実行やその効果は引き起こされません
- 必ずしも別の論理スレッドで実行する必要はありません
- 実行中の計算をキャンセルすることができます
- Haskellの機能と同様に副作用を制御することができます(HaskellのI/Oと同じような効果があります)
- 実装上、スレッドをブロックすることはありません
- スレッドをブロックするようなAPIコールを公開していません

これらの仕様を表にまとめます:

|                    |        先行         |            遅延             |
|:------------------:|:-------------------:|:--------------------------:|
| **同期**            |          A          |           () => A          |
|                    |                     | [Coeval[A]](./coeval.md)   |
| **非同期**          | (A => Unit) => Unit |    (A => Unit) => Unit     |
|                    |      Future[A]      |          Task[A]           |

### ScalaのFutureとの比較

`Task`はScalaの [Future](http://docs.scala-lang.org/overviews/core/futures.html)と似ていますが違った性格を持っており、この2つのタイプは実際には補完関係にあることがわかります。  
賢い人はかつてこう言いました。

> "*未来は時間から切り離された価値を表す*" &mdash; Viktor Klang

価値とは何か、時間とは何かを考えさせられる詩的な発想ですね。
しかし、より重要なことは`Future`が [値](https://en.wikipedia.org/wiki/Value_(computer_science))であるとは言えないものの、*値になりたがっている* とは確実に言えるということです。
つまり、`Future`の参照を受け取ったとき、それを評価しようとしているどんなプロセスも、おそらくすでに始まっていて、またすでに終了しているかもしれません。
これにより、Scalaの`Future`の動作は *先行評価* となり、`map`や`flatMap`のような演算子を呼ぶときに、どのように暗黙の実行コンテキストを取るかを考えれば、確かにその設計は助けになります。

しかし、`Task`は違います。Taskは`遅延評価`です。実際、`Task`では実行モデルを微調整することができますが、これが両者の主な違いです。
`Future`が値のようなものなら、`Task`は関数のようなものです。そして実際に、`Task`は`Future`インスタンスの「ファクトリー」として機能します。

もうひとつの特徴は、`Future`はデフォルトで「*メモ化*される」ということです。
つまり、必要に応じて複数のコンシューマー間で共有されます。
しかし、`Task`の評価は、デフォルトではメモ化されません。メモ化を実現するためには、明示的に指定しなければなりません。

先行評価の動作を持つ`Future`は、効率面で劣ります。
というのも、どのような操作を行っても実装は最終的にスレッドプールに`Runnable`インスタンスを送ることになり、結果が各ステップで常にメモ化されるからです。
一方、`Task`は同期的なバッチで実行することができます。

### Scalazのタスクとの比較

MonixのTaskが、[Scalaz](https://github.com/scalaz/scalaz)のTaskにインスパイアされたことは周知の事実です。Monixライブラリ全体が巨人の肩の上に立っています。  
Monix Taskの実装が異なる点は:

1. ScalazのTaskは実装の詳細を漏らしています。
   というのも、ScalazのTaskはまず*トランポリン*実行を目的としていますが、非同期実行は非同期のトランポリン境界を飛び越えることを目的としているからです。
   例えば、大きなループで現在のスレッドをブロックしないようにするには、`Task.executeAsync`を使って非同期の境界を手動で挿入しなければなりません。
   一方、MonixのTaskは、デフォルトで自動的にそれを行うようになっています。
   これは、[Javascript](http://www.scala-js.org/)の上で実行するときに非常に便利です。
   [協調的マルチタスク](https://en.wikipedia.org/wiki/Cooperative_multitasking)はあったほうがいいだけでなく、必要なのです。
2. ScalazのTaskは同期/非同期の2面を持っています。これは生産者にとっては最適化のために良いことです(例えば、必要がないのになぜスレッドをフォークするのか)。
   しかし、コンシューマの視点から見ると、`def run: A`は、JavascriptやJVM上でAPIを完全にサポートできないことを意味します。
   つまり、Scalazの`Task`は結局、同期評価やスレッドのブロックを偽装することになるのです。
   そして、[スレッドをブロックすることは非常に安全ではありません](../best-practices/blocking.md)
3. ScalazのTaskは、実行中の計算をキャンセルすることはできません。これは非決定論的な操作では重要です。
   例えば、`race`で競合状態を作ったときに、時間内に終わらなかった遅いタスクをキャンセルしたい場合があります。
   というのもリソースをすぐに解放しないと、残念ながら深刻なリークが発生してプロセスがクラッシュしてしまうことがあるからです。
4. Scalazタスクは、非同期実行を扱うJavaの標準ライブラリを利用しています。
   これはポータビリティの観点からは好ましくありません。このAPIは[Scala.js](http://www.scala-js.org/)の上ではサポートされていないからです。
   
## 実行 (runToFuture & foreach)

Taskインスタンスは、`runToFuture`によって実行されるまで何もしません。また、複数のオーバーロードがあります。

`Task.runToFuture`では`ExecutionContext`を継承した[Scheduler](../execution/scheduler.md)を暗黙のパラメータとして求められます。
しかし、ここから`Task`とScala標準の`Future`との設計の違いが出てきます。
遅延という性格を持つ`Task`は、`runToFuture`を使った実行時にのみこの`Scheduler`を必要とし、Scalaの`Future`のようにすべての操作(`map`や`flatMap`など)で必要とされるわけではありません。

ではまず、スコープ内に`Scheduler`を用意しましょう。
この`global`は、Scala独自の`global`に乗っかっているので、次のようなことができます:

```scala mdoc:silent:nest
import monix.execution.Scheduler.Implicits.global
```
```scala mdoc:invisible:nest
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

**注:** [Scheduler](../execution/scheduler.md)は、非同期の境界がどのように強制されるか(あるいはされないか)を決定する設定可能な[実行モデル](../execution/scheduler.md#execution-model)を注入することができます。

最もわかりやすく慣用的な方法は、タスクを実行して[CancelableFuture]({{ page.path | api_base_url }}monix/execution/CancelableFuture.html)を返すことです。
これは、標準的な`Future`と[Cancelable](../execution/cancelable.md)を組み合わせたものです。

```scala mdoc:silent:nest
import monix.eval.Task
import monix.execution.CancelableFuture
import concurrent.duration._

val task = Task(1 + 1).delayExecution(1.second)

val result: CancelableFuture[Int] =
  task.runToFuture

// 気が変わってキャンセルしたくなったら
result.cancel()
```

`Future`を返すのが重すぎるのであれば、シンプルなコールバックを用意したほうがいいかもしれません。
標準の`Future.onComplete`と同様に、`Either[Throwable, A] => Unit`コールバックを使って`runAsync`することもできます。

```scala mdoc:silent:nest
val task = Task(1 + 1).delayExecution(1.second)

val cancelable = task.runAsync { result =>
  result match {
    case Right(value) =>
      println(value)
    case Left(ex) =>
      System.err.println(s"ERROR: ${ex.getMessage}")
  }
}

// 気が変わってキャンセルしたくなったら...
cancelable.cancel()
```

また、[Callback](../execution/callback.md)インスタンスを使って`runAsync`することもできます。
これはJava的なAPIのようなもので、状態を保持したい場合に便利です。
`Callback`は内部的にも使用されています。なぜなら契約違反を防ぎ、`Try[T]`や`Either[E, A]`に特有のボクシングを回避することができるからです。  
例:

```scala mdoc:silent:nest
import monix.execution.Callback

val task = Task(1 + 1).delayExecution(1.second)

val cancelable = task.runAsync(
  new Callback[Throwable, Int] {
    def onSuccess(value: Int): Unit =
      println(value)
    def onError(ex: Throwable): Unit =
      System.err.println(s"ERROR: ${ex.getMessage}")
  })

// 気が変わってキャンセルしたくなったら...
cancelable.cancel()
```

しかし、いくつかの副作用を素早く発生させたい場合は、`foreach`を直接使うことができます:

```scala mdoc:silent:nest
val task = Task { println("Effect!"); "Result" }

task.foreach { result => println(result) }
//=> Effect!
//=> Result

// もしくはfor式も同様です
for (result <- task) {
  println(result)
}
```

注: `Task`の`foreach`はブロックせずに`CancelableFuture[Unit]`を返します。これは任意に実行をブロックしたり、キャンセルするために使用できます。 

### 結果をブロッキングする

[Monixはその哲学としてブロッキングに反対](../best-practices/blocking.md)しており、したがって`Task`にはスレッドをブロックするAPIコールは一切ありません！

しかし、JVMの上では時にはブロックしなければならないこともあります。
なぜなら、標準の`Await.result`と`Await.ready`には、2つの健全な設計上の選択があるからです。

1. これらのコールは、Scalaの`BlockContext`を使って実装されています。
   ブロッキング操作が実行されていることを基礎となるスレッドプールにシグナリングし、スレッドプールがそれに対応できるようにします。
   例えばScalaの`ForkJoinPool`がやっているように、プールにスレッドを追加することを決めるかもしれません。
2. これらの呼び出しには、非常に明示的なタイムアウト・パラメータが必要で、それは`FiniteDuration`として指定されます。

したがって、まず `runToFuture`で結果を`Future`に変換してから、結果をブロックすることができます:

```scala mdoc:silent:reset
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global
import scala.concurrent.Await
import scala.concurrent.duration._

val task = Task.eval("Hello!").executeAsync
val future = task.runToFuture

Await.result(future, 3.seconds)
//=> Hello!
```

```scala mdoc:invisible:reset
import monix.eval._
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

**注:** Scala.jsでは設計上、[ブロッキングはありません](https://github.com/scala-js/scala-js/issues/186)。

### 即時実行を試みる

Monixはブロッキングに反対していることは確かです。
しかし、実行モデルで許可されていれば、現在の論理スレッドですぐに評価できる`Task`インスタンスもあることは明らかです。
そして*最適化のため*、コールバックの処理を避けて、その結果をすぐに実行したいと思うかもしれません。

そのためには、`runSyncStep`を使います:

```scala mdoc:silent:nest
val task = Task.eval("Hello!")

task.runSyncStep match {
  case Left(task) =>
    // このタスクは非同期実行を強く望んでいるのでうまくいきません
    task.runToFuture.foreach(r => println(s"Async: $r"))
  case Right(result) =>
    println(s"Got lucky: $result")
}
```

**注:** 偶然にも、`runSyncStep`のデフォルト評価は、基礎となるループで非同期の境界が強制されない限り、現在のスレッドで処理を実行します。
そのため、このコードは常に*GotLucky*と表示されます。 ;-)

## シンプルなビルダー

非同期の可能性があるという性質を受け入れることができれば、`Task`は引数がない関数、Scalaの名前渡しパラメータ、`lazy val`を置き換えることができます。 また、Scala の `Future`はすべて `Task`に変換できます。

### Task.now

`Task.now`は、`Task`コンテキストで既に知られている値をリフトします。
これは`Future.successful`や`Applicative.pure`に相当します:

```scala mdoc:silent:nest
val task = Task.now { println("Effect"); "Hello!" }
//=> Effect
// task: monix.eval.Task[String] = Delay(Now(Hello!))
```

### Task.eval (遅延)

`Task.eval`は、`Function0`と同等であり`runToFuture`で可能な限り同じスレッドで評価される関数を受け取ります([選択された実行モデル](../execution/scheduler.md#execution-model)によります)。

```scala mdoc:silent:nest
val task = Task.eval { println("Effect"); "Hello!" }
// task: monix.eval.Task[String] = Delay(Always(<function0>))

task.runToFuture.foreach(println)
//=> Effect
//=> Hello!

// 評価(およびそれに伴うすべての副作用)は
// runToFutureごとに起動されます:
task.runToFuture.foreach(println)
//=> Effect
//=> Hello!
```

注: Scalazの場合、この関数は`Task.delay`と呼ばれています。

### Task.evalOnce

`Task.evalOnce`は、Scalaでは正確に表現できない型である`lazy val`に相当します。
`evalOnce`ビルダーは最初の実行時にメモ化を行い、評価の結果を次の実行時にも利用できるようにします。
このビルダーは冪等性を保証し、スレッドセーフです:

```scala mdoc:silent:nest
val task = Task.evalOnce { println("Effect"); "Hello!" }
// task: monix.eval.Task[String] = EvalOnce(<function0>)

task.runToFuture.foreach(println)
//=> Effect
//=> Hello!

// 結果は最初の実行でメモ化されました！
task.runToFuture.foreach(println)
//=> Hello!
```

注: この操作は実質的に`Task.eval(f).memoize`です。

### Task.defer (延期)

`Task.defer`は、タスクのファクトリを構築するためのものです。例えばこれは、`Task.eval`とほぼ同様の動作をします。

```scala mdoc:silent:nest
val task = Task.defer {
  Task.now { println("Effect"); "Hello!" }
}
// task: monix.eval.Task[String] = Suspend(<function0>)

task.runToFuture.foreach(println)
//=> Effect
//=> Hello!

task.runToFuture.foreach(println)
//=> Effect
//=> Hello!
```

注: Scalazの場合、この関数は`Task.suspend`という名前で呼ばれています。

### Task.fromFuture

`Task.fromFuture`は、任意の`Future`インスタンスを`Task`に変換することができます:

```scala mdoc:silent:nest
import scala.concurrent.Future

val future = Future { println("Effect"); "Hello!" }
val task = Task.fromFuture(future)
//=> Effect

task.runToFuture.foreach(println)
//=> Hello!
task.runToFuture.foreach(println)
//=> Hello!
```

なお、`fromFuture`は厳密な引数を取りますが、これはあなたが望むものではないかもしれません。
しかし、`Task`は評価モデルを細かく制御することを目的としているので、ファクトリーが必要な場合は`Task.defer`と組み合わせる必要があります:

```scala mdoc:silent:nest
val task = Task.defer {
  val future = Future { println("Effect"); "Hello!" }
  Task.fromFuture(future)
}
//=> task: monix.eval.Task[Int] = Suspend(<function0>)

task.runToFuture.foreach(println)
//=> Effect
//=> Hello!
task.runToFuture.foreach(println)
//=> Effect
//=> Hello!
```

### Task.deferFuture

`Future`の参照は厳密な値のようなもので、これを受け取るとそれを実行させるためのプロセスがすでに始まっていることになります。  
そのため、Taskを構築する際にFutureの評価を先送りすることに意味があります。

```scala mdoc:silent:nest
val task = Task.defer {
  val future = Future { println("Effect"); "Hello!" }
  Task.fromFuture(future)
}
```

上記の近道として、`deferFuture`ビルダーを使うこともできます:

```scala mdoc:silent:nest
val task = Task.deferFuture {
  Future { println("Effect"); "Hello!" }
}
```

### Task.deferFutureAction

`Future`の結果を生成するコールを`Task`にラップし、必要な`ExecutionContext`として動作する`Scheduler`を注入したコールバックを提供します。

このビルダーは、暗黙の`ExecutionContext`を必要とする`Future`対応APIの使用に役立ちます。  
以下の例を考えてみましょう:

```scala mdoc:silent:nest
import scala.concurrent.{ExecutionContext, Future}

def sumFuture(list: Seq[Int])(implicit ec: ExecutionContext): Future[Int] =
  Future(list.sum)
```

この関数を、呼ばれるたびに和を評価する遅延型の`Task`を返す関数にしたいと思います。
Taskが最もよく機能する方法だからです。しかし、この関数を呼び出すためには`ExecutionContext`が必要です:

```scala mdoc:silent:nest
def sumTask(list: Seq[Int])(implicit ec: ExecutionContext): Task[Int] =
  Task.deferFuture(sumFuture(list))
```

しかしこれは余計なお世話であるだけでなく、`Task`を使用するベストプラクティスに反しています。
その違いは、`Task`は`runToFuture`が呼ばれたときにのみ`Scheduler`(`ExecutionContext`を継承したもの)を取るということです。
しかし、`Task`の参照を構築するためだけにそれを必要とするわけではありません。
`DeferFutureAction`では、渡されたコールバックに`Scheduler`が注入されます:

```scala mdoc:silent:nest
def sumTask(list: Seq[Int]): Task[Int] =
  Task.deferFutureAction { implicit scheduler =>
    sumFuture(list)
  }
```

Voilà!(ほらできました)。もう暗黙のパラメータ`ExecutionContext`を渡す必要はありません。

### Task.executeAsync, Task.asyncBoundary, Task.executeOn

`Task.executeAsync`は、実行時に(論理)スレッドのフォークを強制することで、非同期の境界を確保します。
時には本当に無駄なことをしていて、非同期の境界が発生することを保証したいことがあります。
デフォルトでは[実行モデル](../execution/scheduler.md#execution-model)は最初は現在のスレッドでの実行を好みます。

これにより、私たちのタスクが非同期に実行されることが保証されます:

```scala mdoc:silent:nest
val task = Task.eval("Hello!").executeAsync
```

`ExecuteOn`では、使用する代替の`Scheduler`を指定することができます。
つまり、`Task`の実行ループでは常に利用できる`Scheduler`が使用されますが、特定の操作では別のスケジューラに処理を振り分けたい場合があります。
例えば、ブロッキングI/O操作を別のスレッドプールで実行したい場合などです。

2つのスレッドプールがあるとします:

```scala mdoc:silent:nest
// デフォルトのScheduler
import monix.execution.Scheduler.Implicits.global

// I/Oに特化したSchedulerの作成
import monix.execution.Scheduler
lazy val io = Scheduler.io(name="my-io")
```
```scala mdoc:reset:invisible
import monix.eval._
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
lazy val io = TestScheduler()
```

そして、何がどのように実行されるかを管理することができます:

```scala mdoc:silent:nest
// デフォルトのSchedulerをforkで上書きする:
val source = Task(println(s"Running on thread: ${Thread.currentThread.getName}"))
val forked = source.executeOn(io)

source.runToFuture
//=> Running on thread: ForkJoinPool-1-worker-1
forked.runToFuture
//=> Running on thread: my-io-4
```

デフォルトの`Scheduler`で別の非同期境界がスケジューリングされない限り、最後に使用されたスケジューラー(スレッドプール)で実行されます。  
この組み合わせで何が起こるかというと:

```scala mdoc:silent:nest
val onFinish = Task.eval(
  println(s"Ends on thread: ${Thread.currentThread.getName}")
)

val cancelable = {
  source.flatMap(_ => forked)
    .doOnFinish(_ => onFinish)
    .runToFuture
}

//=> Running on thread: ForkJoinPool-1-worker-7
//=> Running on thread: my-io-1
//=> Ends on thread: my-io-1
```

しかし、別の非同期バウンダリを挿入するとデフォルトに戻ります:

```scala mdoc:silent:nest
val asyncBoundary = Task.unit.executeAsync
val onFinish = Task.eval(
  println(s"Ends on thread: ${Thread.currentThread.getName}"))

val cancelable = {
  source // executes on global
    .flatMap(_ => forked) // executes on io
    .flatMap(_ => asyncBoundary) // switch back to global
    .doOnFinish(_ => onFinish) // executes on global
    .runToFuture
}

//=> Running on thread: ForkJoinPool-1-worker-5
//=> Running on thread: my-io-2
//=> Ends on thread: ForkJoinPool-1-worker-5
```

しかし、`Task`にはこのようなトリックを手動で行わなくても、非同期境界を導入するための便利なメソッド`Task.asyncBoundary`があります:

```scala mdoc:silent:nest
val task = {
  source // globalで実行
    .flatMap(_ => forked) // ioで実行
    .asyncBoundary // globalに戻す
    .doOnFinish(_ => onFinish) // globalで実行
    .runToFuture
}
```

Schedulerのオーバーライドは一度しかできないことに注意してください。
`Task`インスタンスは不変なので、次のようにはなりません。
なぜなら、`forked`インスタンスでは`Scheduler`はすでに決まったものが設定されているからです:

```scala mdoc:silent:nest
// Trying to execute on global
forked.executeOn(global).runToFuture
//=> Running on thread: my-io-4
```

**助言:** ブロッキングI/Oを行っていない限り、デフォルトのスレッドプールを使い続けます。`global`が良いデフォルトです。
ブロッキングI/Oの場合は、2つ目のスレッドプールを使用しても問題ありません。しかし、それらのI/O操作を分離し、実際のI/O操作のためにのみスケジューラをオーバーライドします。

### Task.raiseError

`Task.raiseError`は、`Task`のモナディックコンテキストでエラーをリフトすることができます:

```scala mdoc:silent:nest
import scala.concurrent.TimeoutException

val error = Task.raiseError[Int](new TimeoutException)
// error: monix.eval.Task[Int] =
//   Delay(Error(java.util.concurrent.TimeoutException))

error.runAsync(result => println(result))
//=> Left(java.util.concurrent.TimeoutException)
```

### Task.never

`Task.never`は、決して完了しない`Task`インスタンスを返します:

```scala mdoc:silent:nest
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

// 終了しないTaskインスタンス
val never = Task.never[Int]

val timedOut = never.timeoutTo(3.seconds,
  Task.raiseError(new TimeoutException))

timedOut.runAsync(r => println(r))
// 3秒後:
// => Left(java.util.concurrent.TimeoutException)
```

このインスタンスは共有されているので、ガベージコレクターのストレスを軽減することができます。

### Task.unit

`Task.unit`は、すでに完了した`Task[Unit]`インスタンスを返します。
これはユーティリティとして提供されており、`Task.now(())`で新しいインスタンスを作成しなくて済むようになっています:

```scala mdoc:silent:nest
val task = Task.unit
// task: monix.eval.Task[Unit] = Delay(Now(()))
```

このインスタンスは共有されているので、ガベージコレクターのストレスを軽減することができます。

## 非同期型ビルダー

あらゆる非同期APIを使って、`Task`を作ることができます。
安全でないバージョンと、細かい処理を自動的に行うセーフバージョンがあります。
これは、細かい作業を自動的に処理するものです。

### Task.create

`Task.create`は、Scalazをお使いならおなじみの`Task.async`と同じです。コールバックベースのAPIを使って非同期の`Task`を作成することができます。  
例えば、与えられた遅延時間で式を評価するユーティリティを作ってみましょう:

```scala mdoc:silent:nest
import scala.util.Try
import concurrent.duration._

def evalDelayed[A](delay: FiniteDuration)
  (f: => A): Task[A] = {

  // 実行時にはスケジューラーとコールバックが注入されます ;-)
  Task.create { (scheduler, callback) =>
    val cancelable =
      scheduler.scheduleOnce(delay) {
        callback(Try(f))
      }

    // 非同期の計算をキャンセルする次のようなものを返さなければなりません
    cancelable
  }
}
```

そして、[Task.fromFuture](#taskfromfuture)を実装する場合の可能性として、以下のようなものを自分自身で実装します:

```scala mdoc:silent:nest
import monix.execution.Cancelable
import scala.concurrent.Future
import scala.util.{Success, Failure}

def fromFuture[A](f: Future[A]): Task[A] =
  Task.create { (scheduler, callback) =>
    f.onComplete({
      case Success(value) =>
        callback.onSuccess(value)
      case Failure(ex) =>
        callback.onError(ex)
    })(scheduler)

    // ScalaのFutureたちはキャンセルできないので、
    // キャンセルできるかのように振る舞うべきではありません！
    Cancelable.empty
  }
```

いくつかの注意点:

- このビルダーで作成されたタスクは、非同期に(別の論理スレッドで)実行されることが保証されています。
- [Scheduler](../execution/scheduler.md)が注入され、それによって非同期実行のために物事をスケジュールしたり、遅延させたりすることができます。
- しかし、前述のとおりこのコールバックはすでに非同期に実行されるので、用意された`Scheduler`で実行するように明示的にスケジュールする必要はありません。
- [コールバック](../execution/callback.md)は実行時に注入され、そのコールバックには契約があります。
  具体的には、`onSuccess`、`onError`、`apply`を一度だけ実行する必要があります。
- 実行された処理が本当に時間内にキャンセルできない場合は、`Cancelable.empty`を返しても構いませんが、
  できれば実行をキャンセルするような可能性があれば`Cancelable`を返すように努めるべきです。


## メモ化 

[Task#memoize]({{ page.path | api_base_url }}monix/eval/Task.html#memoize:monix.eval.Task[A])メソッドは、
任意の`Task`を受け取り、最初の`runToFuture`でメモ化を適用することができます:

1. 冪等性が保証されている場合、`runToFuture`を複数回呼び出しても、1回呼び出したのと同じ効果が得られます。
2. 後続の`runToFuture`呼び出しは、最初の`runToFuture`で計算された結果を再利用します。

つまり、`memoize`は、最初の`runToFuture`の結果を効果的にキャッシュするのです。  
実際にはこう言えます:

```scala
Task.evalOnce(f) <-> Task.eval(f).memoize
```

これらは事実上同じで、`memoize`は以下のように動作します:

```scala mdoc:silent:nest
// 非同期実行で、.applyのセマンティクスを行う
val task = Task { println("Effect"); "Hello!" }

val memoized = task.memoize

memoized.runToFuture.foreach(println)
//=> Effect
//=> Hello!

memoized.runToFuture.foreach(println)
//=> Hello!
```

### 成功したときだけメモ化する

時には、冪等性を保証した上で、成功した値だけをメモ化したい場合があります。
失敗した場合には 成功した値が得られるまで再試行したい場合があります。

そこで便利なのが`memoizeOnSuccess`メソッドです:

```scala mdoc:silent:nest
var effect = 0

val source = Task.eval {
  effect += 1
  if (effect < 3) throw new RuntimeException("dummy") else effect
}

val cached = source.memoizeOnSuccess

val f1 = cached.runToFuture // RuntimeException が生じる
val f2 = cached.runToFuture // RuntimeException が生じる
val f3 = cached.runToFuture // 3 になる
val f4 = cached.runToFuture // 3 になる
```

### MemoizeとrunToFutureの比較

次の例を見てみましょう:

```scala mdoc:silent:nest
val task = Task { println("Effect"); "Hello!" }
val future = task.runToFuture
```

この`Future`インスタンスは、最初の`runToFuture`実行のメモ化された値で、他の`onComplete`サブスクライバで再利用できます。

この違いは、`Task`と`Future`の違いと同じです。
`memoize`の操作は遅延で、最初の`runToFuture`でのみ評価が行われます。
一方、`runToFuture`の結果は先行です。

## 操作

### FlatMapと末尾再帰ループ

まず、フィボナッチ数列のN番目の数を計算する簡単な例を見てみましょう:

```scala mdoc:silent:nest
import scala.annotation.tailrec

@tailrec
def fib(cycles: Int, a: BigInt, b: BigInt): BigInt = {
 if (cycles > 0)
   fib(cycles-1, b, a + b)
 else
   b
}
```

これは末尾再帰的であることが必要で、そのためには[@tailrec](http://www.scala-lang.org/api/current/index.html#scala.annotation.tailrec)アノテーションを使用していますが，これはScalaの標準ライブラリにあります。
そして，もしこれを`Task`で記述すると、次のような実装が可能です:

```scala mdoc:silent:nest
def fib(cycles: Int, a: BigInt, b: BigInt): Task[BigInt] = {
 if (cycles > 0)
   Task.defer(fib(cycles-1, b, a+b))
 else
   Task.now(b)
}
```

そして今、すでに違いがあります。N番目のフィボナッチ数は、`runToFuture`するまで計算されないので、これは遅延です。
またこれはスタック(とヒープ)セーフなので、`@tailrec`アノテーションも必要ありません。

`Task`は`flatMap`を持っており、これは単項演算の`bind`です。
これは、`Task`や`Future`のように、再帰性や順序付けを記述する操作です。(例：これを実行してからあれを実行して、あれを実行する)
そして、再帰的な呼び出しを記述するために使うことができます:

```scala mdoc:silent:nest
def fib(cycles: Int, a: BigInt, b: BigInt): Task[BigInt] =
  Task.eval(cycles > 0).flatMap {
    case true =>
      fib(cycles-1, b, a+b)
    case false =>
      Task.now(b)
  }
```

繰り返しになりますが、これはスタックセーフで一定量のメモリを使用します。`tailrec`アノテーションは必要ありません。
また、これは遅延な動作をします。なぜなら、`runToFuture`が発生するまで、何もトリガーされないからです。

でも、**mutually tail-recursive calls**を持つこともできます。w00t!

```scala mdoc:silent:nest
// 相互末尾再帰, 最高!!!
{
  def odd(n: Int): Task[Boolean] =
    Task.eval(n == 0).flatMap {
      case true => Task.now(false)
      case false => even(n - 1)
    }

  def even(n: Int): Task[Boolean] =
    Task.eval(n == 0).flatMap {
      case true => Task.now(true)
      case false => odd(n - 1)
    }

  even(1000000)
}
```

繰り返しになりますが、これはスタックセーフで一定量のメモリを使用します。
そして何よりも [実行モデル](../execution/scheduler.md#execution-model)のおかげで、デフォルトではこれらのループは現在のスレッドを永遠にブロックしません。バッチでの実行を好みます。

### 並列性 (cats.Parallel)

`flatMap`を使っていると、よくこんなことになってしまいます:

```scala mdoc:silent:nest
val locationTask: Task[String] = Task.eval(???)
val phoneTask: Task[String] = Task.eval(???)
val addressTask: Task[String] = Task.eval(???)

// flatMapに基づく順序付け操作 ...
val aggregate = for {
  location <- locationTask
  phone <- phoneTask
  address <- addressTask
} yield {
  "Gotcha!"
}
```

ここでの問題は、これらの操作が順番に実行されるということです。
これはScalaの標準的な`Future`でも起こることで、時には望ましくない効果をもたらすこともありますが，`Task`は遅延的に評価されるためこの効果は`Task`でより顕著になります。

しかし、`Task`には[cats.Parallel](https://typelevel.org/cats/typeclasses/parallel.html)という実装があり、複数のタスクの評価を並行して行うことができます。
そのため、`parZip2`、`parZip3`などのユーティリティがあり、現在では`parZip6`まで実装されています。
上記の例は，次のように書くことができます:

```scala mdoc:silent:nest
val locationTask: Task[String] = Task.eval(???)
val phoneTask: Task[String] = Task.eval(???)
val addressTask: Task[String] = Task.eval(???)

// 並行して実行される可能性がある 
val aggregate =
  Task.parZip3(locationTask, phoneTask, addressTask).map {
    case (location, phone, address) => "Gotcha!"
  }
```

タプルへのボックス化を避けるために、`parMap2`を使うこともできます。  
`parMap3`... `parMap6`:

```scala mdoc:silent:nest
Task.parMap3(locationTask, phoneTask, addressTask) {
  (location, phone, address) => "Gotcha!"
}
```

また、`parMapN`にはCatsの構文が使えます:

```scala mdoc:silent:nest
import cats.syntax.all._

(locationTask, phoneTask, addressTask).parMapN {
  (location, phone, address) => "Gotcha!"
}
```

[cats.Parallel](https://typelevel.org/cats/typeclasses/parallel.html) も併せてご覧ください。

### タスクのシーケンスから結果を収集する

`Task.sequence`は、`Seq[Task[A]]`を取り、`Task[Seq[A]]`を返します。
このようにして、任意のタスクのシーケンスを順序付けられた効果と結果を持つタスクに変換します。
つまり、タスクが並列に実行されることはありません。

```scala mdoc:silent:nest
val ta = Task { println("Effect1"); 1 }
val tb = Task { println("Effect2"); 2 }

val list: Task[Seq[Int]] =
  Task.sequence(Seq(ta, tb))

// 常に同じ順番になります:
list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> List(1, 2)
```

結果は初期シーケンスの順に並んでいるので、上の例ではまず`ta`(最初のタスク)の結果が得られ、次に`tb`(2番目のタスク)の結果が得られることが保証されていることになります。
また、実行自体にも順序があるので、`ta`が`tb`の前に実行され、完了します。

`Task.parSequence`は、`Parallel.parSequence`と同様に、`Task.sequence`の非決定論的なバージョンです。
`Seq[Task[A]]`を受け取り、`Task[Seq[A]]`を返すことで、タスクのシーケンスを順番に並べられた結果のシーケンスを持つタスクに変換します。
しかし、結果は順序付けられていません。つまり、並列実行の可能性があるということです。

```scala mdoc:silent:nest
import scala.concurrent.duration._

val ta = {
  Task { println("Effect1"); 1 }
    .delayExecution(1.second)
}

val tb = {
  Task { println("Effect2"); 2 }
    .delayExecution(1.second)
}

val list: Task[Seq[Int]] = Task.parSequence(Seq(ta, tb))

list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> List(1, 2)

list.runToFuture.foreach(println)
//=> Effect2
//=> Effect1
//=> List(1, 2)
```

`Task.parSequenceUnordered`は`parSequence`と似ていますが、結果や効果の順序付けができないことを除けば，`parSequence`と同じです。
そのため結果は非常に非決定的ですが、`parSequence`よりも良いパフォーマンスが得られます。

```scala mdoc:silent:nest
import scala.concurrent.duration._

val ta = {
  Task { println("Effect1"); 1 }
    .delayExecution(1.second)
}

val tb = {
  Task { println("Effect2"); 2 }
    .delayExecution(1.second)
}

val list: Task[Seq[Int]] =
  Task.parSequenceUnordered(Seq(ta, tb))

list.runToFuture.foreach(println)
//=> Effect2
//=> Effect1
//=> Seq(2,1)

list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> Seq(1,2)
```

`Task.traverse`は、`Seq[A]`, `f: A => Task[B]`を受け取り、`Task[Seq[B]]`を返します。
これは`Task.sequence`と似ていますが、`f`を使ってそれぞれの`Task`を生成します。

すべての`Task.sequence`セマンティクスは、効果が順序付けられていることを意味し、Taskは
並行して実行されることはありません。

```scala mdoc:silent:nest
def task(i: Int) = Task { println("Effect" + i); i }

val list: Task[Seq[Int]] =
  Task.traverse(Seq(1, 2))(i => task(i))

// We always get this ordering:
list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> List(1, 2)
```

`Task.parTraverse`は、`Parallel.parTraverse`と同様に`Task.traverse`の非決定論的バージョンです。
これもまた、`Seq[A]`, `f: A => Task[B]`を受け取り`Task[Seq[B]]`を返します。
シーケンスの各要素に`f`を適用して、それを`Task`に変換し、結果を収集します。
出力シーケンスの順序は維持されますが、並列実行の可能性があるということです。

```scala mdoc:silent:nest
import scala.concurrent.duration._

def task(i: Int) = 
  Task { println("Effect" + i); i }.delayExecution(1.second)

val list: Task[Seq[Int]] = Task.parTraverse(Seq(1, 2))(i => task(i))

list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> List(1, 2)

list.runToFuture.foreach(println)
//=> Effect2
//=> Effect1
//=> List(1, 2)
```

`parSequenceUnordered`と同様に、`parTraverse`の順序を保証しないバージョンである`parTraverseUnordered`もあります。

**注:** 可能であればCats構文で提供されているメソッドではなく、明示的に`Task`のメソッドを使用することをお勧めします。
Catsのデフォルト実装は他のメソッドから派生したもので、最適化された`Task`バージョンよりもはるかに遅いことがよくあります。

以下に対応表を示します:

|        Monix        |            Cats            |
|:-------------------:|:--------------------------:|
|    Task.sequence    |    Traverse[F].sequence    |
|    Task.traverse    |    Traverse[F].traverse    |
|    Task.parSequence |    Parallel.parSequence    |
|    Task.parTraverse |    Parallel.parTraverse    |

### Race
 
`racePair`オペレーションは、2つの`Task`を並行して実行し、勝者を選びます。

```scala mdoc:silent:nest
import scala.concurrent.duration._

val ta = Task(1 + 1).delayExecution(1.second)
val tb = Task(10).delayExecution(1.second)

val race = Task.racePair(ta, tb).runToFuture.foreach {
  case Left((a, fiber)) =>
    fiber.cancel.flatMap { _ =>
      Task.eval(println(s"A succeeded: $a"))
    }
  case Right((fiber, b)) =>
    fiber.cancel.flatMap { _ =>
      Task.eval(println(s"B succeeded: $b"))
    }
}
```

生成される結果はタプルの`Either`で、レースに負けたもう一方のタスクに何かをする機会を与えてくれます。
そのタスクをキャンセルするか、その結果を何らかの形で利用するか、あるいは単に無視するかをユースケースによって選択できます。

### Race Many

`raceMany`オペレーションは、タスクのリストを入力として受け取ります。実行すると、最初のタスクが完了してレースに勝利した結果を生成します:

```scala mdoc:silent:nest
import scala.concurrent.duration._

val ta = Task(1 + 1).delayExecution(1.second)
val tb = Task(10).delayExecution(1.second)

{
  Task.raceMany(Seq(ta, tb))
    .runToFuture
    .foreach(r => println(s"Winner: $r"))
}
```

これは、Scalaの[Future.firstCompletedOf](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future$@firstCompletedOf[T(futures:TraversableOnce[scala.concurrent.Future[T])(implicitexecutor:scala.concurrent.ExecutionContext):scala.concurrent.Future[T]) に似ています。
`Task`を操作することを除いて、タスクがレースに勝つと他のタスクが即座にキャンセルされるので、より良いモデルになっています。

エラーを無視して、最初に成功した結果を待ちたい場合には`onErrorHandleWith`や`timeout`と組み合わせることができます:

```scala mdoc:silent:reset
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global
import monix.execution.exceptions.DummyException
import scala.concurrent.duration._

val timeout = 30.second

val task1 = Task.eval(10).delayExecution(3.second)
val task2 = Task.raiseError[Int](DummyException("error")).delayExecution(2.second)
val task3 = Task.raiseError[Int](DummyException("error")).delayExecution(1.second)
val tasks: List[Task[Int]] = List(task1, task2, task3)

val result: Task[Int] = Task.raceMany(tasks.map(_.onErrorHandleWith(_ => Task.never))).timeout(timeout)

println(result.runSyncUnsafe()) // 10をプリントする
```
これにより、失敗したタスクは非終了になります。

タイムアウトは、すべてのタスクが失敗した場合に必要です。上の例では、`task1`も失敗した場合、成功した結果が得られないことがわかっているにもかかわらず、タイムアウトが切れるのを待たなければなりません。

これを最適化するには、`Semaphore`を使った2回目の`race`を行う必要があります:

```scala mdoc:silent:reset
import cats.effect.concurrent.Semaphore
import cats.syntax.apply._
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global
import monix.execution.exceptions.DummyException
import scala.concurrent.duration._

val task1 = Task.raiseError[Int](DummyException("error")).delayExecution(3.second)
val task2 = Task.raiseError[Int](DummyException("error")).delayExecution(2.second)
val task3 = Task.raiseError[Int](DummyException("error")).delayExecution(1.second)
val tasks: List[Task[Int]] = List(task1, task2, task3)

val semaphore = Semaphore[Task](0)

val result: Task[Either[Unit, Int]] = semaphore.flatMap { sem =>
  Task.race(
    sem.acquireN(tasks.length),
    Task.raceMany(tasks.map(_.onErrorHandleWith(_ => sem.release *> Task.never)))
  )
}

println(result.runSyncUnsafe()) // 3秒後にプリントして終了する
```

```scala mdoc:reset:invisible
import monix.execution.CancelableFuture
import monix.eval.Task
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

### 遅延実行

`Task.delayExecution`は、その名のとおり与えられたタスクの実行を与えられた時間だけ遅らせます。

この例では、`source`の実行を3秒遅らせています:

```scala mdoc:silent:nest
import scala.concurrent.duration._

val source = Task {
  println("Side-effect!")
  "Hello, world!"
}

val delayed = source.delayExecution(3.seconds)
delayed.runToFuture.foreach(println)
```

また、遅延の代わりに、別の `Task`を実行開始のシグナルとして使用したい場合もあります。  
上の例と同じです:

```scala mdoc:silent:nest
val source = Task {
  println("Side-effect!")
  "Hello, world!"
}

Task.unit
  .delayExecution(3.seconds)
  .flatMap(_ => source)
```

### 結果の遅延シグナリング

`Task.delayResult`は結果の通知を遅らせますが、`Task`の実行は遅らせません。  
この例を考えてみましょう:

```scala mdoc:silent:nest
import scala.concurrent.duration._

val source = Task {
  println("Side-effect!")
  "Hello, world!"
}

val delayed = {
  source
    .delayExecution(1.second)
    .delayResult(5.seconds)
}

delayed.runToFuture.foreach(println)
```

ここでは、`副作用はわずか1秒後に起こるが、結果の通知はさらに5秒後に起こる`ことがわかります。

### 述語が真になるまで再起動

`Task`は仕様なので、自由に再起動することができます。  
`Task.restartUntil(predicate)`はまさに、述語が`true`になるまで何度も再実行します:

```scala mdoc:silent:nest
import scala.util.Random

val randomEven = {
  Task.eval(Random.nextInt())
    .restartUntil(_ % 2 == 0)
}

randomEven.runToFuture.foreach(println)
//=> -2097793116
randomEven.runToFuture.foreach(println)
//=> 1246761488
randomEven.runToFuture.foreach(println)
//=> 1053678416
```

### 終了時にリソースをのクリーンアップ

`Task.doOnFinish`は、`Option[Throwable] => Task[Unit]`が終了したときに提供された関数を実行します。  
これはリソースのクリーンアップや予定されている副作用の実行を意味します:

```scala mdoc:silent:nest
val task = Task(1)

val withFinishCb = task.doOnFinish {
  case None =>
    println("Was success!")
    Task.unit
  case Some(ex) =>
    println(s"Had failure: $ex")
    Task.unit
}

withFinishCb.runToFuture.foreach(println)
//=> Was success!
//=> 1
```

### リアクティブ・パブリッシャーへの変換

Monixが[Reactive Streams](http://www.reactive-streams.org/)の仕様に統合されていることをご存知ですか？

さて、`Task`は`org.reactivestreams.Publisher`として見ることができます。サブスクリプション時に正確に1つのイベントを発行し、その後停止します。  
そして、私たちは任意の`Task`を直接そのようなパブリッシャーに変換することができます:

```scala mdoc:silent:nest
val task = Task.eval(Random.nextInt())

val publisher: org.reactivestreams.Publisher[Int] =
  task.toReactivePublisher
```

これは他のライブラリとの相互運用を目的としていますが、直接使いたい場合は少し低レベルです。  
でも可能です:

```scala mdoc:silent:nest
import org.reactivestreams._

publisher.subscribe(new Subscriber[Int] {
  def onSubscribe(s: Subscription): Unit =
    s.request(Long.MaxValue)

  def onNext(e: Int): Unit =
    println(s"OnNext: $e")

  def onComplete(): Unit =
    println("OnComplete")

  def onError(ex: Throwable): Unit =
    System.err.println(s"ERROR: $ex")
})

// 次のようにプリントされる:
//=> OnNext: -228329246
//=> OnComplete
```

凄いでしょう？

(◑‿◐)

## エラーハンドリング

`Task`はエラー処理を非常に重要視しています。
ほら、*観察* についてこんな有名な話があるじゃないですか。  
[思考実験](https://en.wikipedia.org/wiki/If_a_tree_falls_in_a_forest):

> "*森の中で木が倒れても、周りにそれを聞く人がいなければ
> その木は音を発しますか？*"

これは、エラー処理に非常によく当てはまります。非同期プロセスでエラーが発生しても、それを聞く人がいなければキャッチしてログに残したり、回復したりすることはしません。
そして得られるのは[非決定論](https://en.wikipedia.org/wiki/Nondeterministic_algorithm)のようなもので、エラーのヒントはありません。

Monixは常に発生したエラーをキャッチしてシグナルを送るか、少なくともログに記録します。
何らかの理由でシグナリングができない場合(コールバックがすでに呼び出されているなど)、ロギングは提供されている`Scheduler.reportFailure`を使って行われます。
SLF4Jを経由するなど、より具体的なものを提供しない限りはデフォルトで`System.err`となります。

Monixはそのメソッドに与えられる引数が`flatMap`のように純粋であるか、少なくともエラーから保護されていることを期待し、エラーをキャッチして`runAsync`でシグナルを送ります:

```scala mdoc:silent:nest
val task = Task(Random.nextInt()).flatMap {
  case even if even % 2 == 0 =>
    Task.now(even)
  case odd =>
    throw new IllegalStateException(odd.toString)
}

task.runAsync(r => println(r))
//=> Right(-924040280)

task.runAsync(r => println(r))
//=> Left(java.lang.IllegalStateException: 834919637)
```

`runAsync`に提供されたコールバックでエラーが発生した場合、以下のようになります。
Monixはもはや`onError`のシグナルを出すことはできません。契約違反になるからです([Callback](../execution/callback.md)を参照)。  
しかし、それはまだエラーを記録します:

```scala mdoc:silent:nest
import scala.concurrent.duration._

// 非同期の実行を保証
// 現在のスレッドではアクションは起こらない
val task = Task(2).delayExecution(1.second)

task.runAsync { r =>
  throw new IllegalStateException(r.toString)
}

// 1秒後に、スタックトレース全体をログに記録します:
//=> java.lang.IllegalStateException: Right(2)
//=>    ...
//=>	at monix.eval.Task$$anon$3.onSuccess(Task.scala:78)
//=>	at monix.eval.Callback$SafeCallback.onSuccess(Callback.scala:66)
//=>	at monix.eval.Task$.trampoline$1(Task.scala:1248)
//=>	at monix.eval.Task$.monix$eval$Task$$resume(Task.scala:1304)
//=>	at monix.eval.Task$AsyncStateRunnable$$anon$20.onSuccess(Task.scala:1432)
//=>    ....
```

同様に`Task.create`を使用した場合、Monixはエラーを捕捉しようとしますが、提供されたコールバックで何が起こったのかわからなかったため契約違反となるエラーを通知できません。([Callback](../execution/callback.md)を参照) 
Monixは以下のようにエラーを記録します:

```scala mdoc:silent:nest
val task = Task.create[Int] { (scheduler, callback) =>
  (throw new IllegalStateException("FTW!")): Unit
}

val future = task.runToFuture

// 以下の内容をSystem.errに記録します:
//=> java.lang.IllegalStateException: FTW!
//=>    ...
//=> 	at monix.eval.Task$$anonfun$create$1.apply(Task.scala:576)
//=> 	at monix.eval.Task$$anonfun$create$1.apply(Task.scala:571)
//=> 	at monix.eval.Task$AsyncStateRunnable.run(Task.scala:1429)
//=>    ...

// このFutureは絶対に完了しません。うわー!
future.onComplete(r => println(r))
```

**警告:** この場合、コンシューマーはシグナルを受け取ることはありません。
この話の教訓は、Monixが正しいことをするために最善の努力をしたとしても、特に`Task.create`では不要な例外からコードを保護すべきです!!!

### エラーロギングのオーバーライド

[Scheduler](../execution/scheduler.md)の記事には、独自のロジックで独自の`Scheduler`インスタンスを構築するためのレシピが掲載されています。
しかしここでは、そのような`Scheduler`を構築するための簡単なスニペットを紹介します。標準的な[SLF4J](http://www.slf4j.org/)のようなライブラリを使って、ロギングを行います:

```scala mdoc:silent:nest
import monix.execution.Scheduler
import monix.execution.Scheduler.{global => default}
import monix.execution.UncaughtExceptionReporter
import org.slf4j.LoggerFactory

val reporter = UncaughtExceptionReporter { ex =>
  val logger = LoggerFactory.getLogger("monix")
  logger.error("Uncaught exception", ex)
}

implicit val global: Scheduler =
  Scheduler(default, reporter)
```

### タイムアウトのトリガー

`Task`の実行に時間がかかりすぎた場合には、`Task.timeout`を使って`TimeoutException`を発生させることができます:

```scala mdoc:silent:nest
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

val source =
  Task("Hello!").delayExecution(10.seconds)

// runAsyncの後、3秒以内にソースが完了しないとエラーを起こす
val timedOut = source.timeout(3.seconds)

timedOut.runAsync(r => println(r))
//=> Failure(TimeoutException)
```

タイムアウトになると、ソースはキャンセルされます(キャンセルをサポートしているソースの場合)。
また、エラーの代わりに`fallback`タスクにタイムアウトすることもできます。
次の例は、上の例と同じです:

```scala mdoc:silent:nest
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

val source =
  Task("Hello!").delayExecution(10.seconds)

val timedOut = source.timeoutTo(
  3.seconds,
  Task.raiseError(new TimeoutException)
)

timedOut.runAsync(r => println(r))
//=> Left(TimeoutException)
```

### エラーからの回復

`Task.onErrorHandleWith`は、起こりうる例外を望ましいフォールバックの結果に例外をマッピングする関数を取る操作です:

```scala mdoc:silent:nest
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

val source = {
  Task("Hello!")
    .delayExecution(10.seconds)
    .timeout(3.seconds)
}

val recovered = source.onErrorHandleWith {
  case _: TimeoutException =>
    // あぁ、タイムアウトのことを知ってるから回復するよ
    Task.now("Recovered!")
  case other =>
    // 何が起こったのかわからないから、エラーを上げる！
    Task.raiseError(other)
}

recovered.runToFuture.foreach(println)
//=> Recovered!
```

部分関数を受け取る`Task.onErrorRecoverWith`もあるので、`other`の部分を省略することができます:

```scala mdoc:silent:nest
val recovered = source.onErrorRecoverWith {
  case _: TimeoutException =>
    // あぁ、タイムアウトのことを知ってるから回復するよ
    Task.now("Recovered!")
}

recovered.runToFuture.foreach(println)
//=> Recovered!
```

`Task.onErrorHandleWith`と`Task.onErrorRecoverWith`は、エラーの場合に限り`flatMap`に相当します。
フォールバックの結果を遅延で評価できることがわかっている場合には、ショートカットの操作`Task.onErrorHandle`を使用することができます:

```scala mdoc:silent:nest
val recovered = source.onErrorHandle {
  case _: TimeoutException =>
    // あぁ、タイムアウトのことを知ってるから回復するよ
    "Recovered!"
  case other =>
    throw other // 再スロー
}
```

または、`onErrorRecover`を使った部分関数バージョン:

```scala mdoc:silent:nest
val recovered = source.onErrorRecover {
  case _: TimeoutException =>
    // あぁ、タイムアウトのことを知ってるから回復するよ
    "Recovered!"
}
```

### エラーのときに再起動する

`タスク`タイプは単なる仕様であるため、通常は最終的な結果を得るためのあらゆるプロセスを再起動することができます。
また、エラーが発生した場合には必要な回数だけソースを再起動することができます:

```scala mdoc:silent:nest
import scala.util.Random

val source = Task(Random.nextInt()).flatMap {
  case even if even % 2 == 0 =>
    Task.now(even)
  case other =>
    Task.raiseError(new IllegalStateException(other.toString))
}

// 偶数のランダムな数字に対して4回リトライします
// またはmaxRetriesに達した場合は失敗します！
val randomEven = source.onErrorRestart(maxRetries = 4)
```

また、与えられた述語で再開することもできます:

```scala mdoc:silent:nest
import scala.util.Random

val source = Task(Random.nextInt()).flatMap {
  case even if even % 2 == 0 =>
    Task.now(even)
  case other =>
    Task.raiseError(new IllegalStateException(other.toString))
}

// IllegalStateExceptionが発生する限り、再試行を続けます
val randomEven = source.onErrorRestartIf {
  case _: IllegalStateException => true
  case _ => false
}
```

あるいは、指数関数的なバックオフによる独自のリトライを実装することもできます。  
そうするのがクールだからです:

```scala mdoc:silent:nest
def retryBackoff[A](source: Task[A],
  maxRetries: Int, firstDelay: FiniteDuration): Task[A] = {

  source.onErrorHandleWith {
    case ex: Exception =>
      if (maxRetries > 0)
        // 再帰呼び出しも、Monixはスタック・セーフなので問題なし
        retryBackoff(source, maxRetries-1, firstDelay*2)
          .delayExecution(firstDelay)
      else
        Task.raiseError(ex)
  }
}
```

### エラーを公開する

`Task`のモナディックコンテキストは、Scalaの`Try`や`Future`のように発生したエラーを隠します。
しかし、時にはこれらのエラーを公開してより効率的に回復できるようにしたいこともあります:

```scala mdoc:silent:nest
import scala.util.{Try, Success, Failure}

val source = Task.raiseError[Int](new IllegalStateException)
val materialized: Task[Try[Int]] =
  source.materialize

// これで成功も失敗もフラットにできるようになりました:
val recovered = materialized.flatMap {
  case Success(value) => Task.now(value)
  case Failure(_) => Task.now(0)
}

recovered.runToFuture.foreach(println)
//=> 0
```

また、マテリアライズの逆もあり`Task.dematerialize`となります:

```scala mdoc:silent:nest
import scala.util.Try

val source = Task.raiseError[Int](new IllegalStateException)

// エラーを公開する
val materialized = source.materialize
// materialize: Task[Try[Int]] = ???

// 再びエラーを隠す
val dematerialized = materialized.dematerialize
// dematerialized: Task[Int] = ???
```

また、任意の`Task`を`Task[Throwable]`に変換することもできます。
起こったエラーを公開したり、ソースが成功して完了した場合は`NoSuchElementException`で終了します:

```scala mdoc:silent:nest
val source = Task.raiseError[Int](new IllegalStateException)

val throwable = source.failed
// throwable: Task[Throwable] = ???

throwable.runToFuture.foreach(println)
//=> java.lang.IllegalStateException
```
