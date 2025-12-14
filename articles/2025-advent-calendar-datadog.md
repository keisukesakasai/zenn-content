---
title: "OpenTelemetry を使って収集したトレースと Datadog DBM の情報を関連付けてみる"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Datadog,Observability,OpenTelemetry,Database,Xmas]
published: true
---

こんにちは！
Datadog Advent Calendar 2025 の 12/14 担当の逆井（さかさい）です。
https://qiita.com/advent-calendar/2025/datadog

ちょっと前に、OpenTelemetry のトレースと、Database Monitoring を関連付けするための[ドキュメント](https://docs.datadoghq.com/opentelemetry/correlate/dbm_and_traces/?tab=datadogagentddotcollector)が生えました。OTel Lover にとっては待望？の機能です。これがどういうものなのか、なんで嬉しいか、OTel と ddtrace の差分的なところを書いていきます。

想定読者としては、OpenTelemetry や分散トレース、Datadog の Database Monitoring や sqlcommenter を知っていることを想定していますが、テレメトリーシグナルの関連付けに関心を持っている方であれば読みやすいように仕上げていきます。

## まず
[Correlate OpenTelemetry Traces and DBM](https://docs.datadoghq.com/opentelemetry/correlate/dbm_and_traces/?tab=datadogagentddotcollector) を見ると、セットアップはとても容易です。Database へのスパンにいくつかのタグを埋め込むように書いてあります。幾つかのタグは `db.system`, `db.statement`, `span.type` が必須のタグです。トレースと DBM の相関関係はクエリステートメントを基にしているためこれらのタグが必要になります。

OpenTelemetry のデータベーススパンのための計装ライブラリを使っている場合、例えば [otelsql](https://github.com/XSAM/otelsql) を使うと自然とクエリスパンへ上記のタグが埋め込まれます。`otelsql` の README に書いてある使い方のように、以下では [DB Semantic Cenventions](https://opentelemetry.io/docs/specs/semconv/registry/attributes/db/) で定義された `db.system` に `postgres` をセットしています。
```go
db, err := otelsql.Open("mysql", mysqlDSN, otelsql.WithAttributes(
	semconv.DBSystemPostgreSQL,
))
```
この Semantic Conventions のキーが Datadog では `datadog.span.type` としてマッピングされて、`otelsql` で作られたスパンであっても Datadog の世界でのデータベーススパンとして認識されます（= Datadog APM の画面で SQL Queries にスパンが抽出されます）。OTel の Semantic Conventions と Datadog Conventions の対応表は [こちら](https://docs.datadoghq.com/ja/opentelemetry/mapping/semantic_mapping/?tab=datadogexporter) に掲載されいています。

![](/images/2025-advent-calendar-datadog/database_span.png =750x)
*データベーススパンが「SQL Queries」タブにマッピングされる*

ご認識の通りですが、データベーススパンを作成する際に `otelsql` のような自動計装ライブラリを使えないケースもあると思います。その場合は、手動で属性を付与したり、OTel Collector の [Attributes Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/attributesprocessor/README.md) で属性を付与しましょうと[書かれています](https://docs.datadoghq.com/ja/opentelemetry/correlate/dbm_and_traces/?tab=datadogagentotlpingest#using-manual-instrumentation)。

ここまでで、OpenTelemetry のトレースと、DBM のデータがどのように相関付けられているかをふわっと理解しました。

## じゃあ、どう見えるんだい
ちゃんと関連付いていると APM 側から DBM のデータ（特に実行計画やクエリコスト）、DBM 側からトレース情報を参照することができるようになります。便利！

### APM -> DBM
データベーススパンのオーバビューを開くと、クエリの確認とともに、そのクエリ（DBM が収集している正規化されたクエリ）の実行計画を関連づけて見ることができます。
![](/images/2025-advent-calendar-datadog/apm-dbm.png =750x)
*データベーススパンのオーバービュー*

APM のトレースで特定のデータベースクエリに時間がかかっている処理までを特定し、クエリレベルでどこにコストがかかってしまっているかを簡単に確認することが可能です。DBM を開いて探してもいいのですが、画面遷移は少ければ少ないほど嬉しいものです。
![](/images/2025-advent-calendar-datadog/explain.png =750x)
*データベーススパンから関連づいたクエリの実行計画（DBM 側のデータ）を表示*

### DBM -> APM
DBM サイドからも APM の関連づけを確認することができます。これは DBM をお使いのユーザーしかイメージしにくいかもしれませんが、以下のように正規化されたクエリから「Upstream」タブにジャンプして、クエリに関連付いたトレースを表示することができます。もちろんこれを押すことで、APM の分散トレースが確認可能です。いろいろ関連付いてて良いですね。

![](/images/2025-advent-calendar-datadog/dbm-apm.png =750x)
*データベーススパンから関連づいたクエリの実行計画（DBM 側のデータ）を表示*

ここまででデータベーススパンにデータベースの情報を属性として埋め込むことで、DBM と関連付けができ Datadog の使い勝手を向上できるよということを書きました。

## ここからはだいぶ余談です

さらにいうと、DBM ではクエリが「どのサービスの、どのエンドポイント」から呼び出されているかを確認するための「Calling Service」というものも用意されています。

![](/images/2025-advent-calendar-datadog/calling-service.png =750x)
*Calling Service を見ると、当該クエリは otel-go-dbm サービスの `GET` で実行されている*

この Calling Service の関連付けは dd-trace（Datadog が配布している計装ライブラリ）を使った [APM と DBM の関連づけ](https://docs.datadoghq.com/ja/database_monitoring/connect_dbm_and_apm/?tab=go) で高しきにサポートされている機能である。

OTel と DBM の関連付けでは現状公式的なサポートはされておらず、その理由を調べてみたところクエリステートメント一致で関連づけているわけではなくまた別の機構で関連づけているよう仕様のようだった。

冒頭伏線を貼っておいた sqlcommenter が Calling Service 機能における関連づけの要素である。sqlcommenter でクエリに対して traceparent と Datadog 独自のタグ（以下）を埋め込んでいる。これによって、DBM 側から Calling Service としてアプリケーションの情報を引けるということだ。
https://github.com/DataDog/dd-trace-go/blob/6f0ee938db01de410cd062281396b02a6ab9cf61/ddtrace/tracer/sqlcomment.go#L38-L42

sqlcommenter はクエリに対してコメントを埋め込むことができるライブラリで、オブザーバビリティ文脈ではもっぱらトレース ID とかスパン ID を埋め込んでオブザーバビリティバックエンド側で関連づけるために使われます。間瀬さんの OTel Advent Calendar Day1 の記事が参考になります。
https://zenn.dev/makocchan/articles/otel_sql_agent

今回のサンプルで使った `otelsql` でもサポートされているので強引に ddtrace のタグを参考に埋め込んだところ Calling Service にアプリケーションがマッピングされた。実験的に機能検証しただけなので取り扱いにはご留意ください。（とはいえ、付与してる情報は大体スパンから取得できるものなので、コレクターなり、Datadog Agent で自動的にマッピングしてくれると便利そうではあるので、内部でも働きがけてみようと思う）。
https://github.com/XSAM/otelsql/blob/main/commenter.go#L48-L53

## まとめ
データの関連付けはオブザーバビリティにおいて非常に重要な要素です。あらゆるデータを相関させて、より充実した Datadog ライフを歩みましょう！それでは、良いお年を。来年もユーザーグループなどでよろしくお願いします！