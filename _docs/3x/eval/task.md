---
layout: docs3x
title: Task
type_api: monix.eval.Task
type_source: monix-eval/shared/src/main/scala/monix/eval/Task.scala
description: |
  A data type for controlling possibly lazy &amp; asynchronous computations, useful for controlling side-effects, avoiding nondeterminism and callback-hell.
---

## イントロダクション

タスクは、遅延する可能性のある、非同期計算を制御するためのデータ型です。副作用の制御や非決定論やコールバック地獄の回避に役立ちます。

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

Monixの `Task` を要約します:

- 遅延 &amp; 非同期評価
- 1つのプロデューサーが1つまたは複数のコンシューマーに1つの値だけをプッシュするモデル
- [実行モデル](../execution/scheduler.md#execution-model) をきめ細かく制御することができる
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

`Task`はScalaの [Future](http://docs.scala-lang.org/overviews/core/futures.html) と似ていますが違った性格を持っており、この2つのタイプは実際には補完関係にあることがわかります。  
賢い人はかつてこう言いました。

> "*未来は時間から切り離された価値を表す*" &mdash; Viktor Klang

価値とは何か、時間とは何かを考えさせられる詩的な発想ですね。
しかし、より重要なことは、`Future`が [値](https://en.wikipedia.org/wiki/Value_(computer_science)) であるとは言えないものの、*値になりたがっている* とは確実に言えるということです。
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

MonixのTaskが、[Scalaz](https://github.com/scalaz/scalaz) のTaskにインスパイアされたことは周知の事実です。Monixライブラリ全体が巨人の肩の上に立っています。  
Monix Taskの実装が異なる点は:

1. ScalazのTaskは実装の詳細を漏らしています。
   というのも、ScalazのTaskはまず*トランポリン*実行を目的としていますが、非同期実行は非同期のトランポリン境界を飛び越えることを目的としているからです。
   例えば、大きなループで現在のスレッドをブロックしないようにするには、`Task.executeAsync`を使って非同期の境界を手動で挿入しなければなりません。
   一方、MonixのTaskは、デフォルトで自動的にそれを行うようになっています。
   これは、[Javascript](http://www.scala-js.org/) の上で実行するときに非常に便利です。
   [協調的マルチタスク](https://en.wikipedia.org/wiki/Cooperative_multitasking) はあったほうがいいだけでなく、必要なのです。
2. ScalazのTaskは同期/非同期の2面を持っています。これは生産者にとっては最適化のために良いことです(例えば、必要がないのになぜスレッドをフォークするのか)。
   しかし、コンシューマの視点から見ると、`def run: A`は、JavascriptやJVM上でAPIを完全にサポートできないことを意味します。
   つまり、Scalazの`Task` は結局、同期評価やスレッドのブロックを偽装することになるのです。
   そして、[スレッドをブロックすることは非常に安全ではありません](../best-practices/blocking.md) 
3. ScalazのTaskは、実行中の計算をキャンセルすることはできません。これは非決定論的な操作では重要です。
   例えば、`race`で競合状態を作ったときに、時間内に終わらなかった遅いタスクをキャンセルしたい場合があります。
   というのもリソースをすぐに解放しないと、残念ながら深刻なリークが発生してプロセスがクラッシュしてしまうことがあるからです。
4. Scalazタスクは、非同期実行を扱うJavaの標準ライブラリを利用しています。
   これはポータビリティの観点からは好ましくありません。このAPIは[Scala.js](http://www.scala-js.org/) の上ではサポートされていないからです。
   
## 実行 (runToFuture & foreach)

Taskインスタンスは、`runToFuture`によって実行されるまで何もしません。また、複数のオーバーロードがあります。

`Task.runToFuture`では`ExecutionContext`を継承した[Scheduler](../execution/scheduler.md) を暗黙のパラメータとして求められます。
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

**注:** [Scheduler](../execution/scheduler.md) は、非同期の境界がどのように強制されるか(あるいはされないか)を決定する設定可能な[実行モデル](../execution/scheduler.md#execution-model) を注入することができます。

最もわかりやすく慣用的な方法は、タスクを実行して[CancelableFuture]({{ page.path | api_base_url }}monix/execution/CancelableFuture.html) を返すことです。
これは、標準的な`Future`と[Cancelable](../execution/cancelable.md) を組み合わせたものです。

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

また、[Callback](../execution/callback.md) インスタンスを使って`runAsync`することもできます。
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

[Monixはその哲学としてブロッキングに反対](../best-practices/blocking.md) しており、したがって`Task`にはスレッドをブロックするAPIコールは一切ありません！

しかし、JVMの上では時にはブロックしなければならないこともあります。
なぜなら、標準の`Await.result`と`Await.ready`には、2つの健全な設計上の選択があるからです。

1. これらのコールは、Scalaの`BlockContext`を使って実装されています。
   ブロッキング操作が実行されていることを基礎となるスレッドプールにシグナリングし、スレッドプールがそれに対応できるようにします。
   例えばScalaの`ForkJoinPool`がやっているように、プールにスレッドを追加することを決めるかもしれません。
2. これらの呼び出しには、非常に明示的なタイムアウト・パラメータが必要で、それは`FiniteDuration`として指定されます。

したがって、まず `runToFuture`で結果を`Future` に変換してから、結果をブロックすることができます:

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

非同期の可能性があるという性質を受け入れることができれば、`Task`は引数がない関数、Scalaの名前渡しパラメータ、`lazy val`を置き換えることができます。 また、Scala の `Future` はすべて `Task` に変換できます。

### Task.now

`Task.now`は、`Task`コンテキストで既に知られている値をリフトします。
これは`Future.successful`や`Applicative.pure`に相当します:

```scala mdoc:silent:nest
val task = Task.now { println("Effect"); "Hello!" }
//=> Effect
// task: monix.eval.Task[String] = Delay(Now(Hello!))
```

### Task.eval (遅延)

Task.eval`は、`Function0`と同等であり`runToFuture`で可能な限り同じスレッドで評価される関数を受け取ります([選択された実行モデル](../execution/scheduler.md#execution-model) によります)。

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

`Task.evalOnce` は、Scalaでは正確に表現できない型である「lazy val」に相当します。
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

`Task.defer` は、タスクのファクトリを構築するためのものです。例えばこれは、`Task.eval`とほぼ同様の動作をします。

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

`Future` の結果を生成するコールを`Task`にラップし、必要な`ExecutionContext`として動作する`Scheduler`を注入したコールバックを提供します。

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
その違いは、`Task` は`runToFuture`が呼ばれたときにのみ`Scheduler`(`ExecutionContext`を継承したもの) を取るということです。
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
デフォルトでは[実行モデル](../execution/scheduler.md#execution-model) は最初は現在のスレッドでの実行を好みます。

これにより、私たちのタスクが非同期に実行されることが保証されます:

```scala mdoc:silent:nest
val task = Task.eval("Hello!").executeAsync
```

`ExecuteOn`では、使用する代替の`Scheduler`を指定することができます。
つまり、`Task` の実行ループでは常に利用できる`Scheduler`が使用されますが、特定の操作では別のスケジューラに処理を振り分けたい場合があります。
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

`Task.never` は、決して完了しない`Task`インスタンスを返します:

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

そして、[Task.fromFuture](#taskfromfuture) を実装する場合の可能性として、以下のようなものを自分自身で実装します:

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

- Tasks created with this builder are guaranteed to execute
  asynchronously (on another logical thread)
- The [Scheduler](../execution/scheduler.md) gets injected and with
  it we can schedule things for async execution, we can delay,
  etc...
- But as said, this callback will already execute asynchronously, so
  you don't need to explicitly schedule things to run on the provided
  `Scheduler`, unless you really need to do it.
- [The Callback](../execution/callback.md) gets injected on execution and that
  callback has a contract. In particular you need to execute
  `onSuccess` or `onError` or `apply` only once. The implementation
  does a reasonably good job to protect against contract violations,
  but if you do call it multiple times, then you're doing it risking
  undefined and nondeterministic behavior.
- It's OK to return a `Cancelable.empty` in case the executed
  process really can't be canceled in time, but you should strive to
  return a cancelable that does cancel your execution, if possible.


## Memoization

The
[Task#memoize]({{ page.path | api_base_url }}monix/eval/Task.html#memoize:monix.eval.Task[A])
operator can take any `Task` and apply memoization on the first `runToFuture`,
such that:

1. you have guaranteed idempotency, calling `runToFuture` multiple times
   will have the same effect as calling it once
2. subsequent `runToFuture` calls will reuse the result computed by the
   first `runToFuture`

So `memoize` effectively caches the result of the first `runToFuture`.
In fact we can say that:

```scala
Task.evalOnce(f) <-> Task.eval(f).memoize
```

They are effectively the same.  And `memoize` works
with any task reference:

```scala mdoc:silent:nest
// Has async execution, to do the .apply semantics
val task = Task { println("Effect"); "Hello!" }

val memoized = task.memoize

memoized.runToFuture.foreach(println)
//=> Effect
//=> Hello!

memoized.runToFuture.foreach(println)
//=> Hello!
```

### Memoize Only on Success

Sometimes you just want memoization, along with idempotency
guarantees, only for successful values. For failures you might want to
keep retrying until a successful value is available.

This is where the `memoizeOnSuccess` operator comes in handy:

```scala mdoc:silent:nest
var effect = 0

val source = Task.eval {
  effect += 1
  if (effect < 3) throw new RuntimeException("dummy") else effect
}

val cached = source.memoizeOnSuccess

val f1 = cached.runToFuture // yields RuntimeException
val f2 = cached.runToFuture // yields RuntimeException
val f3 = cached.runToFuture // yields 3
val f4 = cached.runToFuture // yields 3
```

### Memoize versus runToFuture

You can say that when we do this:

```scala mdoc:silent:nest
val task = Task { println("Effect"); "Hello!" }
val future = task.runToFuture
```

That `future` instance is also going to be a memoized value of the
first `runToFuture` execution, which can be reused for other `onComplete`
subscribers.

The difference is the same as the difference between `Task` and
`Future`. The `memoize` operation is lazy, evaluation only being
triggered on the first `runToFuture`, whereas the result of `runToFuture` is
eager.

## Operations

### FlatMap and Tail-Recursive Loops

So let's start with a simple example that calculates the N-th number in
the Fibonacci sequence:

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

We need this to be tail-recursive, hence the use of the
[@tailrec](http://www.scala-lang.org/api/current/index.html#scala.annotation.tailrec)
annotation from Scala's standard library. And if we'd describe it with
`Task`, one possible implementation would be:

```scala mdoc:silent:nest
def fib(cycles: Int, a: BigInt, b: BigInt): Task[BigInt] = {
 if (cycles > 0)
   Task.defer(fib(cycles-1, b, a+b))
 else
   Task.now(b)
}
```

And now there are already differences. This is lazy, as the N-th
Fibonacci number won't get calculated until we `runToFuture`. The
`@tailrec` annotation is also not needed, as this is stack (and heap)
safe.

`Task` has `flatMap`, which is the monadic `bind` operation, that for
things like `Task` and `Future` is the operation that describes
recursivity or that forces ordering (e.g. execute this, then that,
then that). And we can use it to describe recursive calls:

```scala mdoc:silent:nest
def fib(cycles: Int, a: BigInt, b: BigInt): Task[BigInt] =
  Task.eval(cycles > 0).flatMap {
    case true =>
      fib(cycles-1, b, a+b)
    case false =>
      Task.now(b)
  }
```

Again, this is stack safe and uses a constant amount of memory, so no
`@tailrec` annotation is needed or wanted. And it has lazy behavior,
as nothing will get triggered until `runToFuture` happens.

But we can also have **mutually tail-recursive calls**, w00t!

```scala mdoc:silent:nest
// Mutual Tail Recursion, ftw!!!
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

Again, this is stack safe and uses a constant amount of memory.  And
best of all, because of the
[execution model](../execution/scheduler.md#execution-model), by
default these loops won't block the current thread forever, preferring to
execute things in batches.

### Parallelism (cats.Parallel)

When using `flatMap`, we often end up with this:

```scala mdoc:silent:nest
val locationTask: Task[String] = Task.eval(???)
val phoneTask: Task[String] = Task.eval(???)
val addressTask: Task[String] = Task.eval(???)

// Ordered operations based on flatMap ...
val aggregate = for {
  location <- locationTask
  phone <- phoneTask
  address <- addressTask
} yield {
  "Gotcha!"
}
```

For one the problem here is that these operations are executed in
order. This also happens with Scala's standard `Future`, being
sometimes an unwanted effect, but because `Task` is lazily evaluated,
this effect is even more pronounced with `Task`.

But `Task` also has a
[cats.Parallel](https://typelevel.org/cats/typeclasses/parallel.html)
implementation, being able to trigger evaluation of multiple
tasks in parallel and hence it has utilities, such
as `parZip2`, `parZip3`, up until `parZip6` (at the moment of writing). 
The example above could be written as:

```scala mdoc:silent:nest
val locationTask: Task[String] = Task.eval(???)
val phoneTask: Task[String] = Task.eval(???)
val addressTask: Task[String] = Task.eval(???)

// Potentially executed in parallel
val aggregate =
  Task.parZip3(locationTask, phoneTask, addressTask).map {
    case (location, phone, address) => "Gotcha!"
  }
```

In order to avoid boxing into tuples, you can also use `parMap2`,
`parMap3` ... `parMap6`:

```scala mdoc:silent:nest
Task.parMap3(locationTask, phoneTask, addressTask) {
  (location, phone, address) => "Gotcha!"
}
```

And you can use Cats' syntax for `parMapN`:

```scala mdoc:silent:nest
import cats.syntax.all._

(locationTask, phoneTask, addressTask).parMapN {
  (location, phone, address) => "Gotcha!"
}
```

Also see the documentation for
[cats.Parallel](https://typelevel.org/cats/typeclasses/parallel.html).

### Gather results from a Seq of Tasks

`Task.sequence`, takes a `Seq[Task[A]]` and returns a `Task[Seq[A]]`,
thus transforming any sequence of tasks into a task with a sequence of
results and with ordered effects and results. This means that the
tasks WILL NOT execute in parallel.

```scala mdoc:silent:nest
val ta = Task { println("Effect1"); 1 }
val tb = Task { println("Effect2"); 2 }

val list: Task[Seq[Int]] =
  Task.sequence(Seq(ta, tb))

// We always get this ordering:
list.runToFuture.foreach(println)
//=> Effect1
//=> Effect2
//=> List(1, 2)
```

The results are ordered in the order of the initial sequence, so that
means in the example above we are guaranteed in the result to first
get the result of `ta` (the first task) and then the result of `tb`
(the second task). The execution itself is also ordered, so `ta`
executes and completes before `tb`.

`Task.parSequence`, similar to `Parallel.parSequence`, is the nondeterministic
version of `Task.sequence`.  It also takes a `Seq[Task[A]]` and
returns a `Task[Seq[A]]`, thus transforming any sequence of tasks into
a task with a sequence of ordered results. But the effects are not
ordered, meaning that there's potential for parallel execution:

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

`Task.parSequenceUnordered` is like `parSequence`, except that you don't get
ordering for results or effects. The result is thus highly nondeterministic,
but yields better performance than `parSequence`:

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

`Task.traverse`, takes a `Seq[A]`, `f: A => Task[B]` and returns a `Task[Seq[B]]`.
This is similar to `Task.sequence` but it uses `f` to generate each `Task`.

All `Task.sequence` semantics hold meaning the effects are ordered and the tasks
WILL NOT execute in parallel.

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

`Task.parTraverse`, similar to `Parallel.parTraverse`, is the nondeterministic
version of `Task.traverse`.  It also takes a `Seq[A]`, `f: A => Task[B]` and
returns a `Task[Seq[B]]`. It applies `f` to each element in the sequence transforming it
into `Task` and then collecting results. The order in the output sequence is preserved, but 
the effects are not ordered, meaning that there's potential for parallel execution:

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

Similar to `parSequenceUnordered` there is also unordered version of `parTraverse` called `parTraverseUnordered`.

**NOTE:** If you have the possibility, prefer explicitly using `Task` operators instead of
those provided by Cats syntax. Their default implementations are derived from other
methods and are often much slower than optimized `Task` versions.

Refer to the table below to see corresponding methods:


|        Monix        |            Cats            |
|:-------------------:|:--------------------------:|
|    Task.sequence    |    Traverse[F].sequence    |
|    Task.traverse    |    Traverse[F].traverse    |
|    Task.parSequence |    Parallel.parSequence    |
|    Task.parTraverse |    Parallel.parTraverse    |

### Race

The `racePair` operation will choose the winner between two
`Task` that will potentially run in parallel:

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

The result generated will be an `Either` of tuples, giving you the
opportunity to do something with the other task that lost the race.
You can cancel it, or you can use its result somehow, or you can
simply ignore it, your choice depending on use-case.

### Race Many

The `raceMany` operation takes as input a list of tasks,
and upon execution will generate the result of the first task
that completes and wins the race:

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

It is similar to Scala's
[Future.firstCompletedOf](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future$@firstCompletedOf[T](futures:TraversableOnce[scala.concurrent.Future[T]])(implicitexecutor:scala.concurrent.ExecutionContext):scala.concurrent.Future[T])
operation, except that it operates on `Task` and upon execution it has
a better model, as when a task wins the race the other tasks get
immediately canceled.

If you want to ignore errors and wait for the first successful result you could 
combine it with `onErrorHandleWith` and `timeout`:

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

println(result.runSyncUnsafe()) // will print 10
```
It will turn any failed tasks into non-terminating.

Timeout is necessary in case all tasks fail. In the example above, if `task1` also fails we will have to wait for the timeout
to expire despite knowing that we won't get any successful result.

We can optimize it by doing second `race `that uses `Semaphore`:

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

println(result.runSyncUnsafe()) // will finish and print after 3 seconds
```

```scala mdoc:reset:invisible
import monix.execution.CancelableFuture
import monix.eval.Task
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

### Delay Execution

`Task.delayExecution`, as the name says, delays the execution of a
given task by the given timespan.

In this example we are delaying the execution of the source by 3
seconds:

```scala mdoc:silent:nest
import scala.concurrent.duration._

val source = Task {
  println("Side-effect!")
  "Hello, world!"
}

val delayed = source.delayExecution(3.seconds)
delayed.runToFuture.foreach(println)
```

Or, instead of a delay we might want to use another `Task` as the
signal for starting the execution, so the following example is
equivalent to the one above:

```scala mdoc:silent:nest
val source = Task {
  println("Side-effect!")
  "Hello, world!"
}

Task.unit
  .delayExecution(3.seconds)
  .flatMap(_ => source)
```

### Delay Signaling of the Result

`Task.delayResult` delays the signaling of the result, but not the
execution of the `Task`. Consider this example:

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

Here, you'll see the "side-effect happening after only 1 second, but
the signaling of the result will happen after another 5 seconds.

### Restart Until Predicate is True

The `Task` being a spec, we can restart it at will.
`Task.restartUntil(predicate)` does just that, executing the source
over and over again, until the given predicate is true:

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

### Clean-up Resources on Finish

`Task.doOnFinish` executes the supplied
`Option[Throwable] => Task[Unit]` function when the source finishes,
being meant for cleaning up resources or executing
some scheduled side-effect:

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

### Convert to Reactive Publisher

Did you know that Monix integrates with the
[Reactive Streams](http://www.reactive-streams.org/)
specification?

Well, `Task` can be seen as an `org.reactivestreams.Publisher` that
emits exactly one event upon subscription and then stops. And we can
convert any `Task` to such a publisher directly:

```scala mdoc:silent:nest
val task = Task.eval(Random.nextInt())

val publisher: org.reactivestreams.Publisher[Int] =
  task.toReactivePublisher
```

This is meant for interoperability purposes with other libraries, but
if you're inclined to use it directly, it's a little lower level,
but doable:

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

// Will print:
//=> OnNext: -228329246
//=> OnComplete
```

Awesome, isn't it?

(◑‿◐)

## Error Handling

`Task` takes error handling very seriously. You see, there's this famous
[thought experiment](https://en.wikipedia.org/wiki/If_a_tree_falls_in_a_forest)
regarding *observation*:

> "*If a tree falls in a forest and no one is around to hear it, does
> it make a sound?*"

Now this applies very well to error handling, because if an error is
triggered by an asynchronous process and there's nobody to hear it, no
handler to catch it and log it or recover from it, then it didn't
happen. And what you'll get is
[nondeterminism](https://en.wikipedia.org/wiki/Nondeterministic_algorithm)
without any hints of the error involved.

This is why Monix will always attempt to catch and signal or at least
log any errors that happen. In case signaling is not possible for
whatever reason (like the callback was already called), then the
logging is done by means of the provided `Scheduler.reportFailure`,
which defaults to `System.err`, unless you provide something more
concrete, like going through SLF4J or whatever.

Even though Monix expects for the arguments given to its operators,
like `flatMap`, to be pure or at least protected from errors, it still
catches errors, signaling them on `runAsync`:

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

In case an error happens in the callback provided to `runAsync`, then
Monix can no longer signal an `onError`, because it would be a
contract violation (see [Callback](../execution/callback.md)). But it still
logs the error:

```scala mdoc:silent:nest
import scala.concurrent.duration._

// Ensures asynchronous execution, just to show
// that the action doesn't happen on the
// current thread
val task = Task(2).delayExecution(1.second)

task.runAsync { r =>
  throw new IllegalStateException(r.toString)
}

// After 1 second, this will log the whole stack trace:
//=> java.lang.IllegalStateException: Right(2)
//=>    ...
//=>	at monix.eval.Task$$anon$3.onSuccess(Task.scala:78)
//=>	at monix.eval.Callback$SafeCallback.onSuccess(Callback.scala:66)
//=>	at monix.eval.Task$.trampoline$1(Task.scala:1248)
//=>	at monix.eval.Task$.monix$eval$Task$$resume(Task.scala:1304)
//=>	at monix.eval.Task$AsyncStateRunnable$$anon$20.onSuccess(Task.scala:1432)
//=>    ....
```

Similarly, when using `Task.create`, Monix attempts to catch any
uncaught errors, but because we did not know what happened in the
provided callback, we cannot signal the error as it would be a
contract violation (see [Callback](../execution/callback.md)), but Monix does
log the error:

```scala mdoc:silent:nest
val task = Task.create[Int] { (scheduler, callback) =>
  (throw new IllegalStateException("FTW!")): Unit
}

val future = task.runToFuture

// Logs the following to System.err:
//=> java.lang.IllegalStateException: FTW!
//=>    ...
//=> 	at monix.eval.Task$$anonfun$create$1.apply(Task.scala:576)
//=> 	at monix.eval.Task$$anonfun$create$1.apply(Task.scala:571)
//=> 	at monix.eval.Task$AsyncStateRunnable.run(Task.scala:1429)
//=>    ...

// The Future NEVER COMPLETES, OOPS!
future.onComplete(r => println(r))
```

**WARNING:** In this case the consumer side never gets a completion
signal. The moral of the story is: even if Monix makes a best effort
to do the right thing, you should protect your freaking code against
unwanted exceptions, especially in `Task.create`!!!

### Overriding the Error Logging

The article on [Scheduler](../execution/scheduler.md) has recipes
for building your own `Scheduler` instances, with your own logic. But
here's a quick snippet for building such a `Scheduler` that could do
logging by means of a library, such as the standard
[SLF4J](http://www.slf4j.org/):

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

### Trigger a Timeout

In case a `Task` is too slow to execute, we can cancel it and trigger
a `TimeoutException` using `Task.timeout`:

```scala mdoc:silent:nest
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

val source =
  Task("Hello!").delayExecution(10.seconds)

// Triggers error if the source does not
// complete in 3 seconds after runAsync
val timedOut = source.timeout(3.seconds)

timedOut.runAsync(r => println(r))
//=> Failure(TimeoutException)
```

On timeout the source gets canceled (if it's a source that supports
cancelation). And instead of an error, we can timeout to a `fallback`
task. The following example is equivalent to the above one:

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

### Recovering from Error

`Task.onErrorHandleWith` is an operation that takes a function mapping
possible exceptions to a desired fallback outcome, so we could do
this:

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
    // Oh, we know about timeouts, recover it
    Task.now("Recovered!")
  case other =>
    // We have no idea what happened, raise error!
    Task.raiseError(other)
}

recovered.runToFuture.foreach(println)
//=> Recovered!
```

There's also `Task.onErrorRecoverWith` that takes a partial function
instead, so we can omit the "other" branch:

```scala mdoc:silent:nest
val recovered = source.onErrorRecoverWith {
  case _: TimeoutException =>
    // Oh, we know about timeouts, recover it
    Task.now("Recovered!")
}

recovered.runToFuture.foreach(println)
//=> Recovered!
```

`Task.onErrorHandleWith` and `Task.onErrorRecoverWith` are the
equivalent of `flatMap`, only for errors. In case we know or can
evaluate a fallback result eagerly, we could use the shortcut
operation `Task.onErrorHandle` like:

```scala mdoc:silent:nest
val recovered = source.onErrorHandle {
  case _: TimeoutException =>
    // Oh, we know about timeouts, recover it
    "Recovered!"
  case other =>
    throw other // Rethrowing
}
```

Or the partial function version with `onErrorRecover`:

```scala mdoc:silent:nest
val recovered = source.onErrorRecover {
  case _: TimeoutException =>
    // Oh, we know about timeouts, recover it
    "Recovered!"
}
```

### Restart On Error

The `Task` type, being just a specification, it can usually restart
whatever process is supposed to deliver the final result and we can
restart the source on error, for how many times are needed:

```scala mdoc:silent:nest
import scala.util.Random

val source = Task(Random.nextInt()).flatMap {
  case even if even % 2 == 0 =>
    Task.now(even)
  case other =>
    Task.raiseError(new IllegalStateException(other.toString))
}

// Will retry 4 times for a random even number,
// or fail if the maxRetries is reached!
val randomEven = source.onErrorRestart(maxRetries = 4)
```

We can also restart with a given predicate:

```scala mdoc:silent:nest
import scala.util.Random

val source = Task(Random.nextInt()).flatMap {
  case even if even % 2 == 0 =>
    Task.now(even)
  case other =>
    Task.raiseError(new IllegalStateException(other.toString))
}

// Will keep retrying for as long as the source fails
// with an IllegalStateException
val randomEven = source.onErrorRestartIf {
  case _: IllegalStateException => true
  case _ => false
}
```

Or we could implement our own retry with exponential backoff, because
it's cool doing so:

```scala mdoc:silent:nest
def retryBackoff[A](source: Task[A],
  maxRetries: Int, firstDelay: FiniteDuration): Task[A] = {

  source.onErrorHandleWith {
    case ex: Exception =>
      if (maxRetries > 0)
        // Recursive call, it's OK as Monix is stack-safe
        retryBackoff(source, maxRetries-1, firstDelay*2)
          .delayExecution(firstDelay)
      else
        Task.raiseError(ex)
  }
}
```

### Expose Errors

The `Task` monadic context is hiding errors that happen, much like
Scala's `Try` or `Future`. But sometimes we want to expose those
errors such that we can recover more efficiently:

```scala mdoc:silent:nest
import scala.util.{Try, Success, Failure}

val source = Task.raiseError[Int](new IllegalStateException)
val materialized: Task[Try[Int]] =
  source.materialize

// Now we can flatMap over both success and failure:
val recovered = materialized.flatMap {
  case Success(value) => Task.now(value)
  case Failure(_) => Task.now(0)
}

recovered.runToFuture.foreach(println)
//=> 0
```

There's also the reverse of materialize, which is `Task.dematerialize`:

```scala mdoc:silent:nest
import scala.util.Try

val source = Task.raiseError[Int](new IllegalStateException)

// Exposing errors
val materialized = source.materialize
// materialize: Task[Try[Int]] = ???

// Hiding errors again
val dematerialized = materialized.dematerialize
// dematerialized: Task[Int] = ???
```

We can also convert any `Task` into a `Task[Throwable]` that will
expose any errors that happen and will also terminate with an
`NoSuchElementException` in case the source completes with success:

```scala mdoc:silent:nest
val source = Task.raiseError[Int](new IllegalStateException)

val throwable = source.failed
// throwable: Task[Throwable] = ???

throwable.runToFuture.foreach(println)
//=> java.lang.IllegalStateException
```
