---
title: "BindPlane を使ってオブザーバビリティパイプラインをリモート管理し、Cloud Trace にトレースを送ってみる"
emoji: "🤶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GoogleCloud,Bindplane,Xmas,Observability,OpenTelemetry]
published: true
---

## はじめに
こんにちは。Google Cloud Champion Innovator 逆井です。この記事は Champion Innvotor アドベントカレンダーの 21 日目の記事になります。納品が少し遅れてしまいました。
https://adventar.org/calendars/10061

AI ネタが盛り上がっていますが、オブザーバビリティが好きなので、今回は（も）オブザーバビリティのネタを書きます。本記事では、BindPlaneを使って Google Cloud にテレメトリーシグナルを送信するパイプラインをリモート管理するする方法をハンズオン的に紹介しようと思います 👋

## まず BindPlane とはを簡単に
BindPlane は [observIQ](https://observiq.com/) が開発している、オブザーバビリティパイプラインやテレメトリエージェントを大規模に管理するためのプロダクトです。BindPlane OP というエージェント群を管理するための UI を持つコントロールプレーンと、BindPlane Agent から構成されます。BindPlane Agent は OpenTelemetry Collector の BindPlane ディストリビューションです。

BindPlane Agent でテレメトリーシグナルの受信、処理、送信を行い、その BindPlane Agent を BindPlane OP から制御、管理します。送信先（Destinations）も[多くをサポート](https://observiq.com/docs/resources/destinations)しており、このあと書く Google Cloud もサポートされています。わたしが所属している Datadog もサポート範囲内です。テレメトリー受信元（Sources）の[ラインナップも豊富](https://observiq.com/docs/resources/sources)で、今回は OpenTelemetry を使っていきます。下記のアーキテクチャは[ドキュメント](https://observiq.com/docs/getting-started/quickstart-guide)引用しています。

![](/images/2024-advent-calendar-champ/bindplane.png =650x)

ここでは深く触れませんが、BindPlane OP は SaaS とオンプレミスを選択できたり、いくつかのエディションが存在しています。詳しくは BindPlane の[ソリューションページ](https://observiq.com/solutions)を参照してください。

### 落穂拾い...🍂
BindPlane OP が BindPlane Agent（OTel Collector）を管理する仕組みは Open Agent Management Protocol（OpAMP）です。OpAMP は大規模エージェント群をリモート管理するためのネットワークプロトコルで OpenTelemetry で管理されています。
https://opentelemetry.io/docs/specs/opamp/
OpAMP はもともと BindPlane で使用されていたカスタムプロトコルが源流にあります（[ソース](https://opentelemetry.io/blog/2023/opamp-status/)）。
BindPlane OP が OpAMP サーバーであり、BindPlane Agent で OpAMP クライアントが起動しています。BindPlane Agent 自体は GitHub で[公開](https://github.com/observIQ/bindplane-otel-collector)されています。OpAMP については今年の Cloud Native Days で取り上げて話たので気になるかたは見てみてください。このセッションで出てくる OpAMP バックエンド（サンプル）のめちゃくちゃリッチ版が BindPlane OP です。
https://speakerdeck.com/k6s4i53rx/getting-started-opentelemetry-collector-with-opamp

## セットアップ
実際に動かしてみましょう。ワイワイ。以下のような構成です。
![](/images/2024-advent-calendar-champ/archi.png =650x)
GKE 上にトレース計装したアプリと、BindPlane OP / Agent をデプロイします。BindPlane OP の UI からオブザーバビリティパイプラインをリモート操作して、トレース情報に適当なタグ（`Advent:Calendar...!!`）を付与してみます。簡単にセットアップ手順も見て行きます。

### BindPlane OP デプロイ
[Helm](https://github.com/observIQ/bindplane-op-helm) が用意されており、[手順](https://observiq.com/docs/advanced-setup/kubernetes-installation/server/install)に従って簡単にデプロイができます。`Single Instance`、`High Availability` モードがあり、今回はデモなので `Single Instance` モードでセットアップします。

BindPlane のライセンスが必要なので [Free](https://observiq.com/download) で作成をしました。また、割と大きめのリソースリクエストが設定されているのでデプロイ時はご留意ください。
```yaml
resources:
  # Request 2 cores and allow cpu bursting.
  # Request fixed amount of memory, 8Gb.
  requests:
    cpu: '2000m'
    memory: '8192Mi'
  limits:
    memory: '8192Mi'
```

正常にデプロイできていれば、以下のように Pod を確認できます。
```sh
NAME                                        READY   STATUS    RESTARTS   AGE
bindplane-0                                 1/1     Running   0          3m42s
bindplane-prometheus-0                      1/1     Running   0          3m42s
bindplane-transform-agent-f5c6fb575-5cgtr   1/1     Running   0          3m42s
```

### BindPlane Agent のデプロイ
次に BindPlane ディストリビューション OTel Collector である、BindPlane Agent をデプロイしていきます。これは BindPlane OP の UI から行っていきます。UI は 3001 ポートで開いています。全て書くと細かいのでほどよい粒度で書いています。

#### Configurations の作成

まずは「Configurations > New」から BindPlane の Configuration を作成します。今回は Kubernetes に `Gateway` としてデプロイする Configutations を作成します。他にも [Cluster や Node があります](https://observiq.com/docs/advanced-setup/kubernetes-installation/agent/architecture)。

![](/images/2024-advent-calendar-champ/config.png =650x)

テレメトリーの受信（Source）を設定します。今回は OTLP にしました。

![](/images/2024-advent-calendar-champ/sources.png =650x)

テレメトリーの送信（Destinations）を設定します。もちろん Google Cloud です！

![](/images/2024-advent-calendar-champ/destination.png =650x)

作成がうまくいくと、以下のような OTLP -> Google Cloud のオブザーバビリティパイプラインの設定が UI で見えます。かっこいい。

![](/images/2024-advent-calendar-champ/config_ok.png =650x)

#### 作った Configurations を使って Agent を作成

次に、ここで作った Configuration を使って Agent を「Agents > Install Agents」から作成していきます。

![](/images/2024-advent-calendar-champ/install_agent.png =650x)

Configuration が作成されるので Kubernetes にデプロイしていきます。デプロイが成功すると BindPlane OP の UI 上で Agent が検知され可視化されます（OpAMP のコネクションが確立されます）。

![](/images/2024-advent-calendar-champ/installed_agent.png =650x)

トレースを OTLP で吐き出し続けるアプリケーションもデプロイ済みなので、先ほどのオブザーバビリティパイプラインで「TRACES」を選択してあげると、トレース情報がパイプラインを流れていることが確認できます。いい感じです。

![](/images/2024-advent-calendar-champ/flow.png =650x)

## BindPlan Agent をリモート設定し、トレース情報を処理してみる
画像が多くて疲れてきたと思いますが、最後 BindPlane OP の真骨頂であるリモート設定を行って終わろうと思います。

パイプラインのプロセッサーを押下します。

![](/images/2024-advent-calendar-champ/add_processor.png =650x)

今回は簡単のために「Add Fields」プロセッサーを追加して、謎のタグとして「`Advent:Calendar...!!`」を付与する設定を追加します。ちなみに、他にもたくさんの[プロセッサーたち](https://observiq.com/docs/resources/processors)が用意されています。見てるだけで楽しくなります。

![](/images/2024-advent-calendar-champ/add_tag_processor.png =650x)

追加できるとオブザーバビリティパイプライン上のプロセッサーの個数が可視化されます。

![](/images/2024-advent-calendar-champ/added.png =650x)

実際に Cloud Trace に送られたトレース情報を見てみましょう。ちゃんと意図したタグが付与されていることがわかりました 🥂🥂🥂

![](/images/2024-advent-calendar-champ/cloud_trace.png =650x)

## まとめ
今回は BindPlane を使った OTel Collector（BindPlane Agent）のリモート制御を行い、Google Cloud Observability におけるオブザーバビリティパイプラインの効率的な管理についてご紹介しました。もちろん Google Cloud 以外でも活用は可能です。

OTel Collector をはじめとするテレメトリーエージェントは、監視対象が大規模になるほど管理コストが膨大になっていきます。このようなリモート管理のプロダクトを用いることで恩恵を受けることができます。またこのような技術は OpenTelemetry の OpAMP という標準プロトコルが下支えをし実現されていることを最後触れて結びとさせていただきます。

それではみなさん、良いお年を🎍