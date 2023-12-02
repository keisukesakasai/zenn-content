---
title: "OpenTelemetry と OpenObserve を使って Kubernetes 監視をかじる"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Xmas,Kubernetes,OpenObserve,Observability]
published: false
---

## ◾️ はじめに
こんにちは！逆井（さかさい）です。
この記事は [OpenTelemetry Advent Calendar 2023](https://qiita.com/advent-calendar/2023/otel) 二日目の記事です 🎄
一日目は [@katzchang](https://twitter.com/katzchang) さんの [OpenTelemetry Meetupを開催しました](https://zenn.dev/katzchang/articles/e5192f2fddda0a) でした。

OTel のアドカレは [2022 年](https://qiita.com/advent-calendar/2022/opentelemetry) から始まった（たぶん）と思いますが、去年は `参加者 13 人` だったのに対して、今年は `参加者 25 人` ということで、OTel 気運の高まりを感じます。（表記は `24 人` ですが 12/12 は代理でわたしが枠取ってるので実質 `25 人` です。）

今回は、OTel と [OpenObserve](https://openobserve.ai/) を使って Kubernetes のイベントログやメトリクスの可視化やってみた記事を書こうと思います。OpenObserve ではログ・トレース・メトリクスを OTLP で取得できる（嬉しい！）ので、それをやってみたかった記事です。Kubernetes 監視は少し後付け的になりますが、アドカレ大量発生中の肩休めとして読んでください👋👋

## ◾️ OpenObserve 使っていきたい
[OpenObserve](https://openobserve.ai/)（以下、O2）はオープンソースのオブザーバビリティプラットフォームです。現状（2023/11 時点）、ログ・トレース・メトリクスの監視として利用することができます。他にもモダンな UI やアラート機能、テレメトリーのクエリライクな検索などリッチ機能が多くあります。特に、速さやストレージコストの低さを謳っており、各種モニタリング SaaS の代替を目指しているようです。詳しくは、O2 の [Introduction Video](https://github.com/openobserve/openobserve#introduction-video) を参照してください。

O2 を使う場合は、バイナリが配布されているので自前ホストするか、OpenObserve Cloud という SaaS 版を選択する形になります。今回は Kubernetes 上に構築して使っていきます。

### O2 のログ / トレース/ メトリクスで OTLP がサポートされたぞ！
トレースのみ OTLP がサポートされていましたが、2023/09 にリリースされた v0.6.0 から、ログとメトリクスでもサポート（`OTLP/gRPC`, `OTLP/HTTP`）されたみたいです。喜ばしいことです。
https://github.com/openobserve/openobserve/releases/tag/v0.6.0
OTel Collector で集約したテレメトリーを O2 に送信するとき、ログでは `Elasticsearch Exporter`、メトリクスでは `Prometheus Remote Write Exporter` を使う必要がありましたが、v0.6.0 より全て `OTLP Exporter` で O2 に送ることができます 🍾

ドキュメントはまだ更新されてないようで、Trace のみ OTLP で Ingestion できるような書き振りとなっているため問い合わせてみたところ、今後更するとの回答をいただきました。
https://openobserve.ai/docs/user-guide/ingestion/

今回は、ログとメトリクスも OTLP で送れるようになったということで、Kubernetes で適当なテレメトリーを OTel Collector で取得して、O2 で可視化するをやっていきます！

## ◾️ OTel Collector を使っていきます

```mermaid
graph LR
    classDef class1 fill:#2E6CE6,fill-opacity:0.3,stroke:#2E6CE6
    classDef class2 fill:#F5A800,fill-opacity:0.3,stroke:#F5A800
    classDef class3 fill:#E14123,fill-opacity:0.3,stroke:#E14123
    A(Kubernetes):::class1
    A --> B(レシーバー：\nKubernetes Cluster Receiver):::class2 --> D
    A --> C(レシーバー：\nKubernetes Objects Receiver):::class2 --> D
    D(エクスポーター：\nOTLP/HTTP Exporter):::class2 --> |OTLP|E
    E(OpenObserve):::class3
```

### Kubernetes 監視するレシーバーについて
Kubernetes では多くのテレメトリーを様々な方法で公開しており、OTel Collector の Receiver を使うことで収集することができます。今回は以下の 2 つの Receiver を使ってみます。

- [Kubernetes Cluster Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sclusterreceiver)
Kubernetes Cluster Receiver は、Kubernetes にデプロイされている Pod のフェーズやノードの状態などクラスタ全体に関するメトリクスを収集します。[メトリクス全体はこちら](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/metadata.yaml)！

- [Kubernetes Objects Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sobjectsreceiver)
Kubernetes Objects Receiver は、Kubernetes API server を用いて Kubernetes イベントログ（例 Pod の生成、削除）を監視します。

これら Receiver を組み込んだ OTel Collector をデプロイします。
OTel Collector の Helm Chart を使う場合、今回のレシーバーたちは `values.yaml` に `presets` として記述することで簡単に組み込むことができます。
以下のように、`clusterMetrics` と `kubernetesEvents` を `true` にするだけで Receiver を構成可能です。簡単ですね。利用可能な Presets の一覧は [こちら](https://opentelemetry.io/docs/kubernetes/helm/collector/#presets) にあります。

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

### O2 へ送信するエクスポーターについて
送信側は至って簡単です。ログもメトリクスも OTLP 形式で送ることができるので、今回は `OTLP/HTTP Exporter` を使ってとてもシンプルに構成していきます。

```yaml
config:
  exporters:
    otlphttp:
      # O2 のエンドポイント
      endpoint: "http://openobserve:5080/api/default"
      # Basic 認証用の情報。こちらを参照: https://openobserve.ai/docs/ingestion/traces/
      headers:
        Authorization: Basic cm9vdEBleGFtcGxlLmNvbTpDb21wbGV4cGFzcyMxMjMK

  service:
    pipelines:
      metrics:
        exporters: [ otlphttp ]
      logs:
        exporters: [ otlphttp ]
```

これで、OTel Collector が Kubernetes のテレメトリーを取得して、O2 に送ってくれます。非常に簡単ですね 🚀

## ◾️ 実際にテレメトリー取得してみる

### 構築
O2 は [Quickstart](https://openobserve.ai/docs/quickstart/#openobserve-cloud) のコマンドを使って Kubernetes にデプロイ可能です。
今回は検証のためシングルノードモードをデプロイします。[高可用性モード](https://openobserve.ai/docs/ha_deployment/)もあるみたいです。

```sh: O2 デプロイ
$ kubectl create ns openobserve
$ kubectl apply -f https://raw.githubusercontent.com/zinclabs/openobserve/main/deploy/k8s/statefulset.yaml
$ kubectl -n openobserve port-forward svc/openobserve 5080:5080
```

OTel Collector も Helm を用いてデプロイします。先述した values.yaml を用いる形にします。

```sh: OTel Collector デプロイ
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

これで準備ができました。あとで使う用の 適当な Pod もデプロイしておきます 🎄

```sh
$ kubectl get po -n openobserve
NAME                                                              READY   STATUS    RESTARTS   AGE
merry-christmas-deployment-8ddb8b585-7665p                        1/1     Running   0          158m
merry-christmas-deployment-8ddb8b585-mrlsb                        1/1     Running   0          156m
openobserve-0                                                     1/1     Running   0          12h
otel-collector-deployment-opentelemetry-collector-75d54488c5gkh   1/1     Running   0          162m
```

### テレメトリー見ていく
今回は、Merry Christmas Deployment のレプリカ数が 2 → 3 になったときのイベントログとメトリクスを確認してみます。サービスのレプリカ数を追跡することは大事です(圧)。

メトリクスは `k8s_deployment_desired` が取得されているので O2 で見てみましょう。O2 のダッシュボードは `http://localhost:5080` でアクセスできます。

確かに `merry-chiristmas-deployment` の `k8s_deployment_desired` が 2 → 3 になったことが確認できました。(赤文字は私の編集です。) メトリクスでは PromQL を使った検索ができます。

![](/images/opentelemetry-and-openobserve-on-k8s/openobserve_metrics.png =800x)

次にログを見てみます。
ReplicaSet がスケールアウトされているログを確認することができます。(青文字は私の編集です。)これは `kubectl get events` で確認できるイベントログです。良い感じ。

![](/images/opentelemetry-and-openobserve-on-k8s/openobserve_log.png =800x)

## ◾️ 最後に
今回は、Kubernetes のテレメトリーを OTel with O2 でサクッと可視化してみました。

モニタリングにおけるテレメトリーの収集は、様々なプログラミング言語の計装ライブラリやエクスポーターを用意する必要がありきっと大変です（プロダクト開発者目線）。しかし OpenTelemetry に乗っかることで、OTel Collector のエクスポーターのみ Go を使って用意（OTel Collector は Go で書かれています）しさえすれば、アプリからは OTel を使い OTLP 標準で様々な言語のアプリのテレメトリー収集に対応可能になります。これはモニタリングプロダクト開発目線でメリットなのかなと思います。また、今回見てきた O2 のように OTLP をネイティブにサポートすることで、エクスポーターさえも OTel に委ねることができ、恐らくしんどいであろうテレメトリー収集の大部分をオフロードできるようになります。OTel による標準化の動向はアプリケーション開発者のみならず、様々な方面に恩恵をもたらし得る流れのように感じますね。今後も OTel の発展に着目し、できることならば貢献もしていきたいなぁと思います。

三日目は [@sumiren_t](https://twitter.com/sumiren_t) さんです！それでは皆さん、良いお年を。