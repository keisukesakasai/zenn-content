---
title: "Amazon SQS を介した AWS Lambda の呼び出しを Datadog で分散トレースする方法"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Datadog,Observability,Lambda,SQS,PHP]
published: false
---

## はじめに
Hi 👋 この記事では SQS を介して Lambda を呼び出す非同期処理の一連の流れを Datadog で可視化する方法を紹介します。

Datadog の提供している Tracer や Lambda Extension Layer については前提知識として解説していきます。

## やりかた
Python アプリから呼び出す場合は `dd-trace-py` を使うことで自動計装で `Python アプリ -> SQS -> Lambda` をトレースすることができます。これは `dd-trace-py` が SQS を呼び出すクライアントライブラリである `botocore` の[自動計装をサポートしている](https://docs.datadoghq.com/ja/tracing/trace_collection/compatibility/python/#%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%AE%E4%BA%92%E6%8F%9B%E6%80%A7)ためです。

では PHP アプリや Go アプリの場合はどうでしょう。実際 PHP で SQS を呼び出す `aws/aws-sdk-php` は `dd-trace-php` の[自動計装対象ではない](https://docs.datadoghq.com/tracing/trace_collection/compatibility/php/#library-compatibility)です。手動計装しトレースコンテキストを SQS や Lambda 側に渡す必要があります。どう渡せば良いでしょうか？

`dd-trace-py` のコードを見てみます。[この辺](https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/contrib/internal/botocore/services/sqs.py#L55)で SQS に送るメッセージの `MessageAttributes` に、`_datadog` キーとしてトレースコンテキストを埋めていることが分かります。
https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/contrib/internal/botocore/services/sqs.py#L55

Datadog Lambda Extension Layer 側の実装も見てみます。[ここ](https://github.com/DataDog/datadog-lambda-python/blob/main/datadog_lambda/tracing.py#L155) でトレースコンテキストを抽出していることがわかります。

:::message
Datadog Lambda Extension Layer は様々な Runtime で用意がありますが、この処理（`_datadog` キーからトレースコンテキストを抽出する処理）がサポートされているのは現状 Python と NodeJS です。
https://docs.datadoghq.com/serverless/aws_lambda/distributed_tracing/?tab=go#python-and-nodejs
:::

こんな感じで、PHP でも SQS に送るメッセージにトレースコンテキストを乗せることで `PHP アプリ -> SQS -> Lambda` の一連の分散トレースができそうです。やってみましょう。

## やってみる
PHP スクリプトのメッセージにトレースコンテキストを付与する部分を抜粋します。ソース全体は[こちら](https://github.com/keisukesakasai/work-2024/blob/main/datadog-php-apm/src/sqs_api_dd.php)を参照してください。
```php
# トレースコンテキストを付与した MessageAttributes の作成
$messageAttributes = [
    '_datadog' => [
        'DataType' => 'String',
        'StringValue' => json_encode(\DDTrace\generate_distributed_tracing_headers()),
    ],
];
# SQS への送信
$sqsClient->sendMessage([
    'QueueUrl' => $queueUrl,
    'MessageBody' => $messageBody,
    'MessageAttributes' => $messageAttributes,
]);
```

構成としてはこんな感じです。
![](/images/ddtrace-php-sqs-lambda-trace/architecture.png =900x)

これで `dd-trace-py` が内部でやっていることと同様のことを実現できます。PHP アプリから SQS を介して Lambda を実行してみます。
無事に以下のように非同期処理でも分散トレースすることが確認できました。
![](/images/ddtrace-php-sqs-lambda-trace/php-sqs-lambda.png =900x)

補足ですが、Lambda の関数処理側で自分で Extract 処理を書くことでトレースコンテキストの伝搬をすることも可能です。Runtime によってはこちらの方法で分散トレースをすることになると思うので補足です。以下のドキュメントが参考になります。
https://docs.datadoghq.com/serverless/aws_lambda/distributed_tracing/?tab=python#sample-extractors

## まとめ
自動計装に SQS の呼び出しがサポートされてない場合に、手動でトレースコンテキストを伝搬し `アプリ -> SQS -> Lambda` を分散トレースする方法を紹介しました。今回は PHP で行いましたが、もちろん同様のことを Go のアプリで実現することも可能です。

非同期処理でも分散トレースすることで、処理が遅い原因特定の解像度を高めましょう。

Kafka を使った非同期処理も、こっちは OpenTelemetry ですが分散トレースする方法を書いてるので、気になる方は見てみてください！
https://zenn.dev/k6s4i53rx/articles/2fa37a293cf228