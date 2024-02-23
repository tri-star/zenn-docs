---
title: "NativeESMのパッケージ(middy v5+)を含むプロジェクトでjest.mockが利用できない問題の対処"
emoji: "🔬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "jest", "NativeESM"]
published: false
---

## 起きた問題

TypeScript(ts-jest)、middy v5.2.4 を利用したプロジェクトで、middy のための Jest 用設定を行いテストを書いていたところ、`jest.mock()`によるモックが利用できなくなりました。

jest.config.ts の `setupFilesAfterEnv` を通して以下を実行し、`sampleFunc()`の戻り値をモックしようとしていますが、モックが出来ない状態です。
(middy とは関係のない部分で起きた事象だったため最初なにが原因か分かりませんでした)

```ts
import { jest } from "@jest/globals";

// 以下のようにモックを定義しているが、テストを実行するとモックされていない
jest.mock("@/libs/sample-func", () => {
  return {
    sampleFunc: () => "mocked",
  };
});
```

## 先に結論

- Jest + TypeScript(ts-jest) + Native ESM のパッケージ(今回は`@middy v5.2.4`)を組み合わせる際に指定する一部のオプションが影響している
  - jest を実行する際 `NODE_OPTIONS=--experimental-vm-modules` を付けると `jest.mock()` が機能しなくなる
- 現時点(2024-02-24)では `jest.mock()` の代わりに `jest.unstable_mockModule()` を利用する
  - [実際に動作確認した例](https://github.com/tri-star/jest-esm-support)

## 詳細

上記はドキュメントを読むと明記されているのですが、自分の ESM についての理解が不十分で話がつながらず、だいぶハマってしまいました。

順を追うと以下の通りです。

- middy は v5 以降、Native ESM 形式のパッケージに切り替わったようです。
- Native ESM のパッケージではロード方法が CommonJS モジュールとは変わるため、利用する側が TopLevelAwait に対応する必要があり、利用する側の tsconfig.json に影響が発生します。
  - [実践 Node.js Native ESM — Wantedly でのアプリケーション移行事例 / なぜ Native ESM に対応する必要があるのか](https://www.wantedly.com/companies/wantedly/post_articles/410531#:~:text=%E3%81%8C%E3%81%82%E3%82%8A%E3%81%BE%E3%81%99%E3%80%82-,%E3%81%AA%E3%81%9CNative%20ESM%E3%81%AB%E5%AF%BE%E5%BF%9C%E3%81%99%E3%82%8B%E5%BF%85%E8%A6%81%E3%81%8C%E3%81%82%E3%82%8B%E3%81%AE%E3%81%8B,-%E3%81%AA%E3%81%9CNative%20ESM)
- Jest+TypeScript(ts-jest)の組み合わせで Native ESM 形式のパッケージをテストする場合、専用の設定が必要なことが幾つかのドキュメントに記載されています。
  - [middy / jest and typescript](https://middy.js.org/docs/intro/testing/#jest-and-typescript)
  - [ts-jest / ESM Support](https://kulshekhar.github.io/ts-jest/docs/guides/esm-support/)
  - [jest / ECMAScript Modules](https://jestjs.io/docs/ecmascript-modules)
- この中に「Jest を実行する際、`NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules" npx jest`」のように Jest を実行する指定が含まれています。

  (experimental-vm-modules の指定が必要な理由までは自分では理解できていません。)

  - このフラグを付けて実行すると、 `jest.mock()` が機能しなくなることを確認しました。ただしこのフラグを外すと、`@middy/core` のインポートに失敗してしまいます。

- 自分が設定した例は以下の通りです。

  package.json

  ```diff
    "scripts": {
  -   "test": "jest"
  +   "test": "NODE_OPTIONS=--experimental-vm-modules jest"
    },
  ```

  tsconfig.json

  ```diff
  {
      "compilerOptions": {
  +      "target": "ES2020",
  +      "lib": ["ESNext"],
  +      "module": "ESNext",
      }
  }
  ```

  jest.config.ts

  ```diff
    export default {
  +   preset: 'ts-jest/presets/default-esm',
      setupFilesAfterEnv: ['<rootDir>/src/libs/jest/setup-mock.ts'],
  +   transform: {
  +       '^.+\\.ts?$': [
  +         'ts-jest',
  +         {
  +           useESM: true,
  +         },
  +       ],
  +   },
  +   transformIgnorePatterns: [`node_modules/(?!${esModules})`],
    }

  ```

- experimental-vm-modules と jest.mock についてググっていると以下の記事を見つけ、参考にさせてもらいました。
- [TypeScript + Jest で native ESM](https://zenn.dev/hankei6km/articles/native-esm-with-typescript-jest#native-esm-%E3%81%A7%E3%81%AE-jest-%E3%83%A2%E3%83%83%E3%82%AF%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB)
- `jest.mock()` の代わりに `jest.unstable_MockModule()` を利用することでモックが適用された状態でテスト出来ることを確認できました。

  src/libs/jest/setup-mock.ts

  ```ts
  import { jest } from "@jest/globals";

  jest.unstable_mockModule("@/libs/sample-func", () => {
    return {
      sampleFunc: () => "mocked",
    };
  });
  ```

## 備考

動作確認した結果は以下に残しています。
(今回の事象の確認のためだけに作成しているため、至る所で簡易な実装になっています)
https://github.com/tri-star/jest-esm-support

今回はプロジェクト全体で共通してモックするため jest.config.ts の `setupFilesAfterEnv` を利用してモックしたため、各テストの開始時にはモックが済んでいる状態でした。

個別のテストファイルの中でモックする場合、`jest.unstable_MockModule()` では `jest.mock()` のような巻き上げが行われないため動的 import を利用するように書き方の変更が必要になるようです。

参考： [実践 Node.js Native ESM — Wantedly でのアプリケーション移行事例 / module mock の対応](<https://www.wantedly.com/companies/wantedly/post_articles/410531#:~:text=%3B%20//%20OK-,module%20mock%E3%81%AE%E5%AF%BE%E5%BF%9C,-jest.mock()%20%E3%82%92>)
