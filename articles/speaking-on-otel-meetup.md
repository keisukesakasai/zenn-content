---
title: "OpenTelemetry Meetup 2023-10 に登壇し、計装事例について登壇しました #oteljp"
emoji: "🎤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに

こんにちは 👋 逆井（さかさい）です。
2023年10月19日に開催された [OpenTelemetry Meetup 2023-10](https://opentelemetry.connpass.com/event/296353/) にて、Otel 活用の事例紹介として登壇をさせていただいただきました。その際の感想をつらつらと書いていきます。
※ はてブロや note などのメディアがないのと Zenn が好きなのでこちらにダンプしようと思います。 `tech` type でいいのかな、、、まぁいいか。

まず初めに、イベントを企画・運営してくださった方々や CARTA HOLDINGS 社や、飲み物を提供してくださった Splunk 社には大変感謝です！終始カジュアルでとても良い雰囲気でした。

以下の写真は開始ちょっと前の会場の様子です。Connpass の会場参加は上限の 60 人、オンライン参加申し込みも 420 人（同時接続数はどんなもんだったんだろ？）と `Otel やっていき気運` を感じます。こんなに聴講者がいる登壇は Master の公聴会ぶりだったので緊張しました。

![](/images/speaking-on-otel-meetup/floor.png =550x)

## ジョインしたチームのマイクロサービスたちを再計装した話
発表したスライドと、当日の動画は以下です。
https://speakerdeck.com/k6s4i53rx/getting-started-tracing-instrument-micro-service-with-opentelemetry

https://www.youtube.com/watch?v=Gy7OMeb1NmY

今回は事例にフォーカスして、Otel 分散トレースの取り組みをお話しさせていただきました。
アプリにトレース計装を施すことはそんな大変じゃないです。しかし、入れ替わりの多いチーム特性上、長期的な運用を見据え、チーム全体がテレメトリー計装から計測、性能改善までできる体質形成を行なっていくことも大切というコンテキストでお話ししたつもりです。（後付け感）

課題として、計装が不十分だったため、特定のサービスで分散トレースが途切れてしまっている点を挙げました。一部の計装は前任者が実施していたが、うまく引き継がれていなかったパターンです。SNS などの反応を見るとあるある見たいで、なるほどとなりました。
今回はその解決策の一つとしてトレース計装のガイドラインを作った事例についてご紹介しました。こう言うドキュメントによる手順化は積み重なることで地味にコストが上がるので、開発スピードとトレードオフと捉えられがちです。ただ、今回の再計装などの課題も踏まえて開発規約として取り扱っています。「これは大切やな。」と共感の声も多く挙がっていた気がします。あとは「こう言うルールの整備は 大手 SIer の文化特性もあるかもね。」という発言もあり、確かにそれもあるかもとなりました。

セッションでは時間の都合上触れませんでしたが、過去の計装をそのまま使えたわけじゃなかったと言うことをフォローアップコメントとして残します。結構古いバージョンの OpenteTemetry SDK を使っていた（メジャーバージョンも 0）ので、これを機にそれらの最新化を行いました。一部メソッドなどが破壊的変更されていて若干改修が必要でした。たとえば古の「label」を現行の「attribute」に直す必要などもありました。ソフトウェアは生き物なので塩漬けにせずちゃんとメンテナンスしましょう。と言う当たり前の課題も見えました。

今回は OpenTelemetry "最初の一歩" を踏み出した事例として紹介しました。意外な会社もモダンな技術を取り入れているんだぞ！も裏テーマだったので、参画しているプロジェクトについてもいろいろ触れることができてよかったです。

ぜひこのセッションを通して、OpenTelemetry 活用の後押しが少しでもできたら幸いです。

## ymotongpoo さんと nabeo さんのセッション
当日は自分の番もありあまり集中して聞けなかったのですが、アーカイブで再度拝見しました。いずれのセッションとても勉強にな理ました。

[@ymotongpoo](https://twitter.com/ymotongpoo) さんの [OpenTelemetryのここ4年の流れ](https://speakerdeck.com/ymotongpoo/opentelemetry-in-last-4-plus-years) は、OpenTelemetry プロジェクト発足やテレメトリー仕様がどう定義されていきたかの歴史的背景を追体験できてとても面白かったです。2022 年にオンライン開催された Observability Conference 2022 であった [OpenTelemetryのこれまでとこれから](https://cloudnativedays.jp/o11y2022/talks/1347) のセッションの要約・更新版の位置づけですが、こちらも OpenTelemetry の内部コンポーネントなどを詳に解説してくださってるのでオススメです。わたしも Otel 振り返りたいときに何度も見てたので、今回オフラインで聞けて個人的に胸熱でした。
今後としては、コレクターの stable やプロファイルの仕様策定、Go でのログが注目されます。
https://speakerdeck.com/ymotongpoo/opentelemetry-in-last-4-plus-years

[@nabeo](https://twitter.com/nabeo) さんの [ヘンリーにおける可観測性獲得への取り組み](https://speakerdeck.com/nabeo/henriniokeruke-guan-ce-xing-huo-de-henoqu-rizu-mi)は、Otel Collector 導入に向けた諸々の検討、コスト面からアーキテクチャまでを詳細に解説していただきとても参考になりました。また、冒頭説明いただいた複雑な業務要件よりシステムも複雑化し、オブザーバビリティが重要という点も説得力があり、自分としても共感が大きかったです。今回 PoC を経て、これから実践投入とのことだったので、今後事例も多く共有してくださる(?!)ことが楽しみです。Otel Collector の現場ノウハウはまだあまり流通してないと思うので、とても勉強になりました。我々もやっていきたいというモチベーションになりました。
https://speakerdeck.com/nabeo/henriniokeruke-guan-ce-xing-huo-de-henoqu-rizu-mi

## 最後に
Cloud Native Days Fukuoka 2023 で otel meetup したいと思っているんだよね的な話を聞いて、第一回目で登壇機会いただけるとは思ってなかったので、企画の katzchang さん、ymotongpoo さんには大変感謝です。

懇親会でも多くの Otel 強者の方とお話しでき、実践エピソードも教えていただいたので、次回の otel meetup 盛り上がり気運を感じています。楽しみです！

あとは今後のじぶんにですが、パネルディスカッションちゃんと頑張りましょう。以上です。

## おまけ：感想など
事例紹介ネタでの登壇初めてだったので、直前まで「この知見は共有の価値あるな」と「これ聞いて何になるねん？」の情緒を行き来してました。X でのポジティブな反応は励みになりました。

@[tweet](https://twitter.com/ymotongpoo/status/1714959322961297519?s=20)

@[tweet](https://twitter.com/sadnessOjisan/status/1714959495313654257?s=20)

@[tweet](https://twitter.com/taka2noda/status/1714955366868582899?s=20)

@[tweet](https://twitter.com/paper2parasol/status/1714959280808587594?s=20)

@[tweet](https://twitter.com/mochizuki875/status/1714959530852032620?s=20)