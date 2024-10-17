---
title: "Serverless Framework v4でLambdaの実装にTypeScript+ESMを使う"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "Serverless Framework", "AWS Lambda"]
published: false
---

## 概要

Serverless Framework v4 では TypeScript がネイティブにサポートされ、
`serverless-esbuild` 等のプラグインも含めずに本体が直接 ESBuild 経由でトランスパイルを行うことが出来ます。

v4.4.6 の時点では、serverless.ts をドキュメントに沿って書いているだけでは、
TypeScript + ESM で記述した関数を Lambda にデプロイしても ESM 形式の JS を処理できずエラーになってしまったため、どう対処したかを残しておこうと思います。

## 結論

v4 で ESBuild でトランスパイルを行う設定は以下に記述があります。

[AWS Lambda Build Configuration](https://www.serverless.com/framework/docs/providers/aws/guide/building#configuration)

この設定に追加して、以下も必要でした。
(以下は serverless.ts として記述する場合。zip ファイルに package.json を含めるために必要です)

```ts
  package: {
    patterns: ["package.json"],
  },
```

:::details serverless.ts 全体の例

```ts
// Requiring @types/serverless in your project package.json
import type { Serverless } from "serverless/aws";
// serverless.ts
const serverlessConfiguration: Serverless = {
  org: "xxxxx",
  app: "example-app",
  service: "example-service",
  configValidationMode: "error",
  frameworkVersion: "4",
  build: {
    esbuild: {
      bundle: true,
      minify: false,
      buildConcurrency: 3,
      packages: "external",
      target: "node20",
      platform: "node",
      sourcemap: {
        type: "linked",
        setNodeOptions: true,
      },
    },
  },
  package: {
    patterns: ["package.json"],
  },
  provider: {
    name: "aws",
    runtime: "nodejs20.x",
    region: "ap-northeast-1",
  },
  functions: {
    hello: {
      handler: "src/handler.hello",
      events: [
        {
          httpApi: {
            method: "get",
            path: "/",
          },
        },
      ],
    },
  },
};

module.exports = serverlessConfiguration;
```

:::

以下はこの件の詳細です。

## package.patterns が必要になる理由

Node.js では JavaScript を ESM として実行するにはいくつかの条件があります。

- (A). package.json に `"type": "module"` が指定されている
- (B). または、ファイルの拡張子が `.mjs` である

幾つかの理由から、Serverless Framework v4 で JS を ESM 形式でトランスパイルして Lambda 用の zip ファイルを生成する場合、zip ファイルに `type: module` が記述された package.json を同梱する必要があります。

これには Serverless Framework v4 が esbuild を通して JS をトランスパイルする際の以下の点が関係してきます。

- (1). アプリケーションを ESM で開発すると、esbuild のトランスパイル結果も ESM になる
  アプリケーションの package.json に `"type": "module"` 指定があると、esbuild の format オプションが `esm` に設定されます。
  https://github.com/serverless/serverless/blob/73b3de8a6aeeedf942c1c6fc2314417c82f149b7/lib/plugins/esbuild/index.js#L361-L363
- (2). トランスパイル後の js ファイルの拡張子は js に固定される
  TS コードを esbuild に渡す際の `outfile` オプションが以下の部分で拡張子 js に固定されています。
  https://github.com/serverless/serverless/blob/73b3de8a6aeeedf942c1c6fc2314417c82f149b7/lib/plugins/esbuild/index.js#L590-L594

v4 では `build.esbuild` の設定で esbuild に渡すオプションをほぼ自由に指定できますが、上記の点は上書きすることが出来ません。
(outfile オプションが内部的に設定されるので、outextension で拡張子を js->mjs のように指定することは出来ません)

`package.patterns` で package.json を含めない場合、 `serverless package` コマンドで生成される内容は以下のようになります。

```bash
ls -l .serverless/build

.serverless/build
├── example-app.zip
├── node_modules
├── package.json
├── pnpm-lock.yaml
└── src
    ├── handler.js
    └── handler.js.map
```

package.json も含まれていて一見問題なさそうです。
(この package.json には`"type": "module"` も含まれています)

しかし、この状態で Lambda にデプロイして動作確認すると以下のエラーが発生します。

```json
{
  "errorType": "Runtime.UserCodeSyntaxError",
  "errorMessage": "SyntaxError: Unexpected token 'export'",
  "stack": [
    "Runtime.UserCodeSyntaxError: SyntaxError: Unexpected token 'export'",
    "    at _loadUserApp (file:///var/runtime/index.mjs:1084:17)",
    "    at async UserFunction.js.module.exports.load (file:///var/runtime/index.mjs:1119:21)",
    "    at async start (file:///var/runtime/index.mjs:1282:23)",
    "    at async file:///var/runtime/index.mjs:1288:1"
  ]
}
```

Lambda 関数の本体の JS のコードを確認すると以下のように export を使用しており、ESM であることが分かります。

```js
// src/handler.ts
var hello = async (event, _context) => {
  // ...
};
export { hello };
//# sourceMappingURL=handler.js.map
```

package.json を付けているのに拡張子 js が ESM として扱われず export がエラーになるのはなぜなのかでハマっていましたが、
先ほどの `serverless package` コマンドの生成結果をもう一度調べて、「zip ファイルの中身はどうなっているか」見てみると、次のようになっています。

```
.
├── node_modules
└── src
    ├── handler.js
    └── handler.js.map
```

package.json は zip に含まれていると思っていましたが、含まれていません。
この状態でデプロイされるので、拡張子 js のファイルが CommonJS としてみなされていたと思われます。

serverless.ts に以下の設定を追加してもう一度 `serverless package` を実行すると、

```ts
  package: {
    patterns: ["package.json"],
  },
```

zip ファイルの内容は次のように変わります。

```
.
├── node_modules
├── package.json
└── src
    ├── handler.js
    └── handler.js.map
```

この package.json はアプリケーションの package.json と同じ内容で、 `"type": "module"` が含まれています。
この状態でデプロイすることで index.js が ESM として扱われるため、ESM で記述された JS で Lambda が動作しました。
