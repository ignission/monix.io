---
layout: docs3x
title: MVar
type_api: monix.catnap.MVar
type_source: monix-catnap/shared/src/main/scala/monix/catnap/MVar.scala
description: |
  空または値を含めることができる可変の変数で、空のときは非同期に読み込みをブロックし、満杯のときは書き込みをブロックします。
---

`MVar`は、空または値を含めることができる可変の変数で、空のときは非同期に読み込みをブロックし、満杯のときは書き込みをブロックします。

## 概要

ユースケース:

1. 同期され、スレッドセーフな可変型変数として
2. `take`と`put`が「受信」と「送信」の役割を持つチャンネルとして 
3. `take`と`put`が「取得」と「解放」の役割を持つバイナリーセマフォとして

これらの基本的でアトミックな操作を持っています:

- `put`は空の場合に変数へ値を入れるか、変数が再び空になるまで(非同期に)ブロックする。
- `tryPut`は空の場合に変数へ値を入れようとし、成功した場合は`true`を返す。
- `take`は満杯になったときにvarを空にして含まれている値を返し、そうでなければ取り出す値があるまで(非同期に)ブロックする。
- `tryTake`は満杯になったら空にし、`None`を返す。 
- `read`は現在の値があると仮定して触れずに読み取るか、そうでなければ`put`で値が利用可能になるまで待つ。  
- `tryRead`は満杯であれば`Some(a)`を変更せずに返し、そうでなければ `None` を返す。
- `isEmpty`は現在空であれば`true`を返す。

<p class="extra" markdown='1'>
この文脈では、"<i>非同期ブロッキング</i>"とはスレッドをブロックしないことを意味します。
その代わりに実装では、操作が終了したことをクライアントに通知するためにコールバックを使用しています(通知方法は[Task](./task.md)によって公開されています)。
そのため、Javascriptの上でも動作します。
</p>

### インスピレーション

このデータタイプは、論文[Concurrent Haskell](http://research.microsoft.com/~simonpj/papers/concurrent-haskell.ps.gz)で紹介されています。
Simon Peyton Jones、Andrew Gordon、Sigbjorn Finneによって実装の詳細は変更されています(特に完全な`MVar`への`put`は以前はエラーになっていましたが、現在は単にブロックするだけです)。

同期プリミティブの構築や単純なスレッド間通信の実行に適しています。
これは`BlockingQueue(capacity = 1)`と同等のものです。
ただし、実際にはスレッドのブロッキングは行われず、`Task`を利用することになります。

## Cats-Effect

`MVar`は汎用的で、[Cats-Effect](https://typelevel.org/cats-effect/)の型クラスを介してエフェクトタイプを抽象化するように作られています。
つまり、Monixの`Task`と同じように`MVar`を[cats.effect.IO](https://typelevel.org/cats-effect/datatypes/io.html)や、`Async`、`Concurrent`を実装したデータタイプと同様に使用できるということです。

なお、`MVar`については[cats.effect.concurrent.MVar](https://typelevel.org/cats-effect/concurrency/mvar.md)ですでに説明されており、Monixは実際にそのインターフェイスを実装しています。

<a href="https://typelevel.org/cats-effect/concurrency/mvar.md" target="_blank" 
  title="cats.effect.concurrent.MVar" alt="cats.effect.concurrent.MVar">
  <img src="{{ site.url }}/public/images/concurrency-mvar.png" width="400" />
</a>

次の理由から、`MVar`はMonixにも残ります:

1. `Future`に対応した代替品として、[monix.execution.AsyncVar]({{ page.path | api_base_url }}monix/execution/AsyncVar.html)と実装を共有している。
2. Monixの[Atomic](../execution/atomic.md)の実装を使うことができる。
3. 現時点でMonixの`MVar`には、次のバージョンの`Cats-Effect`が上流にマージされるのを待つ必要がある。

## ユースケース: 同期した可変型変数

```scala mdoc:invisible:nest
import monix.eval._
import monix.execution._
import monix.execution.Scheduler
import monix.execution.schedulers.TestScheduler

implicit val scheduler: Scheduler = TestScheduler()
```

```scala mdoc:silent:nest
import monix.execution.CancelableFuture
import monix.catnap.MVar
import monix.eval.Task

def sum(state: MVar[Task, Int], list: List[Int]): Task[Int] =
  list match {
    case Nil => state.take
    case x :: xs =>
      state.take.flatMap { current =>
        state.put(current + x).flatMap(_ => sum(state, xs))
      }
  }

val task = 
  for {
    state <- MVar[Task].of(0)
    r <- sum(state, (0 until 100).toList)
  } yield r

// 評価する
task.runToFuture.foreach(println)
//=> 4950
```

このサンプルは`MVar`がどのように変数として使用できるかを示す以外、あまり役に立ちません。
`take`と`put`の操作はアトミックです。
利用可能な値がない場合、`take`の呼び出しは(非同期的に)ブロックします。
一方、`put`の呼び出しは`MVar`にすでに値が入っていて，その値が返されるのを待っている場合にブロックします。

明らかに、`take`の呼び出しの後`put`の呼び出しの前には、同じ変数を更新できる同時実行のロジックがあります。
2つの操作は単独ではアトミックですが、組み合わせる操作はアトミックではありません(つまりアトミックな操作は合成されません)。
このサンプルを*安全*にしたいのであれば、追加の同期が必要になります。

## ユースケース: 非同期ロック(バイナリセマフォ、ミューテックス)

`take`操作は「取得」、`put`操作は「解除」の役割を果たします。
それではやってみましょう。

```scala mdoc:silent:nest
final class MLock(mvar: MVar[Task, Unit]) {
  def acquire: Task[Unit] =
    mvar.take

  def release: Task[Unit] =
    mvar.put(())

  def greenLight[A](fa: Task[A]): Task[A] =
    for {
      _ <- acquire
      a <- fa.doOnCancel(release)
      _ <- release
    } yield a
}

object MLock {
  /** ビルダー */
  def apply(): Task[MLock] =
    MVar[Task].of(()).map(v => new MLock(v))
}
```

そして今度は、先ほどの例に同期を適用してみましょう:

```scala mdoc:silent:nest
val task = 
  for {
    lock <- MLock()
    state <- MVar[Task].of(0)
    task = sum(state, (0 until 100).toList)
    r <- lock.greenLight(task)
  } yield r

// 評価する
task.runToFuture.foreach(println)
//=> 4950
```

## ユースケース: プロデューサー/コンシューマーチャンネル

明白な使用例は、単純なプロデューサーとコンシューマーのチャンネルをモデル化することです。

イベントをプッシュする必要のあるプロデューサーがいるとします。
しかし、バックプレッシャーも必要なので、新しいイベントを生成する前にコンシューマが最後のイベントを消費するのを待つ必要があります。

```scala mdoc:silent:nest
// 完了を検知する必要があるためのシグナリングオプション
type Channel[A] = MVar[Task, Option[A]]

def producer(ch: Channel[Int], list: List[Int]): Task[Unit] =
  list match {
    case Nil =>
      ch.put(None) // できた！
    case head :: tail =>
      // 次、お願いします
      ch.put(Some(head)).flatMap(_ => producer(ch, tail))
  }

def consumer(ch: Channel[Int], sum: Long): Task[Long] =
  ch.take.flatMap {
    case Some(x) =>
      // 次、お願いします
      consumer(ch, sum + x)
    case None =>
      Task.now(sum) // できた！
  }

val count = 100000

val sumTask =
  for {
    channel <- MVar[Task].empty[Option[Int]]()
    producerTask = producer(channel, (0 until count).toList).executeAsync
    consumerTask = consumer(channel, 0L).executeAsync
    // 実際には必要ないが、念のために並列処理を確実に行う。
    sum <- Task.parMap2(producerTask, consumerTask)((_,sum) => sum)
  } yield sum

// 評価する
sumTask.runToFuture.foreach(println)
//=> 4999950000
```

これを実行すると期待通りに動作します。
`producer`が値を`MVar`にプッシュし、`consumer `がそれらの値をすべて消費します。
