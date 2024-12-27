---
title: "CloudNative Days で OpenTelemetry のはなしをしました"
emoji: "🤶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Xmas,Observability,CloudNativeDays]
published: true
---

こんにちは、OpenTelemetry Advent Calendar 2024 の 12/23 担当のものです！
https://qiita.com/advent-calendar/2024/opentelemetry

年末の風物詩である [CloudNative Days](https://cloudnativedays.jp/) で、OpenTelemetry（以下、OTel）の登壇をさせていただきました。思い返すと CNDT2022、CNDT2023、CNDW2024 とありがたいことに OTel ネタで採択していただいたので、それらのスライドと共に内容を振り返りつつ、関連する最近の個人的な OTel ホットトピックについても触れていこうと思います。

## CNDT2022
オブザーバビリティにおける重要な要素の一つととして「テレメトリーシグナル同士の関連付け」をテーマに、OTel でどのようにログやトレース、メトリクス関連づけることができるかを解説しています。収集したテレメトリーシグナルを Grafana スタックで可視化し、簡単にテレメトリー間をジャンプできるデモを行いました。
https://speakerdeck.com/k6s4i53rx/cndt2022-shi-jian-opentelemetry-to-oss-woshi-tuta-observability-ji-pan-nogou-zhu

デモで使っている Span Metrics Processor は廃止され、現在は [Span Metrics Connector](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/spanmetricsconnector) です。またトレースとメトリクスを関連付けるエグザンプラーに関しては、OTel Go SDK の機能として取得可能になりました。これに関しては ymotongpoo さんの[ブログ記事](https://zenn.dev/google_cloud_jp/articles/20240305-trace-exemplar) が詳しいです。

## CNDT2023
OTel の「自動計装」をテーマに、さまざまな言語でどのようにコードに手を加えない計装を行えるかを解説しました。
https://speakerdeck.com/k6s4i53rx/getting-started-auto-instrumentation-with-opentelemetry

この発表ののちに OTel ではコードに手を加えない計装を「[ゼロコード計装](https://opentelemetry.io/docs/zero-code/)」としています。したがって ↑ のスライドのタイトルはイマは正確には「"ゼロコード計装"のイマ」です。

本発表で少し触れた Go のゼロコード計装について簡単にアップデートを書きます。[opentelemetry-go-instrumentation](https://github.com/open-telemetry/opentelemetry-go-instrumentation) は依然として WIP ステータスです。eBPF を使って実現しており、内部の仕組みはふんわり [こちらのスライド](https://speakerdeck.com/k6s4i53rx/getting-started-opentelemetry-operator-on-kubernetes) で解説をしました。

eBPF ではない Go のゼロコード計装として「Go Compile Time Instrumentation」を仕組みとしたツールもあります。代表的なものとしては、Datadog の「[orchestrion](https://github.com/DataDog/orchestrion)」や、Alibaba Cloud の「[opentelemetry-go-auto-instrumentation](https://github.com/alibaba/opentelemetry-go-auto-instrumentation)」が存在しています。コンパイル時に AST から使われているライブラリを解析して、計装コードを挿入する仕組みのようです。サポートしているライブラリの数も eBPF ツールに比べてかなり多いです。本当は orchestrion のコードを読んで解説ブログをアドカレで出したかったのですが、太刀打ちできなかったので冬休みの課題図書としました。

orchestrion は Datadog ユーザー向けの計装ツール（[2024/11に GA](https://github.com/DataDog/orchestrion/releases/tag/v1.0.0)）だったのですが、現在 OTel プロジェクトに寄贈する提案が [Issue](https://github.com/open-telemetry/community/issues/2497) としてあがっています。Alibaba Cloud の opentelemetry-go-auto-instrumentation についても[同様](https://github.com/open-telemetry/community/issues/2344)です。それに伴い、Go Compile Time Instrumentation SIG の立ち上げも[提案](https://github.com/open-telemetry/community/pull/2490)されています。Go ゼロコード計装の今後の動向が気になりますね。

## CNDW2024
OTel Collector のリモート管理をテーマに「Open Agent Management Protocol（OpAMP）」について解説しました。
https://speakerdeck.com/k6s4i53rx/getting-started-opentelemetry-collector-with-opamp

KubeCon/CloudNativeCon で取り上げられていたり、[O’Reilly Learning OpenTelemetry](https://www.amazon.co.jp/Learning-OpenTelemetry-Setting-Operating-Observability-ebook/dp/B0CXC87NF3) にも複数箇所で記述があったりだが、あんまり情報がない領域だったので思い立ってプロポーザルを出しました。
- [Remote Control for Observability Using the Open Agent Management Protocol](https://kccncna2023.sched.com/event/1R2sr)
- [OpAMP: Scaling OpenTelemetry with Flexibility](https://kccncossaidevchn2024.sched.com/event/1eYZt/opamp-scaling-opentelemetry-with-flexibility-opampdaepopentelemetry-husni-alhamdani-cncf-ambassador-herbert-sianturi-krom-bank-indonesia)

本発表では時間の都合上触れられなかったのですが、OpAMP バックエンド製品の BindPlane については以下ブログに書きました。CNDW2024 のセッションでは OpAMP バックエンドの[サンプルコード（シンプルな UI）](https://github.com/open-telemetry/opamp-go/tree/main/internal/examples/server)から OTel Collector をリモート制御しましたが、BindPlane を使うことで高機能な UI で管理を行うことができます。エージェントは BindPlane の OTel Collector ディストリビューションを使うことになります。自前で OpAMP（例えば `opamp-go` 実装）を使って OpAMP サーバーを構築することも可能ですが、このような製品を使うことも選択肢の一つとしてあるかもしれません。詳しくは以下ブログを参照ください。
https://zenn.dev/k6s4i53rx/articles/2024-advent-calendar-gc-champ

OTel Collector を提供するオブザーバビリティベンダーが OpAMP を使っていくケースもありそうです。Grafana Distro の Grafana Alloy や、AWS Distro の ADOT Collector には OpAMP に関する言及がされています。
- [Support for OpAMP to manage fleet of Alloy agents](https://github.com/grafana/alloy/issues/1236)
- ["The way we want to approach this is via OpAMP, where the supervisor controls the collector and with it hotreloads."](https://github.com/aws-observability/aws-otel-collector/issues/2402)

## 最後に
OTel は見るべき範囲が広大だったり、変化も早かったりでキャッチアップするのが大変ですが、[OTel Meetup](https://opentelemetry.connpass.com/) も定期的に開催され毎度勉強になるセッションが聞けますし、ドキュメントの [日本語化プロジェクト](https://opentelemetry.io/ja/) も進んでるので、コミュニティの力を借りながら今後も継続的にウォッチしていければと思います。

最後に CloudNaive Days を運営してる方々に感謝ですが、毎年本当にありがとうございます。