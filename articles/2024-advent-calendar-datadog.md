---
title: "GraphQL トレースにカスタム属性を付与して、Datadog でいい感じに解析してみる"
emoji: "🤶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Datadog,Xmas,Observability,GraphQL,Facets]
published: false
---

### はじめに
こんにちは、逆井です。Datadog Advent Calendar 20 日の記事です 👋
本記事では GraphQL サーバーにおけるトレースの解析性の向上のために、スパンに対してカスタム属性の追加と、Datadog UI におけるファセットのカスタマイズを行う内容を紹介していこうと思います。軽めの内容ですので気軽に読んでください👋👋👋

### 背景と方針
GraphQL の場合、トレースのエンドポイントが `/graphql` と単一になり、リソースのカーディナリティが低下してこの断面（API サーバーの処理ごと）における解析性が低下する場合があるかと思います。実際、そういったお悩みを聞いたことがあります。

そこで GraphQL サーバー側で生成されたトレースのルートスパンに対して、カスタム属性を追加することで ↑ の課題にアプローチしていきます。カスタム属性としては、例えばオペレーション名やら、オペレーションクエリパスなどです。イメージとしては以下のような感じかと思います。

![](/images/2024-advent-calendar-dd/graphql.png =900x)

### 実装
ここではサンプルアプリケーションとして、Java 実装の GraphQL サーバーを使っていきます。[@shukawam](https://x.com/shukawam)が教えてくれました。Thanks 🙏
https://github.com/npalm/graphql-java-demo

ここでは例えば Mutation のオペレーションで呼び出される関数で、アクティブスパンの取り出しと、スパンに対するカスタム属性のアタッチを行います。ここでは簡単のため Resolver のなかに直接実装しています。

```java
import io.opentelemetry.api.trace.Span;
...
public Person addPerson(final InputPerson person) {
    // アクティブスパンを取り出す。今回の場合はルートスパン相当
    Span span = Span.current();
    // アクティブスパンに属性を付与する
    span.setAttribute("graphql.operation.name", "mutation");
    span.setAttribute("graphql.mutation.name", "addPerson");
    if (person != null) {
        span.setAttribute("graphql.mutation.input", person.toString());
    }
    return this.personRepository.save(person.convert());
}
```

カスタム計装については Datadog のドキュメントを参考ください。
https://docs.datadoghq.com/ja/tracing/trace_collection/custom_instrumentation/java/otel/#add-custom-span-tags

### 結果
`POST /graphql` のルートスパンに対して、GraphQL のオペレーションに関する属性情報（下図では Mutation の関数名）を付与することができました。オレンジ色の枠で囲んである部分ですね。

![](/images/2024-advent-calendar-dd/result_1.png =900x)

Datadog のトレースエクスプローラ画面では、スパンの属性をカスタムカラムとして追加できますので、GraphQL オペレーションに関するカラムを表示してみましょう。いい感じですね。デフォルトだと `POST /graphQL` が出ていて何が何だかでしたが、オペレーションが追加されることで操作を可視化しターゲットとなるトレースを探しやすくなりましたね。

![](/images/2024-advent-calendar-dd/result_2.png =900x)

あとは、ファセットリストにスパンタグを追加をすればこちらのものです。オペレーションというディメンションでトレースを集計、可視化することができ GraphQL トレースに関しても Datadog を使って解析性を向上させることができました 🥂
https://docs.datadoghq.com/tracing/trace_explorer/span_tags_attributes/

### あとがき
Datadog は自動計装ツールが豊富で、コードに手を加えなくても多くのトレース情報を抽出することができます。今回は GraphQL トレースを題材にカスタム計装を行う例について書きました。今回の手法は GraphQL トレースだけではなくさまざまなケースで有用であることは認識いただけると思います。アプリケーション固有のパラメーターをスパンに付与することで、今回同様さまざまな切り口で解析することができアプリケーションのオブザーバビリティを向上させることが可能です 👋

では皆さん良いお年を。来年どこかの Datadog かコミュニティイベントでお会いしましょう。