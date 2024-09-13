---
title: "Datadog で SQS を介した Lambda の実行を分散トレースする方法"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Datadog,Observability,Lambda,SQS,PHP]
published: false
---

## はじめに
Hi 👋 この記事では SQS を介して Lambda を呼び出す非同期処理の一連の流れを Datadog で可視化する方法を紹介します。

Datadog の Tracer や Lambda Extension Layer については前提知識として解説していきます。

## やりかた
Python の場合 `dd-trace-py` を使うことで自動計装で `Python App -> SQS -> Lambda` をトレースすることができます。これは `dd-trace-py` が SQS を呼び出すクライアントである boto の自動計装をサポートしているためです。

では PHP や Go の場合はどうでしょう。実際 PHP で SQS を呼び出す `aws/aws-sdk-php` は `dd-trace-php` の自動計装対象ではないです。手動計装し トレースコンテキスト を SQS や Lambda 側に渡す必要があります。ではどう渡せばいいのでしょうか？

`dd-trace-py` のコードを見てみます。[この辺](https://github.com/DataDog/dd-trace-py/blob/main/ddtrace/contrib/internal/botocore/services/sqs.py#L55)で SQS に送るメッセージの `MessageAttributes` に、`_datadog` キーとして トレースコンテキスト を埋めています。

Datadog Lambda Extension Layer 側の実装も見てみます。[この関数](https://github.com/DataDog/datadog-lambda-python/blob/main/datadog_lambda/tracing.py#L155) で ↑ のトレースコンテキストを抽出していることがわかります。
:::message
Lambda Extension Layer は様々な Runtime で用意がありますが、この処理が含まれているのは現状 Python と NodeJS です。
https://docs.datadoghq.com/serverless/aws_lambda/distributed_tracing/?tab=go#python-and-nodejs
:::

なるほど。PHP でも SQS に送るメッセージにトレースコンテキストを乗せることで `PHP App -> SQS -> Lambda` の一連の分散トレースができそうです。やってみましょう。

## やってみる
PHP スクリプトのメッセージにトレースコンテキストを付与する部分を抜粋します。ソース全体は[こちら](https://github.com/keisukesakasai/work-2024/blob/main/datadog-php-apm/src/sqs_api_dd.php)を参照してください。
```php
# トレース情報の抽出
$traceContext = $span->getContext();
$traceContext->getTraceId(), $traceContext->getSpanId()
# MessageAttributes の作成
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

これで `dd-trace-py` が内部でやっていることと同様のことを実現できます。SQS を介して Lambda を実行してみます。無事に以下のように非同期処理でも分散トレースすることが確認できました。
![](/images/ddtrace-php-sqs-lambda-trace/php-sqs-lambda.png =900x)

## まとめ
Tracer の自動計装に SQS の呼び出しがサポートされてない場合に、手動でトレースコンテキストを伝搬する方法を紹介しました。

非同期処理でも分散トレースすることで、処理が遅い原因特定の解像度を高めましょう。