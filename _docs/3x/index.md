---
layout: docs3x
title: Monix ドキュメント
---

## API ドキュメント

[ScalaDoc for Monix 3.x]({{ page.path | api_base_url }})

関連:

- [Cats ScalaDoc](https://typelevel.org/cats/api/){:target=>"_blank"}
- [Cats-Effect ScalaDoc](https://typelevel.org/cats-effect/api/){:target=>"_blank"}

## 始めよう 

- [SBTで使う](./intro/usage.md)
- [バージョニング](./intro/versioning-scheme.md)
- [Hello World!](./intro/hello-world.md)
- [Typelevel Cats との統合](./intro/cats.md)

[クイックスタートのテンプレート](https://github.com/monix/monix-jvm-app-template.g8):

```
sbt new monix/monix-jvm-app-template.g8
```

## サブプロジェクト

### monix-execution

`monix-execution`サブプロジェクトでは、低レベルで副作用の少ない、並行性を扱うユーティリティを提供し、JVMのプリミティブを `scala.concurrent` パッケージ上で公開しています。

<img src="{{ site.baseurl }}public/images/logos/java.png" alt="Java's Duke" title="Java's Duke"
     class="doc-icon" />

- [APIドキュメント]({{ page.path | api_base_url }}monix/execution/index.html)
- [Scheduler](./execution/scheduler.md)
- [Cancelable](./execution/cancelable.md)
- [Callback](./execution/callback.md)
- [Future Utils](./execution/future-utils.md)
- [Atomic](./execution/atomic.md)
- [Local](./execution/local.md)

### monix-catnap

`monix-catnap` サブプロジェクトは、並行性を管理するための汎用的で純粋に機能的なユーティリティで、[Cats-Effect](https://typelevel.org/cats-effect/) type classes の上に構築されています。

<img src="{{ site.baseurl }}public/images/logos/cats.png" alt="Cats" title="Cats"
     class="doc-icon" />

- [APIドキュメント]({{ page.path | api_base_url }}monix/catnap/index.html)
- [Circuit Breaker](./catnap/circuit-breaker.md)
- [MVar](./catnap/mvar.md)

### monix-eval

`monix-eval` サブプロジェクトでは、純粋に関数的な効果を原理的に扱うために、`Task`と`Coeval` データ型を公開しています。

- [APIドキュメント]({{ page.path | api_base_url }}monix/eval/index.html)
- [Task](./eval/task.md)
- [Coeval](./eval/coeval.md)
- [Better Stack Traces](./eval/stacktraces.md)

### monix-reactive

`monix-reactive`サブプロジェクトでは、`Observable`データ型とそれに付随するユーティリティ、高性能なストリーミングの抽象化を公開しています。これは、ScalaのReactiveXのイディオム的な実装です。

<img src="{{ site.baseurl }}public/images/logos/reactivex.png" alt="ReactiveX" title="ReactiveX"
     class="doc-icon" />

- [APIドキュメント]({{ page.path | api_base_url }}monix/reactive/index.html)
- [Observable](./reactive/observable.md)
- [Comparisons with Other Solutions](./reactive/observable-comparisons.md)
- [Observers and Subscribers](./reactive/observers.md)
- [Consumer](./reactive/consumer.md)
- [Javascript Event Listeners](./reactive/javascript.md)

### monix-tail

<img src="{{ site.baseurl }}public/images/logos/many-cats.png" alt="Cats friendly" title="Cats friendly"
     class="doc-icon2x" />

`monix-tail` サブプロジェクトでは、汎用的で純粋に機能的、原理的でプルベースのストリーミングデータタイプである`Iterant`を公開しています。

- [APIドキュメント]({{ page.path | api_base_url }}monix/tail/index.html)

## チュートリアル
  
- [並列処理](./tutorials/parallelism.md)
{% for post in site.posts -%}{% if post.tutorial == "3x" %}
- [{{ post.title }}]({{ post.url }})
{% endif %}{% endfor %}
  
## ベストプラクティス
  
- [スレッドをブロックしてはいけない](./best-practices/blocking.md)

## サンプル

- [Client/Server Communications](https://github.com/monixio/monix-sample/):
  [Play Framework](https://www.playframework.com/) と
  [Scala.js](http://www.scala-js.org/) で作るアプリケーション

## プレゼンテーション

{% include presentations.html %}

更新中です！ よろしくお願いします！
