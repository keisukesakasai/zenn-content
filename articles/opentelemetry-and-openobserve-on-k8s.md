---
title: "OpenTelemetry と OpenObserve を使って Kubernetes 監視をかじってみる"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Qiita,Kubernetes,OpenObserve,Observability]
published: false
---

## はじめに
こんにちは！逆井（さかさい）です。
この記事は [OpenTelemetry Advent Calendar](https://qiita.com/advent-calendar/2023/otel) 二日目の記事です 🎄
一日目は [@katzchang](https://twitter.com/katzchang) さんの *** でした。
OTel のアドカレは 2022 年から始まった（多分）と思いますが、去年は `参加者 13 人` だったのに対して、今年は `参加者 *** 人` ということで、Otel 気運の高まりを感じます。

今回は、Otel と [OpenObserve](https://openobserve.ai/) を使って Kubernetes のイベントログやメトリクスの可視化やってみた記事を書こうと思います。最近 OpenObserve でログ・トレース・メトリクス全て OTLP/HTTP で取得できるようになったらしい（嬉しい！）ので、それをやってみたかっただけです。Kubernetes 監視は後付け的になります、、、が、肩休めとして読んでください👋

## OpenObserve とは
[OpenObserve](https://openobserve.ai/) はオープンソースのオブザーバビリティプラットフォームです。現状（2023/11 時点）、ログ・トレース・メトリクスの監視として利用することができます。他にも モダンな UI やアラート機構、テレメトリーのクエリライクな検索などリッチ機能が多くあります。特に、速さやストレージコストの低さを謳っており、各種モニタリング SaaS の代替を目指しているようです。詳しくは、OpenObserve の Introduction video を参照してください。

https://github.com/openobserve/openobserve#introduction-video

OpenObserve を使う場合は、バイナリが配布されているので自前ホストするか、OpenObserve Cloud という SaaS 版を選択する形になります。今回は Kubernetes 上に構築して使ってみます。

### OpenObserve で OTLP がサポートされた
トレースのみ OTLP がサポートされていましたが、2023/09 にリリースされた v0.6.0 から、ログとメトリクスでもサポート（`OTLP/gRPC`, `OTLP/HTTP`）されたみたいです。喜ばしいことです。
https://github.com/openobserve/openobserve/releases/tag/v0.6.0
OpenTelemetry Collector で集約したテレメトリーを OpenObserve に送信するとき、ログでは `Elasticsearch Exporter`、メトリクスでは `Prometheus Remote Write Exporter` を使う必要がありましたが、今回の対応により、全て `OTLP Exporter` で OpenObserve に送ることができるようになりました 🍾

ドキュメントはまだ更新されてないようで、Trace のみ OTLP で Ingestion できるような書き振りとなっているため現在問い合わせ中です。今後更新されると思います。
https://openobserve.ai/docs/user-guide/ingestion/

今回は、ログとメトリクスも OTLP で送れるようになったということで、Kubernetes で適当なテレメトリーを OpenTelemetry Collector で取得して、OpenObserve で可視化するをやっていきます！

## OpenTelemetry Collector のセッティング

```mermaid
graph LR
    classDef class1 fill:#2E6CE6,fill-opacity:0.3
    classDef class2 fill:#F5A800,fill-opacity:0.3
    classDef class3 fill:#E14123,fill-opacity:0.3
    A(Kubernetes):::class1
    A --> B(レシーバー：\nKubernetes Cluster Receiver):::class2 --> D
    A --> C(レシーバー：\nKubernetes Objects Receiver):::class2 --> D
    D(エクスポーター：\nOTLP/HTTP Exporter):::class2 --> |OTLP|E
    E(OpenObserve):::class3
```

### Kubernetes 監視用のレシーバー
Kubernetes では多くのテレメトリーを様々な方法で公開しており、OpenTelemetry Collector の Receiver を使うことで収集することができます。今回は以下の 2 つの Receiver を使ってみます。

- [Kubernetes Cluster Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sclusterreceiver)
Kubernetes Cluster Receiver は、Kubernetes にデプロイされている Pod のフェーズやノードの状態などクラスタ全体に関するメトリクスを収集します。[メトリクス全体はこちら](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/metadata.yaml)にあります。

- [Kubernetes Objects Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sobjectsreceiver)
Kubernetes Cluster Receiver は、Kubernetes API server を用いて Kubernetes イベントログ（例 Pod の生成、削除）を監視します。

これらを組み込んだ OpenTelemetry Collector をデプロイしていきます。OpenTelemetry Collector の Helm Chart を使う場合、values.yaml に `presets` として記述することで簡単に組み込むことができます。以下のように、`clusterMetrics` と `kubernetesEvents` を `true` にするだけで Receiver を構成可能です。簡単ですね。利用可能な Presets の一覧は以下にあります。
https://opentelemetry.io/docs/kubernetes/helm/collector/#presets

```yaml: OpenTelemetry Collector の values.yaml (一部)
mode: deployment

replicaCount: 1

presets:
  # enables the k8sclusterreceiver and adds it to the metrics pipelines
  clusterMetrics:
    enabled: true
  # enables the k8sobjectsreceiver to collect events only and adds it to the logs pipelines
  kubernetesEvents:
    enabled: true
```

### OpenObserve に送信するエクスポーター
送信側は至って簡単です。ログもメトリクスも OTLP 形式で送ることができるので、今回は `OTLP/HTTP Exporter` を使って最もシンプルに構成していきます。

```yaml
config:
  exporters:
    otlphttp:
      # openobserve のエンドポイント
      endpoint: "http://openobserve:5080/api/default"
      # Basic 認証用の情報。こちらを参照：https://openobserve.ai/docs/ingestion/traces/
      headers:
        Authorization: Basic cm9vdEBleGFtcGxlLmNvbTpDb21wbGV4cGFzcyMxMjMK

  service:
    pipelines:
      metrics:
        exporters: [ otlphttp ]
      logs:
        exporters: [ otlphttp ]
```

これで、OpenTelemetry Collector が Kubernetes のテレメトリーを取得して、OpenObserve に送ってくれます。簡単ですね。

## 構築して、テレメトリー取得してみる

### 構築する
OpenObserve は [Quickstart](https://openobserve.ai/docs/quickstart/#openobserve-cloud) のコマンドを使って簡単に Kubernetes にデプロイ可能です。今回はシングルノードモードをデプロイします。[高可用性モード](https://openobserve.ai/docs/ha_deployment/)もあるみたいです。

```sh: OpenObserve デプロイ
$ kubectl create ns openobserve
$ kubectl apply -f https://raw.githubusercontent.com/zinclabs/openobserve/main/deploy/k8s/statefulset.yaml
$ kubectl -n openobserve port-forward svc/openobserve 5080:5080
```

OpenTelemetry Collector も Helm を用いてデプロイします。先述した values.yaml を用いる形にします。

```sh: OpenTelemetry Collector デプロイ
$ helm install -n openobserve otel-collector open-telemetry/opentelemetry-collector \
    --values values.yaml
```

:::details values.yaml 全文
```yaml: values.yaml 全文
mode: deployment

replicaCount: 1

presets:
  # enables the k8sclusterreceiver and adds it to the metrics pipelines
  clusterMetrics:
    enabled: true
  # enables the k8sobjectsreceiver to collect events only and adds it to the logs pipelines
  kubernetesEvents:
    enabled: true

config:
  exporters:
    otlphttp:
      endpoint: "http://openobserve:5080/api/default"
      headers:
        Authorization: Basic cm9vdEBleGFtcGxlLmNvbTpDb21wbGV4cGFzcyMxMjMK
        stream-name: default

  service:
    pipelines:
      metrics:
        exporters: [ otlphttp ]
      logs:
        exporters: [ otlphttp ]
```

:::

これで準備ができました。あとで使う用の 適当な Pod も立ててます 🎄

```sh
$ kubectl get po -n openobserve
NAME                                                              READY   STATUS    RESTARTS   AGE
merry-christmas-deployment-8ddb8b585-7665p                        1/1     Running   0          158m
merry-christmas-deployment-8ddb8b585-mrlsb                        1/1     Running   0          156m
openobserve-0                                                     1/1     Running   0          12h
otel-collector-deployment-opentelemetry-collector-75d54488c5gkh   1/1     Running   0          162m

```

### テレメトリー取得
今回は、Merry Christmas Deployment のレプリカ数を 2 → 3 になったときのイベントログとメトリクスを確認してみます。サービスのレプリカ数を追跡することは大事ですね。(圧)

メトリクスは `k8s_deployment_desired` が取得されているので OpenObserve で見てみましょう。OpenObserve は `http://localhost:5080` でアクセスできます。

確かに `merry-chiristmas-deployment` の `k8s_deployment_desired` が 2 → 3 になったことが確認できました。(赤文字は私の編集です。) メトリクスでは PromQL を使った検索ができるみたいです。

![](/images/opentelemetry-and-openobserve-on-k8s/openobserve_metrics.png =800x)

次にログを見てみます。
ReplicaSet がスケールアウトされているログを確認することができます。(青文字は私の編集です。)これは `kubectl get events`　で確認できるイベントログです。良い感じ。

![](/images/opentelemetry-and-openobserve-on-k8s/openobserve_log.png =800x)

## まとめ
Kubernetes のテレメトリーを OpenTelemetry と OpenObserve を用いてサクッと可視化してみました。

OpenTelemetry のアドカレですが、Kubernetes とか OpenObserve とか何がメインかわからない記事となりました。OpenTelemetry Collector にはレシーバー含む多くのコンポーネントが用意されているので、興味のある方はステータスも気にしながら見てみると良いでしょう。

明日は [@***]() さんの「」です。それでは皆さん、良いお年を。