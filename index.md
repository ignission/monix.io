---
layout: page
title: Monix
---

Scala と [Scala.js](http://www.scala-js.org/) のための非同期プログラミング。

{% include github-stars.html %}

## 概要

Monixは、非同期でイベントベースのプログラムを構成するための、高性能なScala / Scala.jsライブラリです。

<a href="https://typelevel.org/"><img src="{{ site.baseurl }}public/images/typelevel.png" width="150" style="float:right;" align="right" /></a>

[Typelevel project](http://typelevel.org/projects/) であるMonixは、誇りを持って Scalaの純粋な型付き関数型プログラミングを実証しています。パフォーマンスには一切妥協していません。

ハイライト:
- `Observable`, `Iterant`, `Task`, `Coeval`のデータ型を公開し、必要なサポートを提供しています
- モジュラーになっており、必要なものだけを使うことができます 
- 真の非同期性のために設計され、JVMと [Scala.js](http://scala-js.org) の両方で動作します 
- 優れたテストカバレッジ、コード品質、APIドキュメンテーションをプロジェクトの主要な方針としています

このプロジェクトは、[ReactiveX](http://reactivex.io/) の適切な実装として始まりました。
より強い関数型プログラミングの影響を受け、ゼロから設計されています。
バックプレッシャーに対応し、Scalaの標準ライブラリときれいに連動するように作られています。
また、[Reactive Streams](https://www.reactive-streams.org/) のプロトコルとも互換性があります。
しばらくして、副作用を一時停止するための抽象化やリソース処理を含むように拡張されました。
Monixは [Cats Effect](https://typelevel.org/cats-effect/) の親であり実装の一つになっています。

### プレゼンテーション

{% include presentations.html %}

### ダウンロードと使い方

ライブラリはMaven Centralで公開されています。

- 3.x release (latest): `{{ site.promoted.version3x }}`
  ([ソースコードのダウンロード]({{ site.github.repo }}/archive/v{{ site.promoted.version3x }}.zip))
- 2.x release (older): `{{ site.promoted.version2x }}`
  ([ソースコードのダウンロード]({{ site.github.repo }}/archive/v{{ site.promoted.version2x }}.zip))

最新の3.xリリースでは [Typelevel Cats](https://typelevel.org/cats/) と統合しました。  
SBTからは以下のように使用してください（Scala.jsには`%%%`を使用します）。

```scala
libraryDependencies += "io.monix" %% "monix" % "{{ site.promoted.version3x }}"
```

Monixはモジュール式に設計されているため、à la carte(アラカルト)な利用が可能です。プロジェクトは複数のサブプロジェクトに分割されています。

[SBTでの使用法](./docs/current/intro/usage.md) と、[サブモジュールのグラフ](./docs/current/intro/usage.md#sub-modules--dependencies-graph) を参照してください。また、バイナリの後方互換性を保証する [バージョニングスキーム](./docs/current/intro/versioning-scheme.md) もお見逃しなく。

### ドキュメント

ドキュメントとチュートリアル:

- [Current](/docs/current/) ([3.x series](/docs/3x/))
- [2.x series](/docs/2x/)

API ScalaDoc:

- [Current]({{ site.apiCurrent }}) ([3.x]({{ site.api3x }}))
- [2.x]({{ site.api2x }})
- [1.x]({{ site.api1x }})

依存関係の関連リンク:

- [Cats](https://typelevel.org/cats/)
- [Cats Effect](https://typelevel.org/cats-effect/)

## 最新ニュース

<ol class="news-summary">
  {% for post in site.posts limit: 5 %}
  <li>
    <time itemprop="dateCreated"
      datetime="{{ post.date | date: "%Y-%m-%d" }}">
      {{ post.date | date_to_string }} »
    </time>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ol>

## 利用者

本番環境で Monix を使用している企業(すべて網羅しているわけではありません)のリストです。
この中にありませんか？ [プルリクエストを送りましょう！](https://github.com/monix/monix/blob/series/4.x/README.md)

- [Abacus](https://abacusfi.com)
- [Agoda](https://www.agoda.com)
- [commercetools](https://commercetools.com)
- [Coya](https://www.coya.com/)
- [E.ON Connecting Energies](https://www.eon.com/)
- [eBay Inc.](https://www.ebay.com)
- [Eloquentix](http://eloquentix.com/)
- [Hypefactors](https://www.hypefactors.com)
- [Iterators](https://www.iteratorshq.com)
- [Sony Electronics](https://www.sony.com)
- [Tinkoff](https://tinkoff.ru)
- [Zalando](https://www.zalando.com)
- [Zendesk](https://www.zendesk.com)

## ライセンス

Copyright (c) 2014-{{ site.time | date: '%Y' }} by The Monix Project Developers.
Some rights reserved.

[LICENSEを読む]({% link LICENSE.md %})

## 翻訳者から
この日本語訳ドキュメントは、現在のところ非公式です。  
イシューやプルリクエストは [ignission/monix.io](https://github.com/ignission/monix.io) までお送りください。