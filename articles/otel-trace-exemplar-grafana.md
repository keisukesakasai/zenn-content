---
title: "テレメトリーを関連付けて Grafana で Metrics から Trace にジャンプする"
emoji: "🎪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Go,Observability,Metrics,Grafana]
published: true
---

## はじめに
先日登壇した [OpenTelemetry Observability運用の実例 Lunch LT](https://findy.connpass.com/event/313260/) のフォローアップブログです！
スライドは以下です。[togetter](https://togetter.com/li/2347270) もまとめていただいています。
https://speakerdeck.com/k6s4i53rx/otel-trace-exemplar

途中 Grafana ダッシュボード上で Metrics と Trace を接続する機能を用いて、「異常なメトリクス値からトレースにジャンプしてボトルネックを素早く特定する」というデモをサラッとしたのですが、本ブログではどう設定・構築できるのかをサクッと紹介します。
トレースとメトリクスを紐づけるためのトレースエグザンプラーや、エグザンプラーを使うための OpenTelemetry の設定などについて触れませんので、上記のスライドや ymotongpoo さんのブログ記事が参考になりますので適宜見てください。
https://zenn.dev/google_cloud_jp/articles/20240305-trace-exemplar

## 構成
デモでは下図のような構成で、特筆すべきは以下です。構築パートで詳細に触れていきます。今回はお手元にあった Kubernetes 環境でデモを作ったので Helm を使って各種オブザーバビリティスタックをデプロイしていきます。
- アプリケーションは OpenTelemetry fo Go >= 1.24.0 で計装（エグザンプラーがサポート）
- Prometheus は Feture Flag（`exemplar-strage`）を有効化
- Grafana で `Metrics と Trace を接続する` 設定を行う
![](/images/otel-trace-exemplar-grafana/architecture.png =1000x)

## 構築
アプリケーションはデモ用の [GitHub](https://github.com/keisukesakasai/otel-findy-demo/tree/main/app) を参考にしてください。デモのため、トレースサンプリング設定（`AlwaysSample`）やエグザンプラー設定（`always_on`）であることに留意してください。デプロイすると 8080 ポートでリッスンしているため GET をするとトレースと応答時間をメトリクスとして OTel Collector に送信します。

### オブザーバビリティスタックのデプロイ
Grafana Tempo（以下、Tempo）と Prometheus と Grafana をデプロイしていきます。

Tempo では OTel Collector からトレース情報を OTLP 形式で受け取れるような設定をしています。（`traces.otlp.grpc.enabled=true`）

Prometheus では上記に書いたようにエグザンプラーがアノテートされたメトリクスを取得できるように [Exemplars storage](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage) を有効化しています。また、OTel Collector からメトリクスを Remote Write するため `Remote Write Receiver` も有効化しています。
今回のデモ構成とは異なりますが、アプリケーションでエグザンプラーが付与されたメトリクスを `/metrics` などで配信して、Prometheus 側が Pull 型でスクレイピングする構成でもエグザンプラーの付いたメトリクスを収集できます。
```sh
# Deploy Tempo
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update
$ helm install tempo-distributed grafana/tempo-distributed -n observability --version 1.9.1 --set traces.otlp.grpc.enabled=true --wait

# Deploy Prometheus
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n observability --version 58.0.0  \
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true \
  --set 'prometheus.prometheusSpec.enableFeatures[0]=exemplar-storage' --wait

# Deploy Grafana
$ helm install grafana grafana/grafana -n observability --version 7.3.7 --wait
```

### Grafana 上の設定
データソースとして Tempo と Prometheus を設定し、テレメトリーを可視化します。
Prometheus の設定時 Exemplars の設定があるのでここで以下のような設定を行うことで、メトリクスダッシュボード上に Exemplars として Trace ID が付与されたメトリクスがポイントとしてプロットされ `Metrics と Trace の接続` を行うことができます。
![](/images/otel-trace-exemplar-grafana/metrics_to_trace.png =1000x)

## 実際に見てみる
Proemetheus の Explore を開いてアプリケーションが配信している応答時間のメトリクス `http_request_duration_seconds_bucket` を分析してみます。以下の図では、応答時間の 99%ile をグラフ化しています。画像だと解説しにくいですが、基本的に 100ms 程度で応答をしているが、稀に 3 > 秒程度かかる応答があることが分かります。（アプリケーション側に低頻度で応答を悪くするバグを仕込んでいます。）
また、`Exemplars` を ON にすることでグラフ上にプロットします。グラフで菱形のポイント（赤矢印で示したポイント）がエグザンプラーとして Trace ID がアノテートされたメトリクスです。
![](/images/otel-trace-exemplar-grafana/grafana_metrics.png =1000x)

この菱形のポイントをクリックしてあげると、関連付いているトレース情報に Grafana 上でジャンプすることができます。これにより、メトリクスからより高ディメンションなテレメトリーであるトレースを分析することで処理レベルでどこで遅延しているかを解析することができました！
![](/images/otel-trace-exemplar-grafana/metrics_to_trace_2.png =1000x)

## まとめ
デモで実施した Grafana の Metrics から Trace にジャンプする機能を使う方法を紹介しました。

また、メトリクスからのトレースへのジャンプは Google Cloud Obaservability スタックを使って、Cloud Monitoring と Cloud Trace でも実施できて便利です。そちらに関しては以下のブログ記事にまとめてあるので、興味のある方は見てください。
https://zenn.dev/k6s4i53rx/articles/2023-advent-calendar-google-cloud