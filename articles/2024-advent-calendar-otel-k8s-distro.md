---
title: "OTel Collector の Distribution についてと、OpenTelemetry Collector Kubernetes Distro"
emoji: "🤶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Xmas,Observability,Kubernetes,oteltui]
published: true
---

お疲れ様です。
OpenTelemetry Advent Calendar 2024 に空きがあったので、OTel Collector に関するブログを簡単に書いています。
https://qiita.com/advent-calendar/2024/opentelemetry

OpenTelemetry Collector は公式からいくつかの[ディストリビューション](https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions)が配布されています。2024/11 段階では 4 つあります。
 * コアディストリビューション（otelcol-core）
 * コントリビューションディストリビューション（otelcol-contrib）
 * Kubernetes ディストリビューション（otelcol-k8s）
 * OTLP ディストリビューション（otelcol-otlp）

:::message
下二つの otelcol-k8s と otelcol-otlp のディストリビューションは比較的新しく、otelcol-k8s は v0.98.0（2024/04）、otelcol-otlp は v0.111.0（2024/10）に入っています。
:::

先日あった CloudNative Days Winter 2024 でも少し触れましたが、この記事では Kubernetes ディストリビューションのコンポーネントについて見ていこうと思います。
https://speakerdeck.com/k6s4i53rx/getting-started-opentelemetry-collector-with-opamp?slide=17

### OpenTelemetry Collector Kubernetes Distro
Kubernetes や Kubernetes 上のサービスのテレメトリーシグナルを収集するための OTel Collector コンポーネントを含んでいるディストリビューションです。ベンダー製のコンポーネントを含まずステータスは Alpha 以上のものが含まれます。
Kubernetes ディストリビューションのコンポーネントの一覧は [manifest.yaml](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/main/distributions/otelcol-k8s/manifest.yaml) に記述されていて、OTLP 関連や core ディストリビューションに含まれているレシーバーとともに、下記のような Kubernetes のためのコンポーネントが入っています。
 * Kubernetes Attributes Processor
 * Kubernetes Cluster Receiver
 * Kubernetes Events Receiver
 * Kubernetes Objects Receiver

全量の説明は各コンポーネントの README に委ねますが、Kubernetes Cluster Receiver により Pod のフェーズやノードの状態などクラスタ全体に関するメトリクスを収集したり、Kubernetes Objects Receiver により Kubernetes イベントログ（例 Pod の生成、削除）を収集します。

このようにその名の通りですが、Kubernetes で OTel Collector を構成する場合に必要なコンポーネントが含まれています。

### otelcol-k8s を使ってみる
とても簡単に otelcol-k8s を動かして Kubernetes のイベントを OpenTelemetry で収集してみましょう。Kubernetes に以下のような config で otelcol-k8s をデプロイします。
```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:
  # Kubernetes Cluster Receiver の構成
  k8s_cluster:
    auth_type: serviceAccount
  # Kubernetes Objects Receiver の構成
  k8sobjects:
    auth_type: serviceAccount

processors:
  batch:
  memory_limiter:
    limit_mib: 400
    spike_limit_mib: 100
    check_interval: 5s

exporters:
  otlp:
    endpoint: oteltui.otelcol-k8s-distro-advent.svc.cluster.local:4317
    tls:
      insecure: true      

service:
  pipelines:
    metrics:
      receivers: [otlp, k8s_cluster]
      processors: [batch, memory_limiter]
      exporters: [otlp]
```

デプロイをすると Kubernetes のイベントを OTel Collector が収集し始めます。ここでは [ymtdzzz](https://x.com/ymtdzzz) さんの [otel-tui](https://github.com/ymtdzzz/otel-tui) に OTLP でメトリクスを送信し可視化してみます。otel-tui は OTLP のシグナルを可視化する素晴らしいツールで重宝をしています。

![](/images/2024-advent-calendar-otel-k8s-distro/fig.png =900x)

このように Kubernetes Cluster Receiver が生成したメトリクスを確認することができました。

### まとめ
OTel が配布している OpenTelemetry Collector のディストリビューションについて紹介をしました。自分たちの使いたいコンポーネントのみを構成してカスタムの OTel Collector を構築することも可能です。その場合は、ocb（OpenTelemetry Collector Builder）を使ってビルドします。

https://zenn.dev/k6s4i53rx/articles/df59cb65b34ef8