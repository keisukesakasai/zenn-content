---
title: "SpanMetricsConnector を使ったアプリケーションメトリクス監視"
emoji: "🤶"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Xmas,Observability,Prometheus,監視]
published: false
---

## はじめに
こんにちは、逆井（さかさい）です。
この記事は [OpenTelemetry Advent Calendar 2023](https://qiita.com/advent-calendar/2023/otel) 21日目の記事です。空きがあったので入れちゃいました。表題の通り、SpanMetricsConnector を紹介するだけ内容なのでサッと読めます！

## SpanMetricsConnector とは

### Connector 
SpanMetricsConnector に入る前に、OpenTelemetry Collector の `Connectors`（以下、コネクター）というコンポーネントについて簡単に紹介します。コネクターはエクスポーターとレシーバーの両方の役割を果たし、2 つのテレメトリーパイプラインを繋ぐ働きをします。

どういうことかイメージをつけるために、OpenTelemetry Collector の以下の Configuration を例に `count` コネクターについて見ていきます。これは、OpenTelemetry の Connectors のドキュメントから拝借しています。
https://opentelemetry.io/docs/collector/configuration/#connectors

```yaml
receivers:
  foo:

exporters:
  bar:

connectors:
  count:
    spanevents:
      my.prod.event.count:
        description: The number of span events from my prod environment.
        conditions:
          - 'attributes["env"] == "prod"'
          - 'name == "prodevent"'

service:
  pipelines:
    traces:
      receivers: [foo]
      exporters: [count]
    metrics:
      receivers: [count]
      exporters: [bar]
```

上の設定では、`count` コネクターを用いて、トレースパイプラインとメトリクスパイプラインを接続し、条件（今回でいうと Span 名と Span の attribute）に合致した Span の数をカウントし、メトリクスとして出力するようにしています。このように Connector を使ってテレメトリーの加工・処理を便利に行うパイプラインを構成することができます。

上記は `count` コネクターですが、他にもいろいろ種類があり、今回紹介する SpanMetricsConnector もその一つです。
https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector

### Span Metrics Connector
Span Metrics Connector はトレースパイプラインとメトリクスパイプラインを接続し、取得した Span から Request, Error and Duration (R.E.D) メトリクスを生成してくれるコネクターです。元々 [Span Metrics Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/spanmetricsprocessor) として提供されていましたがこちらは今は deprecated となっています。

このコネクターを使うことで、アプリケーションに分散トレース計装をしていれば、そこから取得したトレース・スパン情報を使ってアプリケーションのメトリクスを収集することができて便利です。

https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector

## Span Metrics Connector を使ってアプリケーションメトリクスを監視してみる
早速、やっていきましょう。
今回は、適当なアプリケーションから分散トレースを OTel Collector で収集し、Span Metrics Connector を使って Prometheus にメトリクスを Remote Writer していく形にします。

### OTel Collector と Prometheus の起動
まずは、OTel Collector の Configuration です。レシーバーは OTLP/gRPC で、エクスポーターには Prometheus Remote Writer を使っています。
```yaml:otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    target_info:
      enabled: true

connectors:
  spanmetrics:
    histogram:
      explicit:
        buckets: [1ms, 10ms, 100ms, 200ms, 300ms, 400ms, 500ms, 600ms, 700ms, 800ms, 900ms, 1s]
    dimensions:
      - name: http.method
        default: GET
      - name: http.status_code
    exclude_dimensions: ['status.code']
    dimensions_cache_size: 1000
    aggregation_temporality: "AGGREGATION_TEMPORALITY_CUMULATIVE"    
    metrics_flush_interval: 3s
    events:
      enabled: true
      dimensions:
        - name: exception.type
        - name: exception.message

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [spanmetrics]
    metrics:
      receivers: [spanmetrics]
      exporters: [prometheusremotewrite]
```

この Configuration を使って Docker Compose で OTel Collecotr と Prometheus を起動します。
Span Metrics Connector が Contribution Distribution にあるので contrib のイメージを使います。

Prometheus には Remote Write でメトリクスを書き込むため、`--enable-feature=remote-write-receiver` オプションを指定してあげます。

```yaml:compose.yml
services:

  # OTel Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    restart: always
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC receiver

  # Prometheus      
  prometheus: 
    image: prom/prometheus:v2.48.1
    restart: always
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-remote-write-receiver'  
```

これで、`docker compose -f compose.yml up` をすれば、OTel Collector と Prometheus は起動します。

```sh
❯ docker ps
CONTAINER ID   IMAGE                                         COMMAND                   CREATED          STATUS          PORTS                                                        NAMES
b5f1410a6ad1   prom/prometheus:v2.48.1                       "/bin/prometheus --c…"   13 minutes ago   Up 13 minutes   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp                    advent-calender-sre-2023-prometheus-1
d53d9df4dcd9   otel/opentelemetry-collector-contrib:0.91.0   "/otelcol-contrib --…"   13 minutes ago   Up 13 minutes   0.0.0.0:4317->4317/tcp, :::4317->4317/tcp, 55678-55679/tcp   advent-calender-sre-2023-otel-collector-1
```

### Flask を使った Python アプリを自動計装して実行
アプリケーションからテレメトリーを送信します。
Python ではトレースの計装を自動で行ってくれる便利なツールがあるので、それを使ってアプリケーションを実行していきます。以下で、計装をしていない app.py からトレース情報を OTel Collector に送ることができます。

```sh
# Install Library
pip install 'flask<3'
pip install requests

# Install OpenTelemetry Library
pip install opentelemetry-distro==0.41b0
opentelemetry-bootstrap -a install
pip install opentelemetry-sdk==v1.20.0 opentelemetry-exporter-otlp==v1.20.0

# Run App
export OTEL_SERVICE_NAME=SRE_ADVENT_CALENDAR_APP
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

opentelemetry-instrument python app.py
```

これで、Python のアプリからトレース情報が OTel Collector に送られます。

## 準備 OK, テレメトリーを見てみましょう
うまく OTel Collector でトレース情報を取得して、Span Metrics を Prometheus 側に送信できていれば、Prometheus で以下のメトリクスを取得できているはずです。
`duration_milliseconds_sum`, `duration_milliseconds_count`, `duration_milliseconds_bucket` これらが Span Metrics Connector により生成されたメトリクスになります。累積ヒストグラム値である `duration_milliseconds_bucket` は設定値で指定した `buckets` を元に以下のように取得できていることが Prometheus から見ることができます。

この累積ヒストグラムの値を使うことで、以下のようなグラフを作ることができます。今回はアプリケーションのメインハンドラーのルートスパンのメトリクスを生成しているため、このサービスにおけるレイテンシーを可視化できています。
このように、分散トレースのみ取得しているアプリケーションからも、OTel Collector のテレメトリー処理に乗っかることで、メトリクスを収集できる一例を見てみました。とても便利ですね。

## 最後に
今回は OTel Collector の Connectors について、その一つである Span Metrics Connector に触れながらサクッと紹介しました。Connector は比較的新しいコンポーネントですので、ステータスに関しても `In Development` のモノが多いですが、OTel Collector の特徴を活かしてテレメトリーの処理をさらに便利にしてくれるコンポーネントですので今後もウォッチしていきたいです。

明日は [@AoToLog_](https://twitter.com/AoToLog_) の 「OpenTelemetry と〇〇」です。〇〇 に何が入るかは 当日のお楽しみということで楽しみです。