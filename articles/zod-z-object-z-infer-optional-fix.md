---
title: "zodのz.objectをz.inferした結果がname?:stringのようにoptionalになってしまう"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zod", "typescript"]
published: true
---

# 概要

zod で、以下のように z.object で作成したスキーマから z.infer で型を求めた時、
基本的には全て必須項目になるはずですがなぜか optional になってしまう事象が起きました。

GitHub の Issue などもあったのですが、起きている事象からだと調べにくいかもと思い記事にしようと思います。

```typescript
export const userSchema = z.object({
  name: z.string(),
});

export type User = z.infer<typeof userSchema>;

/*
User型の内容。各プロパティに"?"が付いてoptionalになっている

type User = {
    id?: string;
    name?: string;
    email?: string;
    createdAt?: string;
    updatedAt?: string;
}
*/
```

# 原因

tsconfig.json で `strictNullChecks` が false に設定されていると上記のように各プロパティが optional なプロパティになるようです。

最初から TypeScript でプロジェクトを始める場合は strict: true で始めることが多いので通常問題が起きないと思いますが、
何らかの理由で strict: false にしたり strictNullChecks だけ false にした場合、この形が起きると思います。

ドキュメントを改めて見ると最初の [Requirements](https://github.com/colinhacks/zod#requirements) のところで **You must enable strict mode** と記述されており、大前提のようでした...。

# 関連資料

- Issue: [Type coming through as optional when using z.infer<>](https://github.com/colinhacks/zod/issues/408)
- Discussion: [strictNullChecks=false makes z.object() properties optional](https://github.com/colinhacks/zod/discussions/2160)
- Document: [Requirements](https://github.com/colinhacks/zod#requirements)
