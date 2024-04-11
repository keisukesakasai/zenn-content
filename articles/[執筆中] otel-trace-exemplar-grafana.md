---
title: "テレメトリーを関連付けて Grafana で Metrics to Trace をする"
emoji: "🎪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [OpenTelemetry,Go,Observability,Metrics,Grafana]
published: false
---

## はじめに
先日登壇した [OpenTelemetry Observability運用の実例 Lunch LT](https://findy.connpass.com/event/313260/) のフォローアップブログです！
スライドは以下です。後日アーカイブも公開される？ようです。
https://speakerdeck.com/k6s4i53rx/otel-trace-exemplar

途中 Grafana ダッシュボード上で Metrics to Trace の機能を用いて、「異常なメトリクス値からトレースにジャンプしてボトルネックを素早く特定する」というデモをサラッとしたのですが、本ブログではどう構築できるのかをサクッと紹介します。