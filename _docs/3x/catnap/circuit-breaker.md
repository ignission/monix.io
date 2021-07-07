---
layout: docs3x
title: サーキットブレーカー
type_api: monix.catnap.CircuitBreaker
type_source: monix-catnap/shared/src/main/scala/monix/catnap/CircuitBreaker.scala
description: |
  分散システムの安定性を確保し、連鎖的な障害を防ぐためのデータタイプ。
---

`CircuitBreaker`は分散システムの安定性を高め、連鎖的な障害を防ぐために使用されます。

## 目的

例として、外部サービスと通信するウェブアプリケーションがあるとします。
外部サービスはキャパシティをオーバーして、データベースが負荷に耐えられなくなったケースを考えます。
そうすると外部サービスからエラーが返ってくるのに長い時間がかかったりしますよね。
ユーザーはフォームの送信に非常に時間がかかるため、ウェブアプリケーションがハングしていると感じるでしょう。
ユーザーは不安に思ったので、既に実行されているリクエストにさらにリクエストを追加します。
その結果、リソースの枯渇によるウェブアプリケーションの障害が発生します。
これは、通信先の問題と関係なくすべてのユーザーに影響してしまいます。

外部サービスの呼び出しにサーキットブレーカーを導入すると、リクエストが早く失敗し始め何かが間違っていてリクエストを更新する必要がないことをユーザーに知らせます。
これはまた、障害の発生は外部サービスに依存した機能を使用しているユーザーだけに限定され、他のユーザーはもはや影響を受けません。
サーキットブレーカーは外部サービスに依存している機能の一部を利用できないようにしたり、遮断中にキャッシュされたコンテンツを表示したりすることができます。

## 仕組み

サーキットブレーカーは、以下の3つのいずれかの状態になることができる並行ステートマシンをモデル化しています:

1. `Closed`: 通常時またはサーキットブレーカーの開始時
  - 例外は`failures`カウンタを増加させます。
  - 成功すると`failures`カウンタはゼロにリセットされます。
  - `failures`カウンタが`maxFailures`に到達すると、サーキットブレーカーは`Open`状態になります。
2. `Open`: サーキットブレーカーがすべてのタスクを拒否する
  - すべてのタスクは`ExecutionRejectedException`ですぐに失敗します。
  - 設定された`resetTimeout`の後、サーキットブレーカーは`HalfOpen`状態になり、接続テストのための1つのタスクが実行されます。  
3. `HalfOpen`: サーキットブレーカーは、接続をテストするためにリセットを試みて、タスクを通過させていきます。
  - `Open`の期限が切れたときの最初のタスクは、高速で失敗することなく実行されます。 サーキットブレーカーが`HalfOpen`になる前の状態です。  
  - `HalfOpen`状態で試行されたすべてのタスクは、`Open`状態と同様に例外を除いて高速に失敗します。
  - そのタスクが成功すると、ブレーカーはリセットされ、`resetTimeout`と`failures`カウントも初期値にリセットされます。
  - タスクの試行が失敗するとブレーカーは再び`Open`状態になります(設定された`maxResetTimeout`を上限に、`resetTimeout`に指数関数的なバックオフ係数が乗じられます)。  

<img src="{{ site.baseurl }}public/images/circuit-breaker-states.png" align="center" style="max-width: 100%" />
(画像の著作権はAkkaのドキュメントにあります)

## 使い方

```scala mdoc:silent:nest
import monix.catnap.CircuitBreaker
import monix.eval._
import scala.concurrent.duration._

val circuitBreaker: Task[CircuitBreaker[Task]] = 
  CircuitBreaker[Task].of(
    maxFailures = 5,
    resetTimeout = 10.seconds
  )
```

ビルダーの戻り値の参照は`Task`のコンテキストで与えられることに注意してください。
というのも、`CircuitBreaker`は状態を共有しているので、場合によっては参照透過性に反することになるからです。

これを回避するには、`unsafe`ビルダーを使用します。そうでなければ安全な方法を選択してください:

```scala mdoc:silent:nest
CircuitBreaker[Task].unsafe(
  maxFailures = 5,
  resetTimeout = 10.seconds
)
```

また、処理中のタスクを保護するために`protect`を使用することができます:

```scala mdoc:silent:nest
val problematic = Task {
  val nr = util.Random.nextInt()
  if (nr % 2 == 0) nr else
    throw new RuntimeException("dummy")
}

for {
  ci <- circuitBreaker
  r  <- ci.protect(problematic)
} yield r
```

サーキットブレーカーを閉じて通常のオペレーションを再開しようとするとき、繰り返す失敗に対して指数関数的なバックオフを適用することもできます:

```scala mdoc:silent:nest
val circuitBreaker = CircuitBreaker[Task].of(
  maxFailures = 5,
  resetTimeout = 10.seconds,
  exponentialBackoffFactor = 2,
  maxResetTimeout = 10.minutes
)
```

このサンプルでは、10秒後、20秒、40秒......と`maxResetTimeout`である最大10分まで間隔を増加し続けます。

### イベントハンドラー

ロギングやメトリクス関連のように、サーキットブレーカーの状態が変化したときにイベントを発生させたい場合:

```scala mdoc:silent:nest
CircuitBreaker[Task].of(
  maxFailures = 5,
  resetTimeout = 10.seconds,
  
  onRejected = Task { 
    println("TaskはOpenまたはHalOpenのときに拒否される")
  },
  onClosed = Task {
    println("Closeに切り替わり、タスクの受付を再開している")
  },
  onHalfOpen = Task {
    println("HalfOpenに切り替わり、テストのために1つのタスクだけ受け付ける")
  },
  onOpen = Task {
    println("Openに切り替わると、その後10秒間すべてのタスクが拒否される、")
  }
)
```

### クローズ後の再試行

リトライ戦略を実装する必要がある場合、素朴な方法として遅延を設けてリトライすることができます:

```scala mdoc:invisible:nest
val circuitBreaker = 
  CircuitBreaker[Task].unsafe(
    maxFailures = 5,
    resetTimeout = 10.seconds
  )
```

```scala mdoc:silent:nest
val task = circuitBreaker.protect(problematic)

task.onErrorRestartLoop(100.millis) { (e, delay, retry) =>
  // 指数的バックオフ、ただし上限あり
  if (delay < 4.seconds)
    retry(delay * 2).delayExecution(delay)
  else
    Task.raiseError(e)
}
```

しかし、一方では`CircuitBreaker`が再び閉じる瞬間を待つことができます:

```scala mdoc:silent:nest
task.onErrorRestartLoop(0) { (e, times, retry) =>
  // 最大10回の再試行
  if (times < 10)
    circuitBreaker.awaitClose.flatMap(_ => retry(times + 1))
  else
    Task.raiseError(e)
}
```

## Credits

<div class='extra' markdown='1'>
このデータタイプは、[Akkaのサーキットブレーカー](http://doc.akka.io/docs/akka/current/common/circuitbreaker.html)からヒントを得ています。
実装やAPIは同じではありませんが 目的や使用しているステートマシンは似ています。

このドキュメントには、Akkaからのコピーペーストされたフラグメントも含まれています。
クレジットが必要な場合はクレジットを表示してください 😉
</div>
