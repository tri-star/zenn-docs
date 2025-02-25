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

# 最短で達成する方法：公式の Lambda レイヤーを利用する

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

# 別の方法：OpenTelemetry のコンポーネントを自分で組み立てる

公式の Lambda レイヤーは色々なモジュール/ライブラリを自動計装出来るようにするためファットになっていますが、OpenTelemetry の SDK は「何をどのように計装するか」を細かく自分で制御出来るようになっています。

このため、自分に必要なモジュール/ライブラリだけを利用したカスタムのレイヤーを作ることで、サイズを抑えつつ必要なトレース情報を X-Ray に送信することが出来ます。

今回はこちらの方法を試してみました。

# 最終的に用意するもの

2025 年 2 月時点では、以下のような組み合わせで動作することを確認しました。
公式のレイヤーを利用しない場合でも、以下のように 2 つの要素を用意する必要があります(理由は後述します)。

![](/images/send-trace-data-from-lambda/custom-layer-image.png)

## (1). Lambda ハンドラーの実行前に読み込むスクリプト

ハンドラー関数の開始前に OpenTelemetry SDK を初期化し、トレース情報を収集、後続のカスタムの Collector に OTLP でトレースを送信するためのスクリプトです。

`NODE_OPTIONS=--import=otel-setup.mjs` を指定しておくことでプリロードされるようにしておきます。

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

OTLP のトレース情報を AWS X-Ray SDK を通して X-Ray に送信する役割を持っています。

(1)のコード内で BatchSpanProcessor などを使いつつ直接 X-Ray に送信する Exporter を使えば(2)は不要になると思ったのですが、
Lambda のフリーズ/再開など

<!--
OTLP のトレースを X-Ray SDK で送信する exporter として [awsxrayexporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/awsxrayexporter/awsxray.go) がありますが、これが Go で実装されていることと、
Lambda Extension として動作することで Lambda 関数の freeze などにも関与出来るようにするため実装が分かれているようです。
-->
