---
title: "PHPStorm等でUI上原因が分かり難いエラーが出た時の追加の調査方法"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PHPStorm","Docker"]
published: true
---

# 概要
WSL2のVM上でPHPのDockerコンテナを使っている環境で、ある日突然以下のようなエラーが出てPHPのデバッグやUnitテスト実行が出来なくなってしまい、困っていました。

```
Failed to parse validation script output
```

:::details 実際の画面
![](/images/2022-04-25-22-47-22.png)
:::

この件について調べていたところ以下のIssueが見つかり、「PHPStormのログを調べる」、「DEBUGログを出力させる」方法が分かったためメモしておきます。
[Failed to parse validation script output with docker-compose](https://youtrack.jetbrains.com/issue/WI-46602/Failed-to-parse-validation-script-output-with-docker-compose#focus=Comments-27-3493716.0-0)


# 環境
PHPStorm: 2021.3.3 (Build #PS-213.7172.28, built on March 17, 2022)
PHP: 8.0.15
Docker for Windows: 4.5.1 (74721)
Docker Engine: 20.10.12

# 調査方法

## 1. DEBUGログを有効にする(省略可能)
以下の手順でDEBUGレベルのログを有効にします。
事象によってはDEBUGレベルまで有効にしなくてもログだけで確認できるかもしれません。

- `Help > Diagnostic Tools > Debug Log Settings...` を選択
    ![](/images/2022-04-25-23-24-56.png)
- 表示されるダイアログに `#com.jetbrains.php` を入力
    ![](/images/2022-04-25-23-34-30.png)
- ここで、一度PHPStormを再起動

## 2. 事象を再現させ、ログを確認する
再起動が完了した後、事象を再現させてログの確認を行います。
ログの確認は以下で出来るようです。

- `Help > Show Log in Explorer` を選択
    ![](/images/2022-04-25-23-39-22.png)
- エクスプローラが表示されるため、`idea.log` を開いて末尾部分を確認します。

![](/images/2022-04-26-00-15-04.png)


今回は以下のような形でDEBUGログが残っており、docker-compose.ymlの中でxdebug.iniを直接VOLUMEマウントしていたことが影響していることがわかりました。

docker-compose.ymlの該当部分をコメントアウトして(docker compose down/upはなし)再度試したところ、解消することが確認出来ました。

実際のログ

```
2022-04-25 21:40:39,421 [  59309]   INFO - ockerComposeIntegrationService - Populate shared volume phpstorm_helpers_PS-213.7172.28 
2022-04-25 21:40:49,919 [  69807]  DEBUG - php.config.phpInfo.PhpInfoUtil - Error on executing '/usr/bin/php [-dxdebug.remote_enable=0, -dxdebug.mode=off, /opt/.phpstorm_helpers/phpinfo.php]': Creating xxxxx_app_run ... 
Creating xxxxx_app_run ... done
Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: process_linux.go:545: container init caused: rootfs_linux.go:76: mounting "/wsl$/Ubuntu-20.04/home/xxxxxxxx/projects/xxxxxxx/docker/app/etc/php/8.0/mods-available/xdebug.ini" to rootfs at "/etc/php/8.0/mods-available/xdebug.ini" caused: mount through procfd: not a directory: unknown: Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type
2022-04-25 21:40:49,920 [  69808]  DEBUG - php.config.phpInfo.PhpInfoUtil - Parsing validation output:  
2022-04-25 21:40:49,936 [  69824]   INFO -                         STDERR - [Fatal Error] :-1:-1: Premature end of file. 
2022-04-25 21:40:49,937 [  69825]   WARN - php.config.phpInfo.PhpInfoUtil - Failed to parse validation output:  
2022-04-25 21:40:49,937 [  69825]   WARN - .PhpRemoteInterpreterComponent - Can not update phpinfo 
```

エラーが解消した後

![](/images/2022-04-26-00-55-31.png)
    

## 3. DEBUGログ設定を元に戻す
(1)で実施したDEBUGログ設定を戻し、再起動させます。


# まとめ
IDEではUI上のエラーメッセージからは原因が分かり難いことが時々あるので、IDE自体のログが確認できることやDEBUGログの出力方法も分かると、より詳しく調査出来ると思いました。

今回問題になっていたコンテナ内のPHPを認識出来ない問題は「ファイルをマウントしていた」ことが影響していましたが、ファイルのマウント自体は問題ないと思っていました。

ログを見る限りDockerデーモン側でエラーが起きていて、何となくPHPStorm側がdocker-compose.ymlの定義に従って別のコンテナを立ち上げる際、「ボリューム指定の対象は全てディレクトリ」という前提になっているのかもしれないと思いました。
本当にそうなのかは確認できていません...。
