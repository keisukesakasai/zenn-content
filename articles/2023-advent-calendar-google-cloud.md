---
title: "Google Cloud Managed Service for Prometheusを使ってメトリクスとトレースを紐付け O11y 強化！"
emoji: "🎅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Xmas,GCP,prometheus,Observability,Exemplar]
published: true
---

## はじめに
こんにちは、逆井（さかさい）と申します。
この記事は Google Cloud Advent Calendar 2023 の 23 日目の記事になります 🎄
https://qiita.com/advent-calendar/2023/gcp

前日は [@takashabe](https://twitter.com/takashabe) さんの [Cloud Runでログとトレースを紐づけてObservabilityを高める](https://zenn.dev/castingone_dev/articles/20231222-gcp_o11y) でした。こちらの記事では「ログとトレース」を紐付け可観測性を高める方法が述べられています！

本記事では「メトリクスとトレース」を紐付けることでシステム状態の解析性を向上させる方法について書いていきます。オブザーバビリティにおいて、テレメトリーデータ（ログやトレース、メトリクスなど）が相互に紐付いて収集されていることはとても重要な要素です。

22、23日目の記事でテレメトリーを紐付ける方法を一気見していきましょう！（並びは偶然）

## 前提
メトリクスとトレースの紐付けのために、Google Cloud Managed Service for Prometheus（以下、GMP）や Cloud Monitoring, Cloud Trace を使用します。本記事では紐付ける方法についてフォーカスするため、これらプロダクトや使用方法についての説明は省略します。

## メトリクスとトレースの紐付け（座学）

### エグザンプラ（Exemplar）について
最初に、メトリクスとトレースを紐付けるために用いるエグザンプラを簡単に紹介します。Exampler ではなく、Exemplar です！以下、エグザンプラと表記します。

Prometheus が収集するような OpenMetrics 形式のメトリクスは以下のようなフォーマットで生成されます。
```sh
# TYPE foo histogram
foo_bucket{le="0.01"} 0
foo_bucket{le="1"} 11
foo_bucket{le="10"} 17
foo_bucket{le="+Inf"} 17
foo_count 17
foo_sum 324789.3
```
これは一般的な Histgram 型メトリクスのサンプルです。見馴染みの深い方も多いでしょう。このメトリクスに対して OpenMetrics エグザンプラを用いて付与情報を追加することができます。
https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#exemplars-1
エグザンプラとしてトレース ID を付与したメトリクスは以下のような形です。`#` の後にラベルとして情報を追加することができます。
```sh
# TYPE foo histogram
foo_bucket{le="0.01"} 0
foo_bucket{le="1"} 11 # {trace_id="KOO5S4vxi0o"} 0.67
foo_bucket{le="10"} 17 # {trace_id="oHg5SJYRHA0"} 9.8 1520879607.789
foo_bucket{le="+Inf"} 17
foo_count 17
foo_sum 324789.3
foo_created  1520430000.123
```

:::message

### エグザンプラによる、メトリクスとトレースの相関
エグザンプラにより、メトリクスに対してトレース情報を紐付けることができます。

これにより、メトリクスから分散トレースへの遷移が可能になり、サービスのパフォーマンス異常が発生した際に、メトリクスの突然の変化に関連する分散トレースを素早く発見することができます。

エグザンプラを使用せず、個別にメトリクスとトレースを保持している場合は、サービスで何が起きているかを特定するために、タイムスタンプなどを使用して手動で各テレメトリーを相互に参照する必要があり手間ですね。きっとあるあるですが。

まさに、オブザーバビリティ界隈でよく目にする "looking for a needle in a haystack"（膨大な情報の中から探しているものを、目的を持って即座に見つける）に通じます。
:::

### Google Cloud のサービスを使う
GMP を使うとエグザンプラを取り込むことができます！！
また取り込んだメトリクスを Cloud Monitoring の Metrics Explorer で可視化すると、エグザンプラから関連するトレース情報をダッシュボード上にプロットし Cloud Trace への遷移が可能です。

以降のパートでは、サンプルアプリケーションを用いることでアプリケーションメトリクスを取得し、レイテンシが大きくなった際のエグザンプラーからトレース情報に遷移しアプリケーション内部のパフォーマンスを詳細に見ていきます。これによりトラブルシューティングを簡便化でき、エグザンプラを使ったメトリクスとトレース情報の紐付けのメリットをデモ的に紹介します。

エグザンプラがサポートされている GMP コレクションのバージョンや、メトリクスの型（例えば、GMP の場合は、ヒストグラム型に適用されたエグザンプラのみ取り込むことが可能）、エグザンプラの保持期間についてはドキュメントを参照してください。
https://cloud.google.com/stackdriver/docs/managed-prometheus/exemplars?hl=ja

## メトリクスとトレースの紐付け（やっていき）
まずは監視対象であるアプリケーションのデプロイと、監視に必要な準備をします。今回は Google Kubernetes Engine（GKE）上で検証を進めていきます。

### アプリケーションのデプロイ
アプリケーションは Go で簡単なサーバーを書きました。特筆すべき点は以下です。
- OpenTelemetry を使って分散トレース計装
- prometheus/client_golang パッケージを使ってメトリクス計装
  - `/` エンドポイントに対するリクエストの Duration を Histgram 型 メトリクスとして収集
  - メトリクスにエグザンプラとして `trace_id` / `span_id` / `project_id` を付与
  - そのメトリクスを `/metrics` エンドポイントで配信

このアプリケーションを Kubernetes にデプロイして、`/metrics` エンドポイントからメトリクスをスクレイピングしてあげると以下のようなメトリクスを取得できます。
エグザンプラとしてメトリクスにトレース情報が付与されていることがわかります。
```sh: /metrics エンドポイントに curl を投げる
$ curl -H 'Accept: application/openmetrics-text' localhost:8080/metrics
# HELP http_request_duration_seconds A histogram of the HTTP request durations in seconds.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.01"} 0
http_request_duration_seconds_bucket{le="0.02"} 0
http_request_duration_seconds_bucket{le="0.04"} 0
http_request_duration_seconds_bucket{le="0.08"} 0
http_request_duration_seconds_bucket{le="0.16"} 0
http_request_duration_seconds_bucket{le="0.32"} 1 # {project_id="***",trace_id="656858120dc09acecf8baa331d736577",span_id="0a9cf4dc142a9fa6"} 0.240129272 1.7034402922686768e+09
http_request_duration_seconds_bucket{le="0.64"} 1
http_request_duration_seconds_bucket{le="1.28"} 1
http_request_duration_seconds_bucket{le="2.56"} 1
http_request_duration_seconds_bucket{le="5.12"} 1
http_request_duration_seconds_bucket{le="10.24"} 1
http_request_duration_seconds_bucket{le="+Inf"} 1
http_request_duration_seconds_sum 0.240129272
http_request_duration_seconds_count 1
```

:::details アプリケーションのコード全文

```go:main.go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"

	texporter "github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/trace"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"go.opentelemetry.io/contrib/detectors/gcp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

var tracer = otel.Tracer("EXEMPLAR-APP")

func main() {
	// OpenTelemetry Traces
	tp, err := setupTracerProvider()
	if err != nil {
		log.Fatalf("Error setting up trace provider: %v", err)
	}
	defer func() { _ = tp.Shutdown(context.Background()) }()

	// Define Metrics
	requestDurations := prometheus.NewHistogram(prometheus.HistogramOpts{
		Name:    "http_request_duration_seconds",
		Help:    "A histogram of the HTTP request durations in seconds.",
		Buckets: prometheus.ExponentialBuckets(0.01, 2, 11),
	})

	registry := prometheus.NewRegistry()
	registry.MustRegister(
		requestDurations,
	)

	// Metrics Endpoint
	http.Handle(
		"/metrics", promhttp.HandlerFor(
			registry,
			promhttp.HandlerOpts{EnableOpenMetrics: true}),
	)

	// Main Handler
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		ctx := context.Background()

		ctx, span := tracer.Start(ctx, "Root Span")
		defer span.End()

		start := time.Now()

		func1(ctx)
		func2(ctx)

		// Dummy Logic
		time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))

		// Record Metrics
		duration := time.Since(start).Seconds()
		traceID := span.SpanContext().TraceID().String()
		spanID := span.SpanContext().SpanID().String()
		requestDurations.(prometheus.ExemplarObserver).ObserveWithExemplar(
			duration, prometheus.Labels{
				"project_id": "***",
				"trace_id":   traceID,
				"span_id":    spanID,
			},
		)

		fmt.Fprintf(w, "Request took %v seconds\n", duration)
	})

	log.Fatalln(http.ListenAndServe(":8080", nil))
}

func func1(ctx context.Context) {
	_, span := tracer.Start(ctx, "Func1")
	defer span.End()

	time.Sleep(time.Millisecond * time.Duration(rand.Intn(100)))
}

func func2(ctx context.Context) {
	_, span := tracer.Start(ctx, "Func2")
	defer span.End()
	rand.NewSource(time.Now().UnixNano())

	if rand.Float64() < 1.0/100.0 {
		time.Sleep(3 * time.Second)
	} else {
		time.Sleep(10 * time.Millisecond)
	}
}

func setupTracerProvider() (*sdktrace.TracerProvider, error) {
	ctx := context.Background()
	projectID := "***"
	exporter, err := texporter.New(texporter.WithProjectID(projectID))
	if err != nil {
		return nil, err
	}

	res, err := resource.New(ctx,
		resource.WithDetectors(gcp.NewDetector()),
		resource.WithTelemetrySDK(),
		resource.WithAttributes(
			semconv.ServiceNameKey.String("EXEMPLAR-APP"),
		),
	)
	if err != nil {
		log.Fatalf("resource.New: %v", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
	)
	otel.SetTracerProvider(tp)

	return tp, nil
}
```

:::

### メトリクス監視
GKE の場合、Autopilot の場合は 1.25 以降、Standard の場合は 1.27 以降で GMP コンポーネントであるマネージドコレクション（Prometheus ベースのコレクタ）が有効化されています。
```sh: GKE にデプロイされている GMP コンポーネント
$ kubectl get po -ngmp-system
NAME                              READY   STATUS    RESTARTS      AGE
alertmanager-0                    2/2     Running   0             38h
collector-b5sbg                   2/2     Running   0             38h
collector-b8l4g                   2/2     Running   0             38h
gmp-operator-8547d84898-pcjn9     1/1     Running   0             38h
rule-evaluator-74bc75b4f7-2h6pq   2/2     Running   2 (38h ago)   38h
```

先ほどのアプリケーションのメトリクスを取り込むために [PodMonitoring カスタムリソース](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed?hl=ja#gmp-pod-monitoring)を作成します。これにより、メトリクスを自動で収集してくれます。今回は 3 秒間隔でスクレイピングするよう設定をしています。

```yaml:pod-monitoring.yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: gcp-advent-calendar
  labels:
    app.kubernetes.io/name: gcp-advent-calendar
spec:
  selector:
    matchLabels:
	  # 監視対象をラベルで選択
      app.kubernetes.io/name: gcp-advent-calendar
  endpoints:
  - port: metrics
    interval: 3s
```

```sh: デプロイした PodMonitoring リソースを確認
$ kubectl get podmonitoring
NAME                  AGE
gcp-advent-calendar   4s
```

:::message
GMP の設定である [OperatorConfig の設定値で「ターゲットステータス機能」](https://cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed?hl=ja#target-status)があります。
デフォルトでは無効化されていますが、有効化することにより PodMonitoring の監視ターゲットのステータスを `kubectl describe` で出力できるようになり便利です。

この設定を有効化し、PodMonitoring を確認すると、確かにサンプルアプリケーションを監視し、ステータスが Active であることが確認できます。

```sh: PodMonitoring を describe して監視対象アプリケーションのステータスを確認
❯ kubectl describe podmonitoring gcp-advent-calendar
Name:         gcp-advent-calendar
Namespace:    google-cloud-advent-calendar
...
  Endpoint Statuses:
    Active Targets:       1
    Collectors Fraction:  1
    Last Update Time:     2023-12-24T18:02:11Z
    Name:                 PodMonitoring/google-cloud-advent-calendar/gcp-advent-calendar/metrics
    Sample Groups:
      Count:  1
      Sample Targets:
        Health:  unknown
        Labels:
          ...
          Container:                   gcp-advent-calendar
        Last Error:
        Last Scrape Duration Seconds:  0
    Unhealthy Targets:                 1
  Observed Generation:                 1
```
:::

### さて、収集したメトリクスを見ていきましょう
アプリケーションから収集したメトリクスである `http_request_duration_seconds` を Metrics Explorer のヒートマップで見てみましょう。

見慣れない方もいるかもしれませんが、そこまで難しくなく、X 軸方向は時間、Y 軸方向に Duration の箱（単位：秒）があり、色がその箱に含まれているデータ数（分布）になります。結果を見ると、基本的に 1 秒弱でレイテンシーが分布しているようです。白線で見えている 95 パーセンタイルの線もその辺をうようよしていることが確認できます。

![](/images/2023-advent-calendar-google-cloud/heatmap.png =900x)

グラフの中にいくつかポイントがあることが分かりますが、これがエグザンプラでトレース情報が付与されたメトリクスです。ポイントを押下すると、トレース情報に基づいて、Cloud Trace の画面を表示してくれます。アプリケーションの各処理でどのくらいの処理時間がかかっているかが分かります。

![](/images/2023-advent-calendar-google-cloud/metrics-trace-1.png =900x)

ヒートマップ上部を見ると、稀に 3 秒くらい応答に時間を要しているリクエストがありそうなこともわかります。（アプリケーションに 1 % くらいで Sleep を仕込んでいるため）

ここにも、エグザンプラによるサンプルトレースがあるのでクリックしてみます。その結果が以下です。上で確認した典型的なメトリクス値のトレースに比べると、`Func2` での処理時間が大きくなっていることが分散トレースから簡単に取得することができます 🎉🎉

![](/images/2023-advent-calendar-google-cloud/metrics-trace-2.png =900x)

このように、メトリクスと分散トレースを紐付けることで、メトリクスの異常値を俯瞰し、さらにそこからアプリケーション内部の処理を細かく見ることができる分散トレースへドリルダウン的に探索しにいけます。エグザンプラを使ったテレメトリーの紐付けはトラブルシューティングにおいても便利です。ぜひやってみてください。

## まとめ
GMP や Clout Monitoring, Cloud Trace を使ってメトリクスとトレースを紐づけてみました。聞き馴染みのないエグザンプラについて知るきっかけになれば嬉しいです！

この記事ではあまり Histgram やエグザンプラについて詳細触れられていません。
[Cloud Native Days Tokyo 2022](https://cloudnativedays.jp/cndt2022) のセッションでもう少し詳しく解説したのでもし興味がある方がいらっしゃったらぜひ見てみてください。
https://speakerdeck.com/k6s4i53rx/cndt2022-shi-jian-opentelemetry-to-oss-woshi-tuta-observability-ji-pan-nogou-zhu?slide=20

それでは皆さん良いお年を！

## 参考
- OREILLY, オブザーバビリティ・エンジニアリング
- https://cloud.google.com/stackdriver/docs/managed-prometheus/exemplars?hl=ja
- https://cloud.google.com/blog/ja/products/devops-sre/trace-exemplars-now-available-in-managed-service-for-prometheus