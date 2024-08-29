---
title: "Google Cloud Client Libraries for Go のデフォルト計装が OTel になった"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GoogleCloud,Go,OpenTelemetry,Observability,Memo]
published: true
---

[Google Cloud Client Libraries for Go](https://github.com/googleapis/google-cloud-go/tree/main) は、Go アプリから Google Cloud のさまざまなリソースを操作するためのクライアントライブラリを提供しています。また、クライアントライブラリは内部の挙動や性能を把握するための方法を備えています。
https://github.com/googleapis/google-cloud-go/blob/main/debug.md#logging-debugging-and-telemetry

一部のクライアントライブラリ^[[リンク先](https://github.com/googleapis/google-cloud-go/blob/main/debug.md#tracing-experimental)に列挙されているライブラリ]は分散トレースのための計装が施されており（`experimental`）、従来 OpenCensus での計装でしたが、2024/05 に OpenTelemetry でのトレース計装がデフォルトになりました^[https://github.com/googleapis/google-cloud-go/pull/10287 で対応がされました]。以前までは OpenTelemetry で計装してるアプリとクライアントライブラリのトレースを接続するためには、以下のいずれかの対応が必要でした。
- アプリ側で [OpenCensus Bridge](https://pkg.go.dev/go.opentelemetry.io/otel/bridge/opencensus) を使用する
- クライアントライブラリ側で OpenTelemetry の使用を設定する（`GOOGLE_API_GO_EXPERIMENTAL_TELEMETRY_PLATFORM_TRACING`=`opentelemetry`）

[#10287](https://github.com/googleapis/google-cloud-go/pull/10287) により OpenTelemetry 計装でデフォルトで繋がります。喜ばしいです。

:::message
`GOOGLE_API_GO_EXPERIMENTAL_TELEMETRY_PLATFORM_TRACING=opencensus` にすることで非推奨ではありますが、OpenCensus の使用を設定することが可能です。ただし、`2024/12/02` に削除が予定されているので注意が必要です。
https://github.com/googleapis/google-cloud-go/blob/main/debug.md#opencensus
:::

OpenTelemetry でトレース計装しているアプリから `cloud.google.com/go/datastore` を使って Datastore に適当にデータを Put してみます。Cloud Trace ではアプリ側のスパンと、クライアントライブラリ側のスパンを確認することができます。

![](/images/gc-client-library-opentelemetry-instrumented/cloud_trace.png =900x)

余談ですが、上図のスパンが実際にクライアントライブラリ側で生成されているコードは [こちら](https://github.com/googleapis/google-cloud-go/blob/main/datastore/datastore.go#L645) です。トレースをすることで性能的な解析はもちろんですが、どういう流れで処理をしているかを大雑把に把握できるので、クライアントライブラリのソースコードを実際に読み解く必要があるときなどに便利です。

今回は Google Cloud Client Libraries for Go のデフォルト計装が OpenTelemetry になったのでメモブログ記事を書きました。