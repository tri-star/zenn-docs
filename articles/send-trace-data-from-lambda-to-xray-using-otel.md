---
title: "LambdaからOTELを使いX-Rayにトレース情報を送信する方法とその仕組み"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Lambda", "OTEL", "X-Ray"]
published: false
---

# 概要

Lambda から X-Ray にトレースを送信する場合、Lambda に標準で用意されているトレーシングを有効にすることで簡単にマネコンからトレースが確認できるようになります。

ただこの方法だとトレースを確認できるのは AWS のマネコンや Baselime など一部のツールに限られてしまいます。
OpenTelemetry でトレースを扱い、最後に X-Ray 向けに送信しておく(X-Ray 用の Exporter を使う)ことで、トレース情報を扱うバックエンドが容易に切り替えられるようになると思い、X-Ray ではなく OpenTelemetry を使う場合はどうなるのか試してみました。

# 最短で達成する方法

AWS では ADOT(AWS Distro for OpenTelemetry)が OpenTelemetry の[ディストリビューション](https://opentelemetry.io/docs/concepts/distributions/)があり、Lambda 用には [Lambda レイヤー](https://aws-otel.github.io/docs/getting-started/lambda/lambda-js#enable-auto-instrumentation-for-your-lambda-function) が公開されています。

このレイヤーを指定することで、Express, Redis, PostgreSQL など様々なミドルウェアのトレースが自動的に収集され X-Ray に送信されるようになっています。
この方法での導入方法は以下に公式のドキュメントがあります。

- [AWS Distro for OpenTelemetry Lambda Support For JavaScript](https://aws-otel.github.io/docs/getting-started/lambda/lambda-js#add-the-arn-of-the-lambda-layer)

レイヤーを追加することで以下のようなイメージでアプリケーション側に改修を加えることなく OTLP のトレース情報を X-Ray に送信することが出来ます。

![](/images/send-trace-data-from-lambda/adot-lambda-layer-diagram.png)

この方法で問題になるのが、レイヤーのサイズが大きいことによる Lambda の 250MB のサイズ制限です。
レイヤーを追加することによる導入方法も一般的だと思いますが、今回はこの記事では扱いません。
(自分のケースでは、この方法では容量制限を超過してしまいました)

AWS が公式で配布しているレイヤーのサイズを調べる方法が分からないのですが、おそらく以下のような形で 100MB 以上(160MB?)を占めると思われます。(※1)

- ADOT Collector for Lambda: 約 40MB
  - Golang 実装のバイナリ、Lambda Extension として動作する
  - ここで測っている時点では X-Ray 向けの Exporter が含まれていないので、実際には更に大きくなっていると思われます。
- Receiver/Processor/Propagator 等の組み合わせ: 124MB
  - Node.js アプリケーションの起動前にスパンを開始、必要な情報を上記の Collector に送信するためのコード
  - Node.js 実装。Express, GraphQL, Redis, MySQL, PostgreSQL, MongoDB など様々なツールの計装を自動的に開始する

※1： [aws-observability/aws-otel-lambda](https://github.com/aws-observability/aws-otel-lambda)を clone して Node.js 用にビルド、レイヤーに関わる部分のサイズを確認していますが、正確には把握できていません。

# OpenTelemetry のコンポーネントを自分で組み立てる

公式の Lambda レイヤーは色々なものに備えて Fat になっていますが、OpenTelemetry の SDK は「何をどのように計装するか」を細かく自分で制御出来るようになっています。
このため、以下のような組み合わせで独自の Lambda レイヤーを用意すれば容量を節約しながら OTLP を用いた計装が可能です。

# 最終的に必要になるもの
