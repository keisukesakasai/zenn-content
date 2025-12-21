---
title: "OpenTelemetry SDK のセットアップにおける Declarative Configuration について"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Xmas,Observability,oteltui,AdventCalendar]
published: true
---

こんにちは 👋
OpenTelemetry Advent Calendar 2025 の 12/21 担当の逆井（さかさい）です。
https://qiita.com/advent-calendar/2025/otel

2025 年も残り 10 日となってしまいました。
2025 年は [入門OpenTelemetry](https://www.oreilly.co.jp/books/9784814401024/) や [実践OpenTelemetry](https://www.oreilly.co.jp/books/9784814401031/) が出版されたりと OpenTelemetry 飛躍の年だったと思います。ありがたいことにわたしもレビューに参加させていただきました。原著も読んだのですが、このような素晴らしい書籍を母国語で読めることのありがたさ身に染みており、翻訳という大変な作業に取り組まれている訳者のかたがた[^1]には感謝してもしきれないです。この場を借りて感謝申し上げます。

[^1]: [ドキュメント](https://opentelemetry.io/ja/docs/)もすごい勢いで翻訳がされており、翻訳プロジェクトのメンバーのかたがたも本当にありがとうございます。

本記事では、2026 年の OTel のホットトピックになりうる OTel SDK の [Declarative Configuration](https://opentelemetry.io/docs/languages/sdk-configuration/declarative-configuration/)（翻訳がついてないので、一旦このまま）について書いていこうと思います。

Declarative Configuration を OTel 文脈で聞いたことありますか？
まあ、そのままではあるんですが、OTel SDK のセットアップを宣言的に書けるような取り組みが進んでいます。先月あった [Observability Day NA](https://events.linuxfoundation.org/kubecon-cloudnativecon-north-america/co-located-events/observability-day/#thank-you-for-attending) 冒頭の基調講演でも [Austin](https://www.linkedin.com/in/austinlparker/) から最近話題の [OBI](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation) などとともに 2026 年に期待されるアップデートとして紹介されています。
https://colocatedeventsna2025.sched.com/event/28FN6/observability-project-updates
![](/images/2025-advent-calendar-otel/declarative-config.png =750x)

アプリケーションにおける OTel SDK の設定はご存知の通り[多くの設定値](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)を環境変数を経由して行うことができますが、より複雑な設定のニーズが高まることで Declarative Configuration のような設定方式が開発されたと語られております。執筆時点においては Java のみがサポートされています。Declarative Configuration は今年[スキーマのメジャーバージョンが1となった](https://github.com/open-telemetry/opentelemetry-configuration/releases/tag/v1.0.0-rc.1)ので、今後、他の言語でも順次サポートされていくでしょう（言語ごとのステータスは[こちら](https://github.com/open-telemetry/opentelemetry-configuration/blob/main/schema-docs.md#language-support-status-)）。

では、どんな設定になるのか見てみましょう。

Declarative Configuration では以下のようなコンフィグファイルを用意します（敢えて書きますが、OTel Collector のコンフィグファイルとは別物です）。最小構成での例です。

```yml:otel-config
file_format: '1.0-rc.3'

resource:
  attributes:
    - name: service.name
      value: dice-roll-demo
    - name: service.version
      value: 1.0.0
    - name: deployment.environment
      value: production
    - name: advent
      value: calendar

propagator:
  composite:
    - tracecontext:
    - baggage:

tracer_provider:
  processors:
    - batch:
        exporter:
          otlp_http:
            endpoint: http://oteltui:4318/v1/traces
```

このコンフィグファイルを渡す形で、アプリケーションを起動します。Java の場合は以下です。いつもの実行コマンドで引数を追加します。まだ `experimental` がついていますね。

```bash
java -javaagent:/app/opentelemetry-javaagent.jar \
　-Dotel.experimental.config.file=/app/otel-config.yaml \
　-jar app.jar
```

こうすることで、トレース情報に `advent:calendar` という属性付けがされます。上記の例では `resorce.attributes` で属性を付与していました。
現状設定できる全ての設定値は[こちら](https://github.com/open-telemetry/opentelemetry-configuration/blob/main/examples/kitchen-sink.yaml)で見れるようです。
![](/images/2025-advent-calendar-otel/oteltui.png =750x)
*oteltui による可視化（今年も oteltui にはお世話になりました 🙏）*

[こちらのブログ](https://opentelemetry.io/blog/2025/declarative-config/)では、トレースサンプリングでよく議論に上がる「ヘルスチェックエンドポイントのサンプリング設定」を Declarative Configuration により簡単且つ柔軟に設定できるようになったと記述されています（ネタ的に、**Why it took 5 years to ignore health check endpoints in tracing** というタイトルで、環境変数設定でのサンプリング設定の難しさが表されているようです）。試しにヘルスチェックトレースをサンプリングしない設定を Declarative Configuration で見てみましょう。ルールベースのサンプリング設定は先週リリースされた [v1.0.0-rc.3](https://github.com/open-telemetry/opentelemetry-configuration/releases/tag/v1.0.0-rc.3) のバージョンに [この PR](https://github.com/open-telemetry/opentelemetry-configuration/pull/410) で追加されており、以下のようなコンフィギュレーションになります。

```bash
file_format: '1.0-rc.3'

resource:
...
  sampler:
    parent_based:
      local_parent_sampled:
        composite/development:
          rule_based:
            rules:
              - attribute_values:
                  key: http.route
                  values:
                    - /healthz
                    - /livez
                sampler:
                  always_off:
```

これであれば 5 年もかからず簡単にセットアップして、不要なエンドポイントにおけるトレースサンプリングをスキップできそうです。もちろん、アプリケーションでサンプラーの設定を書いてもいいですが、言語によらない形でセットアップを宣言的に記述できる点は便利かもしれません。

以上です！
今回は OTel SDK における Declarative Configuration について記しました。今回のデモで使った Java Agent の Declarative Configuration も [Experimental](https://opentelemetry.io/docs/zero-code/java/agent/declarative-configuration/#:~:text=Declarative%20configuration%20is%20experimental.) なステータスなのでご留意いただきつつ、ぜひ 2026 年の Declarative Configuration の進捗を追っていきましょう！明日のアドベントカレンダー担当は [@melonpass](https://qiita.com/melonpass) さんです！

最後に宣伝です！OTel Meetup が Powered by [TAKA_0411](https://x.com/TAKA_0411) により札幌開催されます！2/19 です！ぜひみなさんご参加していただきわいわいしましょう！
https://opentelemetry.connpass.com/event/376362/