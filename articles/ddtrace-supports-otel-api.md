---
title: "Datadog トレーサーで OpenTelemetry 仕様のスパンを受け取る方法"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに
Datadog で分散トレースをしたいときは Datadog が提供しているトレーサーを使ってアプリを計装していきます（Python の場合は、[dd-trace-py](https://github.com/DataDog/dd-trace-py)）。表題や下記ドキュメントにあるように Datadog のトレーサーは OpenTelemetry 仕様のスパンを受け取ることができます。この記事ではどういったケースで本仕様が役立ちそうかを考えていきます。
https://docs.datadoghq.com/ja/tracing/trace_collection/custom_instrumentation/otel_instrumentation/

アプリすでに分散トレースを計装したことがあり、OpenTelemetry や Datadog の自動計装をふんわり理解しているという前提知識のもとで書いていきます。

## どういうことか
アプリに分散トレースを施す場合、大きく以下の準備が必要です。
- ① トレースのセットアップ（TracerProvider の作成など）
- ② スパンの生成（計装ライブラリ使用 / カスタムスパン作成）

Python アプリを Datadog のライブラリを用いてトレース計装する場合は以下のコマンドで自動計装（OpenTelemetry でいうノーコード計装）されます。
```sh
$ pip install ddtrace
$ ddtrace-run python app.py
```
`ddtrace-run` は Python のプログラムで、アプリ実行時に ① トレースのセットアップと、② スパンの生成を自動で行います。スパンの生成はサポートしてるライブラリやフレームワークがある場合、そこに動的にアタッチする形（計装ライブラリ使用）になります。
このプログラムでカスタムスパンを作成したい場合は、ddtrace を使って手動でスパンを作成することもできます^[https://docs.datadoghq.com/ja/tracing/trace_collection/custom_instrumentation/python/dd-api/?tab=decorator]。

このカスタムスパンの作成を OpenTelemetry API を使って行っても、`ddtrace-run` で生成されるスパンと接続され Datadog に送信されるという Datadog トレーサーの機能が本記事の趣旨です。

## やってみる
以下のような Python アプリを使います^[OpenTelemetry Docs の Python サンプルアプリから拝借しています https://opentelemetry.io/docs/languages/python/getting-started/#create-and-launch-an-http-server]。
```python:app.py
import time, logging
from random import randint
from flask import Flask, request

from opentelemetry import trace

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
tracer = trace.get_tracer(__name__)

@app.route("/rolldice")
def roll_dice():
    player = request.args.get('player', default=None, type=str)
    result = str(roll())
    do_work()
    if player:
        logger.warning("%s is rolling the dice: %s", player, result)
    else:
        logger.warning("Anonymous player is rolling the dice: %s", result)
    return result

def do_work():
    # OpenTelemetry Custom Span
    with tracer.start_as_current_span("This's otel span !!!!") as span:
        time.sleep(0.3)

def roll():
    return randint(1, 6)

if __name__ == '__main__':
    app.run(host="0.0.0.0", port="8080")
```

この Python アプリは一部 OpenTelemetry のカスタムスパンが作成されています。このアプリを Datadog トレーサーで計装し、OpenTelemetry のカスタムスパンを接続するには以下のように環境変数 `DD_TRACE_OTEL_ENABLED` を `true` にすることで実現できます。
```sh
$ export DD_TRACE_OTEL_ENABLED="true"
$ ddtrace-run python app.py
```

![](/images/ddtrace-supports-otel-api/otel_custom_span.png =900x)

Flask は `dd-trace` により自動計装がされスパンが作成されており、OpenTelemetry API で作成したカスタムスパンも接続して確認することができます。

## 何が嬉しいのか
今回のサンプルではカスタムスパンは一つですが、実際のアプリではより複雑にカスタムスパンを作成している場合も多いと思います。そういったケースで OpenTelemetry API でカスタムスパンを作成しておくことはプロプライエタリの回避という観点で重要です。例えば、Datadog トレーサーではなく OpenTelemetry のノーコード計装で Python アプリを計装したい場合は、計装プログラム（[opentelemetry-instrument](https://opentelemetry.io/docs/zero-code/python/#configuring-the-agent)）を変更するだけで容易に切り替えることが可能です。環境によってオブザーバビリティツールが異なる場合にも有効かもしれません。
```sh
# $ ddtrace-run pythonapp.py
$ opentelemetry-instrument app.py # OpenTelemetry の Auto Instrumentation ツールを使用
```
:::message
切り替える際の `ddtrace-run` と `opentelemetry-instrument` のサポートライブラリの差分などの互換性やパフォーマンスは確認する必要がもちろんあります。
また、今回はノーコード計装の Python を例に挙げましたが、Go などコードにトレースのセットアップを書く必要がある場合は、切り替える際にコードの改修が一部必要であることは補足します。
:::

## まとめ
ニッチではありますが、オブザーバビリティツールとして Datadog を使いながら、カスタムスパンを作成するアプリ計装は OpenTelemetry を使う方法について書きました。