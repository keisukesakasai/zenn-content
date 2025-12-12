---
title: "Vertex AI Agent Engine と OpenTelemetry における AI エージェント のトレース入門"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GoogleCloud, Observability, OpenTelemetry, AIAgent, VertexAI]
published: true
---

## はじめに
Hi！Google Developer Experts Advent Calendar 2025 の 12/12 担当のものです。
https://adventar.org/calendars/11658

もともと Google Cloud Champion Innovators コミュニティに所属してたんですが、今年のおめでたいことに 2025/3 に [Google Developer Experts（GDE）コミュニティに統合](https://cloud.google.com/blog/ja/products/ai-machine-learning/the-google-developer-experts-program-is-growing/)されたため、今年は GDE として GDE のアドカレに参加しております。

本ブログ記事では AI エージェントにおけるオブザーバビリティ強化の文脈のはなしを書こうと思います。みなさんご存知 Vertex AI Agent Engine は Google Cloud Observability のスタックといい感じに統合されているため、AI エージェント開発におけるオブザーバビリティ周りが手厚くカバーされています。本記事では特に AI エージェントのトレースについて書いていこうと思います👋

## まずトレースとは
トレースはいろいろな意味を持つ用語ですが、ここではアプリケーションや、AI エージェント内部の処理におけるトレースとして解説していきます。トレースにより、「ユーザーや他のアプリからのリクエストをアプリが処理するのにかかる時間や、リクエストの処理に実行されるオペレーションが完了するのにかかる時間」を知ることができます（[ref: Cloud Trace の概要](https://docs.cloud.google.com/trace/docs/overview?hl=ja)）。アプリケーションを開発、運用している人は馴染み深いテレメトリーシグナルの一つでしょう！

![](/images/2025-advent-calendar-gde/tracing.png =650x)
*[ref: OpenTelemetry Distributed traces](https://opentelemetry.io/docs/concepts/observability-primer/#distributed-traces)*

今回は題材として AI エージェントになります。
AI エージェントが **内部でどのような流れで処理や推論を行なっているか**、**パフォーマンスのボトルネックがどこか**、**エラーはどこで出ているのか**を把握する場合も、もちろんトレースが大いに役立ちます。トレースはいつだって我々の心強い武器なのです。そして、Google Cloud のオブザーバビリティスタックの場合には、Cloud Trace というバックエンドがあり、このサービスを使ってトレースの収集や可視化を行っていきます。

では、どう AI エージェントのトレースが Cloud Trace で可視化されるかを詳しく見ていきます。

## ADK を使って Vertex AI Agent Engine にデプロイする
[Agent Development Kit（ADK）](https://google.github.io/adk-docs/) を作った AI エージェントを [Vertex AI Agent Engine](https://docs.cloud.google.com/agent-builder/agent-engine/overview?hl=ja) にデプロイすることで、開発や運用において Google Cloud マネージドにさまざまな恩恵を受けることができます。その中のイチ要素として Google Cloud Observability との統合があることはご存じのかたも多いでしょう。ADK には、[OpenTelemetry](https://opentelemetry.io/) のトレース収集するためのライブラリが使われており、ADK を使った AI エージェントでは開発者が意識をすることなくフラグ一つでトレース情報の収集を有効化することができます。OpenTelemetry は素晴らしいオブザーバビリティにおけるツールセットなのですが、ここでは詳しくは語りません。気になるかたは若干マニアックネタも多い気がしますが、わたしも参加してる [OpenTelemetry Advent Calendar](https://qiita.com/advent-calendar/2025/otel) をチェックしてみてください 🎄

若干脱線したので軌道修正しますが、ADK を使った AI エージェント開発時のトレースの有効化はとても簡単です。`AdkApp` オブジェクトを初期化する際に `enable_tracing=True` を引き渡してあげるだけです（[ref: Agent Observability with Cloud Trace](https://google.github.io/adk-docs/observability/cloud-trace/)）
```python
# deploy_agent_engine.py
import vertexai
...

adk_app = reasoning_engines.AdkApp(
    agent=root_agent,
    enable_tracing=True, # トレースの収集を有効化
)
...
```

ADK 側でどういう処理が行われているのかをもう少し深ぼってみましょう。AI エージェントが呼び出されるときには以下のコードが実行され、ここにスパン（トレースを構成する要素）を生成をする処理が書かれています。お、OpenTelemetry のライブラリが使われていますね。
https://github.com/google/adk-python/blob/main/src/google/adk/agents/base_agent.py#L271-L301

エージェントの呼び出しの他にも、LLM の呼び出しやツールの呼び出しなどをフックにスパンが生成されるような実装が、ADK 側に埋め込まれていることを周辺のコードを読んでみるとわかります（[ref: adk/telemetry/tracing.py](https://github.com/google/adk-python/blob/main/src/google/adk/telemetry/tracing.py)）。これも余談ですが、ライブラリ側に OpenTelemetry で計装されていて、開発者が有効化するだけでそのライブラリの側もトレースできるのは [Google Cloud クライアントライブラリ](https://docs.cloud.google.com/apis/docs/cloud-client-libraries?hl=ja) も同じような感じです（昔は OpenCensus で計装されていたんですが、去年とかに OpenTelemetry がデフォルト計装になりました [ブログ](https://zenn.dev/k6s4i53rx/articles/gc-client-library-opentelemetry-instrumented)）。ライブラリ内部の挙動や性能を把握するための方法が提供されているのはとても良いなと感じます。

さてじゃあ、Vertex AI Agent Engine に簡単な AI エージェントをデプロイしてみましょう。

## 検証してみましょう
デモアプリでは、みんな大好き [Agent Garden](https://console.cloud.google.com/vertex-ai/agents/agent-garden?pli=1) から [Marketing Agency](https://github.com/google/adk-samples/tree/main/python/agents/marketing-agency) を利用してみます。マーケティングにまつわる AI エージェントであり、オンラインビジネスの立ち上げを支援してくれるマルチエージェンティックなアプリです。もちろん ADK を使って作られていて、基本的に [README.md](https://github.com/google/adk-samples/blob/main/python/agents/marketing-agency/README.md) にしたがってサクッと Vertex AI Agent Engine にデプロイできるので、ここでは手順とかは省きます。

:::message
ADK 側でのトレース収集の設定に加えて、Vertex AI Agent Engine 側で Cloud Trace への送信も有効化する必要があります。詳細は [こちら](https://docs.cloud.google.com/agent-builder/agent-engine/manage/tracing?hl=ja) を参照してください。
:::

Vertex AI Agent Engine に AI エージェントをデプロイすると、コンソール上からプロンプトを送信できる「プレイグラウンド」を使って AI エージェントとやり取りができます。

![](/images/2025-advent-calendar-gde/playground.png =750x)
*Marketing Agencyとのやり取り*

このやり取りで収集されたトレースも Vertex AI Agent Engine 上で Cloud Trace と統合されて可視化されます。

![](/images/2025-advent-calendar-gde/agent_trace.png =750x)
*Marketing Agencyとのやり取りで生成されたトレース*

これを見ると `marketing coordinator` という親エージェントが子エージェントである `domain create` エージェントとをツールとして呼び出すという連鎖的な処理が確認でき、それぞれのプロセスでどの程度時間がかかっているかが可視化されています。下部の属性では、各スパンに付与されたタグ情報も参照することができて、LLM 呼び出しのスパンには、プロンプト入出力などもタグとして付与されています。

このように、**特にトレースのためのセッティングを意識することなく**、ADK や Vertex AI Agent Engine のエコシステムに乗っかり AI エージェントの基本的なオブザーバビリティを確保することができるのは開発、運用において非常に強力ですね。

## まとめ
ADK を使った AI エージェントの Vertex AI Agent Engine におけるオブザーバビリティ、特にトレースのはなしを書きました。機能的にご存じのかたも多いかもしれませんが、内部的に OpenTelemetry が使われていて、どのような仕組みでスパンが生成、収集されているかの若干のマニアックな気づきになれば良いです。

**世は大 AI エージェント時代！** 開発とともに、AI エージェントなオブザーバビリティにも取り組んでいきましょう。明日の担当は [mattn](https://x.com/mattn_jp) さんです！それでは良いお年を。

## そういえば
2025/11 に [Google Developer Group - DevFest Tokyo 2025](https://gdg-devfest-tokyo-2025.web.app/timetable/) という素晴らしいカンファレンスがあったんですが、そこでこのあたりのはなしをデモ付きでしたのでこっちにも接続させておきます。
https://speakerdeck.com/k6s4i53rx/getting-started-vertex-ai-agent-engine-with-opentelemetry