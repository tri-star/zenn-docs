---
title: "Laravelでsentry:test実行時に「Please check if your DSN...」が発生する場合の調査方法"
emoji: "🧾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["laravel","sentry"]
published: true
---

# 概要
LaravelにSentryを組み込んだ際、ローカル環境では動作確認できていたのにAWS環境で動作しない問題がありました。
デプロイ先のインスタンスが古いインスタンスだったため通常同じ原因でエラーは起きないと思いますが、
`sentry:test` が出力するエラーが少し分かり難いと感じたため調査方法をメモしておきます。

## 環境
OS: Amazon Linux2 (amzn2-ami-hvm-2.0.20190618-x86_64.xfs.gpt)
Laravel: 5.8.38
sentry/sdk: 3.0.0

# エラーの内容
デプロイ後に動作確認のため `php artisan sentry:test` を実行したのですが、 `Please check if your DSN is set properly in your config ...` というエラーが出てしまい、それ以上の詳細がわかりませんでした。

```
[ec2-user current]$ php artisan sentry:test
[Sentry] DSN discovered!
[Sentry] Generating test Event
[Sentry] Sending test Event
[Sentry] There was an error sending the test event.
[Sentry] Please check if your DSN is set properly in your config or `.env` as `SENTRY_LARAVEL_DSN`
```

# 調査方法
[Error "Please check if your DSN is set properly" when sending a test event on a self-hosted sentry instance](https://github.com/getsentry/sentry-laravel/issues/510#issuecomment-898413281) を参考にしたところ、AppServiceProviderに以下のような記述を追加することで追加のログを出力できるようになるようです。

```php
$this->app->extend(\Sentry\ClientBuilderInterface::class, function (\Sentry\ClientBuilderInterface $builder) {
    $builder->setLogger(new class extends \Psr\Log\AbstractLogger {
        public function log($level, $message, array $context = [])
        {
            error_log("[sentry-laravel] {$level}: {$message} | " . json_encode($context));
        }
    });

    return $builder;
});
```

# 今回の原因と対処方法
上記を追記した状態で再度確認したところ、ログには以下のような情報が出力されていました。

```log
[sentry-laravel] error: Failed to send the event to Sentry. Reason: "SSL certificate problem: certificate has expired for "https://XXXXXXX.ingest.sentry.io/api/XXXXXXX/store/".".
```

OSのルート証明書が期限切れとなっていた点が問題でした。
この場合は対処自体は以下で解消出来ました。

```bash
sudo yum update ca-certificates
```

## 確認結果

```bash
[ec2-user current]$ php artisan sentry:test
[Sentry] DSN discovered!
[Sentry] Generating test Event
[Sentry] Sending test Event
[Sentry] Transaction sent: XXXXXXXXXX
[Sentry] Event sent with ID: XXXXXXXXXXX
```
