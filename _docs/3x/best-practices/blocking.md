---
layout: docs3x
title: "Best Practice: Should Not Block Threads"
description: "Blocking threads is incredibly error prone. And if you must block, do so with Scala's BlockContext and with explicit timeouts."
---

選択肢があるのであれば、絶対にスレッドをブロックしてはいけません。例えば、次のようなコードです:

```scala
def fetchSomething: Future[String] = ???

// いろいろ処理をしたあとで ...
val result = Await.result(fetchSomething, Duration.Inf)
result.toUpperCase
```

プログラムの最後まで、ずっとそのFutureの文脈を保つことが望ましいです。

```scala
def fetchSomething: Future[String] = ???

fetchSomething.map(_.toUpperCase)
```

**プロによるアドバイス:** [Scala-Async](https://github.com/scala/async) プロジェクトをチェックアウトすると、Scalaの `Future` をより理解しやすくなります。

**理由:** ブロッキング・スレッドを使用するとエラーが発生しやすくなります。
これは基盤となるスレッドプールの設定を知り、制御する必要があるからです。
例えばScala の `ExecutionContext.Implicits.global` には、生成されるスレッド数の上限があり、*デッドロック* 状態になってしまう可能性があります。
なぜならすべてのスレッドがブロックされると、必要なコールバックを完了するためにスレッドプールで利用可能なスレッドがなくなるからです。

## もしブロッキングするなら、明示的にタイムアウトを指定してください

ブロックしなければならない場合は、失敗したときのタイムアウトを明示的に指定してください。
何らかの結果でブロックされるAPIや、明示的なタイムアウトがないAPIは絶対に使用しないでください。

例えば、Scala の `Await.result` は非常によくできていて、次のような利点があります。
これは良い例です:

```scala
Await.result(future, 3.seconds)
```

しかし、次の例は [JavaのFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html) を使用しています。  
このような使い方をしないでください:

```scala
val future: java.util.concurrent.Future[T] = ???

// 悪い例です。絶対に避けてください !!!
future.get
```

その代わり、常にタイムアウトを指定します。
スレッドプールが限られていてもうスレッドが残っていない場合、少なくとも指定された時間が経過すると、いくつかのスレッドのブロックが解除されます。

```scala
val future: java.util.concurrent.Future[T] = ???

// 良いですね
future.get(TimeUnit.SECONDS, 3)
```

## もしブロッキングするなら、ScalaのBlockContextを使用してください

次の例には、SQLクエリを含むすべてのブロッキングI/Oが含まれます。

```scala
// 悪い例です！
Future {
  DB.withConnection { implicit connection =>
    val query = SQL("select * from bar")
    query()
  }
}
```

ブロッキングコールは、どのスレッドプールが影響を受けるかを正確に認識しなければならないため、エラーが発生しやすくなります。
バックエンドアプリのデフォルト設定では、非決定的なデッドロックが発生する可能性があります。これは本番でも起こりうるバグです。

ここでは、この問題を説明するための簡単な例を紹介します:

```scala
implicit val ec = ExecutionContext
  .fromExecutor(Executors.newFixedThreadPool(1))

def addOne(x: Int) = Future(x + 1)

def multiply(x: Int, y: Int) = Future {
  val a = addOne(x)
  val b = addOne(y)
  val result = for (r1 <- a; r2 <- b) yield r1 * r2

  // スレッドプールのサイズが限られているため、デッドロックが発生する可能性があります！
  Await.result(result, Duration.Inf)
}
```

このサンプルは効果を決定論的にするために単純化されていますが、すべての上限を設定したスレッドプールは、遅かれ早かれこの影響を受けます。

ブロッキングコールは、`BlockContext` にブロッキング操作のシグナルを送る `blocking` コールでマークする必要があります。
これはScalaの非常に優れたメカニズムで、ブロック操作が発生したことを`ExecutionContext`に知らせ、`ExecutionContext`がそれに対して何をすべきかを決めることができます。
例えば，スレッドプールにさらにスレッドを追加するとか（これはScalaのForkJoinスレッドプールがそうです）です。

**警告:** Scalaの `ExecutionContext.Implicits.global` は、スレッド数に絶対的な制限を設けたクールな `ForkJoinPool` 実装に支えられています。
これが意味するところは、よくできたコードにもかかわらず、その制限にぶつかってしまいデッドロックに陥る可能性があるということです。
これが、スレッドをブロックすることがエラーになりやすい理由であり、自分がブロックしてしまうスレッドプールを知り、制御することができないからです。

## ブロッキングするなら、ブロッキングI/O用に別のスレッドプールを使用してください

ブロッキングI/Oを多用している場合(JDBCの呼び出しが多い場合など)、 2つ目のスレッドプール/実行コンテキストを作成して、すべてのブロッキングコールをそのスレッドプールで実行したほうがいいでしょう。
アプリケーションのスレッドプールにはCPU負荷のかかる処理を任せます。

そこで、I/O関連用の別のスレッドプールを以下のように初期化します:

```scala
import java.util.concurrent.Executors

// ...
private val io = Executors.newCachedThreadPool(
  new ThreadFactory {
    private val counter = new AtomicLong(0L)

    def newThread(r: Runnable) = {
      val th = new Thread(r)
      th.setName("io-thread-" +
      counter.getAndIncrement.toString)
      th.setDaemon(true)
      th
    }
  })
```

ここでは、境界のない「キャッシュされたスレッドプール」を使用したいと思います。制限はありません。
ブロッキングI/Oを行う場合、アイデアとしてはブロックできるだけのスレッドを持っていなければなりません。
しかし、もしそれが多すぎる場合は用途に応じて後で微調整することができます。

もちろん、Monix の `Scheduler.io` を使用することもできますが、これもまた「キャッシュされたスレッドプール」に支えられています:

```scala
import monix.execution.Scheduler

private val io = 
  Scheduler.io(name="engine-io")
```

次のようにヘルパーメソッドを用意するのもいいでしょう:

```scala
def executeBlockingIO[T](cb: => T): Future[T] = {
  val p = Promise[T]()

  io.execute(new Runnable {
    def run() = try {
      p.success(blocking(cb))
    }
    catch {
      case NonFatal(ex) =>
        logger.error(s"Uncaught I/O exception", ex)
        p.failure(ex)
    }
  })

  p.future
}
```


