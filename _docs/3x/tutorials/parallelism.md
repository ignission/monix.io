---
layout: docs3x
title: 並列処理 
description: |
  並列化のためのレシピ
---

Monixでは、ユースケースに応じて複数の並列化の方法が用意されています。

このドキュメントのサンプルはコピー・ペースト可能ですが、importを整理します:

```scala mdoc:silent:nest
// 評価時にはスケジューラーが必要
import monix.execution.Scheduler.Implicits.global
// Taskの使用に必要
import monix.eval._
// Observableの使用に必要
import monix.reactive._
```

```scala mdoc:invisible:nest
import monix.execution.schedulers.TestScheduler
implicit val global = TestScheduler()
```

## Taskでの並列処理

[タスク](../eval/task.md) の助けを借りて、結果の決定論的(順序付けられた)シグナリングを行うバッチ単位の並列実行が可能です。

### ネイティブに使う

次の例では，[Task.parSequence]({{ page.path | api_base_url }}monix/eval/Task$.htmll#parSequenceN[A](parallelism:Int)(in:Iterable[monix.eval.Task[A]]):monix.eval.Task[List[A]]) を使用しています。
これは、結果の順序を保持したまま並列処理を行うものです。

```scala mdoc:silent:nest
val items = 0 until 1000

// 実行に必要なすべてのタスクのリスト
val tasks = items.map(i => Task(i * 2))
// 並列に処理する
val aggregate = Task.parSequence(tasks).map(_.toList)

// 評価:
aggregate.foreach(println)
//=> List(0, 2, 4, 6, 8, 10, 12, 14, 16,...
```

結果の順序が重要でない場合は、`parSequence`の代わりに[Task.parSequenceUnordered]({{ page.path | api_base_url }}monix/eval/Task$.htmll#parSequenceUnordered[A](in:Iterable[monix.eval.Task[A]]):monix.eval.Task[List[A]]) を使用できます。
ノンブロッキングで実行できるので、より良い結果が得られるかもしれません。

### 並列度制限の導入

上の例のように、`Task.parSequence`ビルダーは与えられたすべてのタスクを潜在的に並列実行します。
しかし、問題はこれが非効率につながる可能性があるということです。
例えばHTTPリクエストを行っている場合、10000のHTTPリクエストを並行して実行することは、相手側のサーバーをダウンさせる可能性があるため必ずしも賢明ではありません。

この問題を解決するには、ワークロードを並列タスクのバッチに分割し、それを順番に実行します:

```scala mdoc:silent:nest
val items = 0 until 1000
// 実行に必要なすべてのタスクのリスト
val tasks = items.map(i => Task(i * 2))
// 10個のタスクのバッチを構築して並列に実行する:
val batches = tasks.sliding(10,10).map(b => Task.parSequence(b)).toIterable
// バッチを配列にし、最終的な結果をフラットにする
val aggregate = Task.sequence(batches).map(_.flatten.toList)

// 評価する:
aggregate.foreach(println)
//=> List(0, 2, 4, 6, 8, 10, 12, 14, 16,...
```

この戦略はScalaの`Future`では実現が難しいことに注意してください。
なぜなら、`Future.sequence`があるにもかかわらず、その動作は厳密なものであるためシーケンスとパラレルをうまく使い分けることができません。
この動作は`Future.sequence`にlazyまたはstrictシーケンスを渡すことで制御されますが、これは明らかにエラーを起こしやすいです。

### Batched Observables

これを`Observable.flatMap`と組み合わせて、リクエストをバッチ処理することもできます。

```scala mdoc:silent:nest
import monix.eval._
import monix.reactive._

// `bufferIntrospective`は、ダウンストリームがビジー状態である限り、
// 特定の`bufferSize`までバッファリングされたすべてのイベントのシーケンスを
// 一度にストリーミングします。
val source = Observable.range(0,1000).bufferIntrospective(256)

// バッチ処理には`Task`を使用しています
val batched = source.flatMap { items =>
  // 実行に必要なすべてのタスクのリスト
  val tasks = items.map(i => Task(i * 2))
  // 10個のタスクを並列に実行するバッチの構築:
  val batches = tasks.sliding(10,10).map(b => Task.parSequence(b)).toIterable
  // バッチを配列にし、最終的な結果をフラットにする
  val aggregate = Task.sequence(batches).map(_.flatten.iterator)
  // flatMapに必要なObservableへの変換
  Observable.fromIterator(aggregate)
}

// 評価:
batched.toListL.foreach(println)
//=> List(0, 2, 4, 6, 8, 10, 12, 14, 16,...
```

[bufferIntrospective]({{ page.path | api_base_url }}monix/reactive/Observable.html#bufferIntrospective(maxSize:Int):Self[List[A]]) の使用に注意してください。
これは、ダウンストリームがビジー状態の間受信したイベントをバッファリングし、その後はバッファを1つのバンドルとして放出します。
[bufferTumbling]({{ page.path | api_base_url }}monix/reactive/Observable.html#bufferTumbling(count:Int):Self[Seq[A]]) のようなメソッドは、より決定性の高い代替手段となり得ます。

## Observable.mapParallelUnordered

並列化を実現するもう一つの方法は[Observable.mapParallelUnordered]({{ page.path | api_base_url }}monix/reactive/Observable.html#mapParallelUnordered[B](parallelism:Int)(f:A=>monix.eval.Task[B]):Self[B]) メソッドを使用することです:

```scala mdoc:silent:nest
val source = Observable.range(0,1000)
// 並列度を指定する必要があります
val processed = source.mapParallelUnordered(parallelism = 10) { i =>
  Task(i * 2)
}

// 評価する:
processed.toListL.foreach(println)
//=> List(2, 10, 0, 4, 8, 6, 12...
```

上記の例のように`Task.parSequence`を使用する場合と比較して、このメソッドではソースから通知された結果の**順序は維持されません**。

これは、少なくとも1つのワーカーがアクティブである限り、ソースがバックプレッシャーを受けることがないため、より効率的な実行につながります。
一方、上記の例のようなバッチ実行戦略では、1つの非同期タスクが完了するのに時間がかかりすぎて非効率になることがあります。

## Observable.mergeMap

もし、`Observable.mapParallelUnordered`が`Task`で動作するならば、
[Observable.mergeMap](https://monix.io/api/2.2/monix/reactive/Observable.html#mergeMap[B](f:A=%3Emonix.reactive.Observable[B])(implicitos:monix.reactive.OverflowStrategy[B]):Self[B]) は`Observable`のインスタンスをマージして動作します。

```scala mdoc:silent:nest
val source = Observable.range(0,1000)
// 並列度を指定する必要があります
val processed = source.mergeMap { i =>
  Observable.eval(i * 2).executeAsync
}

// 評価する:
processed.toListL.foreach(println)
//=> List(0, 4, 6, 2, 8, 10, 12, 14...
```

なお、`mergeMap`はソースから出力される観測可能なストリームを並行してサブスクライブされるため、
結果が非決定的であることを除いて`concatMap`(Monixでは`flatMap`のエイリアス)と似ています。

上の例のように、この `mergeMap` 呼び出しはオプションの`parallelism`パラメータを持っていないことに注意してください。
つまりソースがおしゃべりであれば、並行してサブスクライブされたオブザーバが *大量* にできてしまうということです。

問題は、`mergeMap`メソッドが実際の並列処理のためのものではなく、アクティブで同時進行しているストリームを結合するためのものです。

## Consumer.loadBalancer

`MapParallelUnordered`のような操作をconsumer側に適用することができます。
consumer側では、チュートリアルの[Consumer](./reactive/consumer.md) で例示されているように負荷分散されたコンシューマーを使って、すべてのワーカーの結果を最終的に集約することができます:

```scala mdoc:silent:nest
import monix.eval._
import monix.reactive._

// ストリームの要素を折り返すコンシューマ
// 結果として和を生成する
val sumConsumer = Consumer.foldLeft[Long,Long](0L)(_ + _)

// 合計を並列処理する場合はもちろん役に立たないが、
// I/Oに縛られたロジックには非常に有効である。
val loadBalancer = {
  Consumer
    .loadBalance(parallelism=10, sumConsumer)
    .map(_.sum)
}

val observable: Observable[Long] = Observable.range(0, 100000)
// このコンシューマーは、observableをTaskの合計処理に変えてくれます w00t!
val task: Task[Long] = observable.consumeWith(loadBalancer)

// ストリーム全体を消費して結果を得る
task.runToFuture.foreach(println)
//=> 4999950000
```

詳しくは[Consumer](../reactive/consumer.md) をご覧ください。
