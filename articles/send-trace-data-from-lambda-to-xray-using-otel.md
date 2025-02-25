---
title: "LambdaからOTELを使いX-Rayにトレース情報を送信する方法とその仕組み"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Lambda", "OTEL", "XRay"]
published: false
---

# 概要

Lambda から X-Ray にトレースを送信する場合、Lambda に標準で用意されているトレーシングを有効にすることで簡単にマネコンからトレースが確認できるようになります。

ただこの方法だとトレースを確認できるのは AWS のマネコンや Baselime など一部のツールに限られてしまいます。
OpenTelemetry でトレースを扱い、最後に X-Ray 向けに送信しておく(X-Ray 用の Exporter を使う)ことで、トレース情報を扱うバックエンドが容易に切り替えられるようになると思い、X-Ray ではなく OpenTelemetry を使う場合はどうなるのか試してみました。

# 最短で達成する方法：公式の Lambda レイヤーを利用する

AWS には ADOT(AWS Distro for OpenTelemetry)という OpenTelemetry の[ディストリビューション](https://opentelemetry.io/docs/concepts/distributions/)があり、Lambda 用に [Lambda レイヤー](https://aws-otel.github.io/docs/getting-started/lambda/lambda-js#enable-auto-instrumentation-for-your-lambda-function) が公開されています。

このレイヤーを指定することで、Express, Redis, PostgreSQL など様々なミドルウェアのトレースが自動的に収集され X-Ray に送信されるようになっています。

- 公式のレイヤーを追加して組み込む場合の手順
  - [AWS Distro for OpenTelemetry Lambda Support For JavaScript](https://aws-otel.github.io/docs/getting-started/lambda/lambda-js#add-the-arn-of-the-lambda-layer)

この方法では以下のようなイメージでアプリケーション側に改修を加えることなく OTLP のトレース情報を X-Ray に送信することが出来ます。

![](/images/send-trace-data-from-lambda/adot-lambda-layer-diagram.png)

この方法で問題になるのが、レイヤーのサイズが大きいことによる Lambda の 250MB のサイズ制限です。
公式のレイヤーを追加することによる導入方法は一般的だと思いますが、今回はこの記事では扱いません。
(自分のケースでは、この方法では容量制限を超過してしまいました)

AWS が公式で配布しているレイヤーのサイズを調べる方法が分からないのですが、おそらく以下のような形で 100MB 以上(160MB?)を占めると思われます。(※1)

- ADOT Collector for Lambda: 約 40MB
  - Golang 実装のバイナリ、Lambda Extension として動作する
  - ここで測っている時点では X-Ray 向けの Exporter が含まれていないので、実際には更に大きくなっていると思われます。
- Receiver/Processor/Propagator 等の組み合わせ: 124MB
  - Node.js アプリケーションの起動前にスパンを開始、必要な情報を上記の Collector に送信するためのコード
  - Node.js 実装。Express, GraphQL, Redis, MySQL, PostgreSQL, MongoDB など様々なツールの計装を自動的に開始する

※1： [aws-observability/aws-otel-lambda](https://github.com/aws-observability/aws-otel-lambda)を clone して Node.js 用にビルド、レイヤーに関わる部分のサイズを確認していますが、正確には把握できていません。

# 別の方法：OpenTelemetry のコンポーネントを自分で組み立てる

公式の Lambda レイヤーは色々なモジュール/ライブラリを自動計装出来るようにするためファットになっていますが、OpenTelemetry の SDK は「何をどのように計装するか」を細かく自分で制御出来るようになっています。

このため、自分に必要なモジュール/ライブラリだけを利用したカスタムのレイヤーを作ることで、サイズを抑えつつ必要なトレース情報を X-Ray に送信することが出来ます。

今回はこちらの方法を試してみました。

# 最終的に用意するもの

2025 年 2 月時点では、以下のような組み合わせで動作することを確認しました。
公式のレイヤーを利用しない場合でも、以下のように 2 つの要素を用意する必要があります(理由は後述します)。

- (1). OpenTelemetry SDK を初期化しトレースを開始するスクリプト(Node.js 実装)
  - この時点では X-Ray には送信せず、HTTP で(2)の Collector にトレース情報を送ります。
- (2). トレース情報を受け取り AWS X-Ray SDK で X-Ray に送信する Lambda Extension(Go 実装。実体は OpenTelemetry Collector)

![](/images/send-trace-data-from-lambda/custom-layer-image.png)

## (1). OpenTelemetry SDK を初期化しトレースを開始するスクリプト

ハンドラー関数の開始前に OpenTelemetry SDK を初期化し、トレース情報を収集、後続のカスタムの Collector に OTLP でトレースを送信するためのスクリプトです。

`NODE_OPTIONS=--import=/opt/nodejs/otel-setup.mjs` を指定しておくことでプリロードされるようにしておきます。

手順は ADOT のサイトの以下のドキュメントを参考にしたものがベースになっています。

- [Tracing with the AWS Distro for OpenTelemetry JavaScript SDK and X-Ray](https://aws-otel.github.io/docs/getting-started/js-sdk/trace-manual-instr#setting-up-the-global-tracer)

:::details package.json

```json
{
  "name": "common-layer",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "type": "module",
  "dependencies": {
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/id-generator-aws-xray": "^1.2.2",
    "@opentelemetry/instrumentation-aws-lambda": "^0.50.3",
    "@opentelemetry/instrumentation-undici": "^0.10.0",
    "@opentelemetry/propagator-aws-xray-lambda": "^0.53.2",
    "@opentelemetry/resource-detector-aws": "^1.11.0",
    "@opentelemetry/resources": "^1.30.1",
    "@opentelemetry/sdk-node": "^0.57.2",
    "@opentelemetry/sdk-trace-base": "^1.30.1",
    "@opentelemetry/semantic-conventions": "^1.30.0",
    "@prisma/instrumentation": "^6.4.1"
  }
}
```

:::

otel-setup.mjs

```js
// ESMのプリロードに必要
// https://github.com/open-telemetry/opentelemetry-js/issues/4933
import { register } from "module";
import { createAddHookMessageChannel } from "import-in-the-middle";

import opentelemetry from "@opentelemetry/sdk-node";
import { diag, DiagConsoleLogger, DiagLogLevel } from "@opentelemetry/api";

import { AWSXRayLambdaPropagator } from "@opentelemetry/propagator-aws-xray-lambda";
import {
  // BatchSpanProcessor,
  SimpleSpanProcessor,
} from "@opentelemetry/sdk-trace-base";
import { Resource } from "@opentelemetry/resources";
import { ATTR_SERVICE_NAME } from "@opentelemetry/semantic-conventions";
import { UndiciInstrumentation } from "@opentelemetry/instrumentation-undici";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { AwsLambdaInstrumentation } from "@opentelemetry/instrumentation-aws-lambda";
import prismaInstrumentation from "@prisma/instrumentation";

const { registerOptions, waitForAllMessagesAcknowledged } =
  createAddHookMessageChannel();
register("import-in-the-middle/hook.mjs", import.meta.url, registerOptions);

diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.DEBUG);

const _resource = Resource.default().merge(
  new Resource({
    [ATTR_SERVICE_NAME]: "sample-otel-app",
  })
);

const _traceExporter = new OTLPTraceExporter();

const sdk = new opentelemetry.NodeSDK({
  textMapPropagator: new AWSXRayLambdaPropagator(),
  instrumentations: [
    // ここの部分で必要なモジュールのInstrumentationを指定する
    new AwsLambdaInstrumentation(),
    // Node.jsのfetchを計装する場合このInstrumentationを追加
    new UndiciInstrumentation(),
    new prismaInstrumentation.PrismaInstrumentation(),
  ],
  resource: _resource,
  traceExporter: _traceExporter,
  spanProcessors: [new SimpleSpanProcessor(_traceExporter)],
});

sdk.start();

process.on("SIGTERM", () => {
  sdk
    .shutdown()
    .then(() => console.log("Tracing and Metrics terminated"))
    .catch((error) =>
      console.log("Error terminating tracing and metrics", error)
    )
    .finally(() => process.exit(0));
});

await waitForAllMessagesAcknowledged();
```

今回はプリロード時に実行させるスクリプトとして ESM のコードを利用しています。

ESM を扱う上での注意事項は以下に記述されていました。

また、このようなセットアップがアプリケーションコードの開始前に必要なことも明記されています。

- [ECMAScript Modules vs. CommonJS](https://github.com/open-telemetry/opentelemetry-js/blob/966ac176af249d86de6cb10feac2306062846768/doc/esm-support.md)
  - [計装のセットアップはアプリケーションコードの開始前に行う必要がある](https://github.com/open-telemetry/opentelemetry-js/blob/966ac176af249d86de6cb10feac2306062846768/doc/esm-support.md#initializing-the-sdk)
- [use module.register(...) in recommended bootstrap code for ESM](https://github.com/open-telemetry/opentelemetry-js/issues/4933)
  - Node.js v18.19, 20.6、22 以降では ESM のプリロード方法として--experimental-loader ではなく--import と register()が推奨される件が記述されています。

## (2). カスタムの Collector

Node.js から OTLP のトレース情報を受け取り、AWS X-Ray SDK を通して X-Ray に送信する役割を持っています。

この部分を一から実装することは難しいことと、Lambda 用にこのような処理を行う OpenTelemetry Collector が以下で公開されており、以下をベースとしています。

- [OpenTelemetry Collector AWS Lambda Extension layer](https://github.com/open-telemetry/opentelemetry-lambda/tree/6a8b279f6f16d3747f2ca509ab53447f763801f1/collector)
  - 2025 年 2 月時点では、この Collector の実装には今回の目的である「AWS X-Ray 向けにトレースをエクスポートする」実装が含まれていません。
  - `make package` や `make publish-layer` でレイヤーのビルドや公開が実行出来るようになっています。
- [AWS managed OpenTelemetry Lambda Layers /adot/collector/lambdacomponents](https://github.com/aws-observability/aws-otel-lambda/tree/main/adot/collector/lambdacomponents)
  - こちらは AWS Lambda Extension layer と似た実装になっていて awsxrayexporter が含まれていますが、独立してビルドすることが出来ません。

今回は上記の "OpenTelemetry Collector AWS Lambda Extension layer"をフォークする形で X-Ray へのエクスポートに対応したバージョンを用意しました。
この方法でビルドした Collector は 45MB 程のサイズになっています。

TODO: fork したリポジトリの URL を掲載

::: message
(1)のコード内で 直接 X-Ray に送信する Exporter を使えば(2)は不要になると思ったのですが、そのような形を取っていない理由について明記されたドキュメントを見つけることが出来ませんでした。

- ネイティブなバイナリで実装されていた方がパフォーマンスが良い
- 再送などの処理をアプリケーションとは別のプロセスで行うことでアプリケーション側の負荷が軽くなり、実装もシンプルになる

:::

## デプロイ時に行うこと

前述の 2 つのレイヤーは事前に何らかの方法でデプロイしているとして、Lambda 関数を Serverless Framework でデプロイする場合は以下のような指定を行います。

serverless.ts

```ts
// serverless.ts
const serverlessConfiguration: Serverless = {
  provider: {
    environment: {
      NODE_OPTIONS: "--import=/opt/nodejs/otel-setup.mjs",
    },
    layers: [
      "arn:aws:lambda:${env:AWS_REGION}:${env:AWS_ACCOUNT}:layer:otel-collector:1",
      "arn:aws:lambda:${env:AWS_REGION}:${env:AWS_ACCOUNT}:layer:common:1",
    ],
    // CollectorがAWS X-Ray SDKを使用してX-Rayに送信するので、IAM 権限が必要
    iamRoleStatements: [
      {
        Effect: "Allow",
        Action: ["xray:PutTraceSegments", "xray:PutTelemetryRecords"],
        Resource: "*",
      },
    ],
  },
};

module.exports = serverlessConfiguration;
```

::: message
Go の LambdaExtension と Node.js でレイヤーを 2 つに分けていますが、公式レイヤーのようにこの 2 つを 1 つにまとめることも出来ると思います。
:::

::: message
改めて記事にまとめたいこと

- OTEL の登場人物はこんな感じ
- Lambda 関数の目線で見ると、まず SDK を初期化してスパンを開始するところから始まる
- 作成したスパンは Processor, Exporter に渡され送信される
  - Node.js 用の Processor としては BatchProcessor などがある
- この時点のスパンを見ると、parentSpanId が設定されていない
  - TraceId が X-Ray 用のフォーマットにもなっていない
- 実はスパンを作成する前に Propagation という仕組みが動いている
  - Propagator には inject, extract というメソッドがあり、トレースのコンテキストに情報を埋め込んだり抽出することが出来る
  - Propagator は AwsLambdaInstrumentation という Lambda を自動計装する仕組みの中で実行されている
    - このため、実際にはこの処理が一番最初に実行されることで X-Ray 経由で発行されたトレース ID、スパン ID と繋がるようになっている
- Lambda 関数が内部で fetch などの関数経由で別の Lambda 関数を呼び出す場合、
  呼び出された側の Lambda 関数はトレース ID を引き継いで繋がらなければならない。
  この部分は UndiciInstrumentation というライブラリが行っている
  - Node.js の fetch は内部では undici というライブラリが使われており、
    :::
