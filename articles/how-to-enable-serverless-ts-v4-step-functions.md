---
title: "Serverless Framework v4 のserverless.tsでStepFunctionsを書く時に型で保護する方法"
emoji: "📐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "Serverless Framework", "Step Functions"]
published: true
---

# 概要

Serverless Framework v4 では serverless.ts というファイル名で TypeScript で設定ファイルを記述することができます。
この型定義は `@types/serverless` というパッケージで管理されていますが、StepFunctions の設定に関する型定義が含まれていません。

Step Functions に関する型定義は `@types/serverless-step-functions` として別に切り出されていますが、以下を見ての通り
これをインストールするだけでは Serverless 型の型定義は拡張されません。

- [types/serverless-step-functions/index.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/serverless-step-functions/index.d.ts)

今回は serverless.ts 内で stepFunctions の設定が補完が効くようにするために行った手順についてまとめます。

# 結論

- `@types/serverless` を devDependencies としてインストールする(通常、これは実施されると思います)
- `@types/serverless-step-functions` を devDependencies としてインストールする
- `serverless.d.ts` などのファイル名で型定義ファイルを作成し、以下の内容を記述する

  ```ts
  import StepFunctions from "serverless-step-functions";

  declare module "serverless/aws" {
    interface Serverless {
      stepFunctions?: StepFunctions;
    }
  }
  ```

- serverless.ts 内で stepFunctions プロパティの補完が効くことを確認する

# この手順が必要になる背景

ServerlessFramework v4 では、serverless.ts 内で以下のように型定義を取り込むように変わりました。

```ts
import { Serverless } from "serverless/aws";
// 以前は import type { AWS } from '@serverless/typescript'
```

ここで import している `Serverless` 型には stepFunctions のプロパティは含まれていないですが、
stepFunctions プロパティに相当する型定義が `@types/serverless-step-functions` として公開されている状態です。

このため、以下のようにして Serverless 型に stepFunctions というプロパティを新しくマージする必要があります。

```ts
import StepFunctions from "serverless-step-functions";

declare module "serverless/aws" {
  interface Serverless {
    stepFunctions?: StepFunctions;
  }
}
```

こうすることで serverless.ts の中で stepFunctions プロパティ内の補完が効くようになります。

:::details 実際の serverless.ts の例

```ts
import { Serverless } from "serverless/aws";

// stages, buildオプションは serverless/aws パッケージに型が定義されていないため仮にobjectとしています。
type ServerlessV4Partial = {
  stages: object;
  build: object;
};

const serverlessConfiguration: Serverless & ServerlessV4Partial = {
  service: "xxxxxxxxx",
  frameworkVersion: "4",

  plugins: ["serverless-step-functions"],

  provider: {
    name: "aws",
    runtime: "nodejs20.x",
    region: "ap-northeast-1",
  },

  build: {
    esbuild: {
      bundle: true,
      external: ["@aws-sdk/*", "aws-lambda"],
      sourcemap: {
        type: "linked",
        setNodeOptions: true,
      },
    },
  },

  functions: {
    listFilesFunc: {
      handler: "functions/list-files.handler",
    },
    countLinesFunc: {
      handler: "functions/count-lines.handler",
    },
  },
  stepFunctions: {
    stateMachines: {
      sampleStateMachine: {
        name: "my-state-machine",
        definition: {
          Comment: "sample step function",
          QueryLanguage: "JSONata",
          StartAt: "ListFiles",
          States: {
            ListFiles: {
              Type: "Task",
              Resource: { "Fn::GetAtt": ["listFilesFunc", "Arn"] },
              Next: "ProcessFile",
              Assign: { files: "{% $parse($states.result.body).files %}" },
            },
            ProcessFile: {
              Type: "Map",
              Items: "{% $files %}",
              MaxConcurrency: 3,
              ItemProcessor: {
                StartAt: "CountLines",
                States: {
                  CountLines: {
                    Type: "Task",
                    Resource: { "Fn::GetAtt": ["countLinesFunc", "Arn"] },
                    Next: "Wait",
                  },
                  Wait: {
                    Type: "Wait",
                    Seconds: 1,
                    End: true,
                  },
                },
              },
              End: true,
            },
          },
        },
      },
    },
  },
};

module.exports = serverlessConfiguration;
```

:::
