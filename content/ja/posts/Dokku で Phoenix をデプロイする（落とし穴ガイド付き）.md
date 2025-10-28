+++
date = '2025-08-21T22:41:51+09:00'
draft = false
title = 'Dokku で Phoenix をデプロイする（落とし穴ガイド付き）'
categories= ['Programming']
tags= ['Elixir', 'Deploy', 'Phoenix', 'Dokku', 'Postgres', 'Cloudflare']
+++

[Kamal で Web サービスをデプロイする](<./Kamal を使用した Web サービス（PostgreSQL を含む）のデプロイ方法.md>) をしばらく運用してみたものの、最終的には Kamal を手放すことにしました。理由は次の通りです。

1. 公式ドキュメントがあまりにも簡潔で、多くの箇所が一文で済まされており、ユーザーは経験豊富なデプロイ担当者であることが前提になっているように感じました。そのため、問題が発生するとインターネットをひたすら調べたり、ソースコードを読む必要がありました。
2. Phoenix プロジェクトで利用する際、環境変数の流れが複雑すぎます（コードに届くまで 4 段階もあり、中間フローが 2 つ余計に増える）：システム環境変数 --> `.kamal/secrets` --> `configs/deploy.yaml` --> `configs/runtime.exs`

色々と調べて比較した結果、Heroku の OSS 版ともいえる [Dokku](https://dokku.com/) に乗り換えることにしました。エコシステムは Kamal よりはるかに充実しており、ドキュメントも揃っています。

## Dokku の仕組み

---

まずは Dokku の動作イメージを説明します。構造を理解しておくとトラブルシューティングが楽になります。すでに理解していれば、この章は読み飛ばして「デプロイ手順」を確認してください。

1. サーバーにインストールして使うプログラムであり、内部に Git サーバー（ほかのモジュールもあります）が立っています。別の場所にあるコードリポジトリからこの Git サーバーに push すると、Dokku はフック（Git Hook）を受け取り、デプロイ処理を開始します。
2. アプリケーションはサーバー上で Docker コンテナとして動きます。Docker イメージは、コードを push したあと Dokku がサーバー上でビルドします。Dockerfile があればそれを使い、なければ Herokuish（Heroku の Buildpack に相当するスクリプト群）でビルドします。

## デプロイ手順

---

**今回のゴール：** Phoenix アプリケーションを Postgres データベースに接続し、Cloudflare に管理させたドメインで公開する。

以下は公式ドキュメント、コミュニティ記事、そして実際にハマった内容を踏まえて整理した手順です。

0. [公式ドキュメント](https://dokku.com/docs/getting-started/installation/) を参考に、サーバーへ Dokku をインストールし、SSH 鍵とドメインを設定する。
1. サーバーでアプリケーションを作成する（ローカル開発のアプリ名と同じで OK）

```bash
# Dokku をインストールしたサーバー上で実行
dokku apps:create ruby-getting-started
```

2. dokku postgres プラグインをインストールする（サービス名 railsdatabase は任意の名前で構いません）

```bash
# Dokku をインストールしたサーバー上で実行
# postgres プラグインをインストールする
# プラグインのインストールは root 権限が必要なので sudo が付いている
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git0

# railsdatabase という名前で Postgres サービスを作成
dokku postgres:create railsdatabase
```

3. アプリにデータベース接続用の URL 設定を追加する

postgres:link の仕組みは概ね以下の通りです。

```bash
# Dokku をインストールしたサーバー上で実行
# 公式のデータストアには、任意のアプリケーションへサービスをリンクする `link` メソッドが用意されている
dokku postgres:link railsdatabase ruby-getting-started
```

4. Herokuish（buildpacks）を追加する

```bash
echo "https://github.com/HashNuke/heroku-buildpack-elixir.git" >> .buildpacks
# gigalixir のビルドパックを使用すると、https://github.com/gjaldon/heroku-buildpack-phoenix-static/issues/127 の問題が解消される
echo "https://github.com/gigalixir/gigalixir-buildpack-phoenix-static.git" >> .buildpacks
```

4. buildpack を設定する

```bash
# プロジェクトのルートディレクトリで elixir_buildpack.config を作成
touch elixir_buildpack.config
```

以下の内容を書き込みます。

`config_vars_to_export` はプロジェクトの事情に合わせて調整してください。以下は私の設定例です。

```txt
# バージョン名は必ず小数点を含めて、リストに掲載されている表記と完全に一致させる必要があります。以前 27 とだけ書いたら認識されませんでした。
erlang_version=27.0

# Elixir version
elixir_version=1.18.1

# Always rebuild from scratch on every deploy?
always_rebuild=true

# Export heroku config vars
config_vars_to_export=(DATABASE_URL GMAIL_PASSWORD GMAIL_USERNAME SECRET_KEY_BASE)
```

```bash
# プロジェクトのルートディレクトリで phoenix_static_buildpack.config を作成
touch phoenix_static_buildpack.config
```

続いて以下を記述します。

```txt
# Node のバージョンを高めに指定しておかないと、非常に古いバージョンが入って最終的に失敗します
node_version=23.11.0
npm_version=10.9.2
```

5. Dokku が利用する Heroku Stack をダウングレードする  
　先ほど追加した [HashNuke/heroku-buildpack-elixir](https://github.com/HashNuke/heroku-buildpack-elixir?tab=readme-ov-file#version-support) は最新の stack `heroku-24` に対応していません。しかしサーバーへインストールした Dokku はデフォルトで最新の stack `heroku-24` を使うため、デプロイ時にエラーになります。

```bash
=====> Downloading Buildpack: https://github.com/HashNuke/heroku-buildpack-elixir.git
=====> Detected Framework: Elixir
-----> Will export the following config vars:
       CURL_CONNECT_TIMEOUT
       CURL_TIMEOUT
       DATABASE_URL
       DOKKU_APP_TYPE
       GIT_REV
       GMAIL_PASSWORD
       GMAIL_USERNAME
       dokuya_GEMINI_API_KEY
       SECRET_KEY_BASE
       STRIPE_LITE_MONTHLY_PRICE_ID
       STRIPE_PLUS_MONTHLY_PRICE_ID
       STRIPE_SECRET
       STRIPE_ULTIMATE_MONTHLY_PRICE_ID
       STRIPE_WEBHOOK_SECRET
       * MIX_ENV=prod
-----> Checking Erlang and Elixir versions
       Will use the following versions:
       * Stack heroku-24
       * Erlang 27.2
       * Elixir v1.18.1
       Sorry, Erlang '27.2' isn't supported yet or isn't formatted correctly. For a list of supported versions, please see https://github.com/HashNuke/heroku-buildpack-elixir#version-support
remote:  !     Failure during app build
remote:  !     Removing invalid image tag dokku/dokuya:latest
remote:  !     App build failed
To dokuya.ai:dokuya
 ! [remote rejected] main -> main (pre-receive hook declined)
error: failed to push some refs to 'dokuya.ai:dokuya'
```

この問題を解決するには、Dokku が利用する Heroku Stack を [heroku-22 へダウングレードする](https://stackoverflow.com/questions/76957214/dokku-how-to-change-heroku-stack-to-heroku-20-or-heroku-22-from-heroku-18) 必要があります。このスタックには上記の buildpack が対応しており、かつ [最新の Erlang](https://builds.hex.pm/builds/otp/ubuntu-22.04/builds.txt) も含まれています。

```bash
# アプリが現在利用している stack を確認する
# my-app は自分のアプリ名に置き換えてください
dokku buildpacks:report my-app
# heroku-22 へ切り替える
dokku buildpacks:set-property my-app stack gliderlabs/herokuish:latest-22
```

6. あらかじめ環境変数を設定する

サーバー上で次を実行します。

```bash
# 複数の環境変数をまとめて設定できる
# 変数を設定するたびにアプリは再起動する。アプリがまだデプロイされていなくても心配はいらない
dokku config:set my-app ENV=prod COMPILE_ASSETS=1
```

7. データベースマイグレーションを自動で実行する

私のプロジェクトでは Phoenix のマイグレーションが必要なので、デプロイのたびに自動で migrate が走るようにしたいと考えました。そのためには、プロジェクトのルートディレクトリに `app.json` を作成します。

```elixir
{
  "scripts": {
    "dokku": {
      "postdeploy": "mix ecto.migrate"
    }
  }
}
```

ほかにも Dokku のフックはいくつか用意されています。詳しくは [Dokku のドキュメント](https://dokku.com/docs/appendices/file-formats/app-json/?h=postde#scripts) を参照してください。

8. 新しい Git リモートを追加する

```bash
git remote add dokku dokku@<mydomain.com>:<app_name>
```

Dokku サーバーに公開鍵を登録します。

公開鍵は（GitHub への SSH ログインを設定済みなら）通常 `~/.ssh/id_rsa.pub` にあります。ファイルの中身をコピーして、サーバー上の `~/.ssh/authorized_keys` に追記してください。

9. アプリをデプロイする

今後はデプロイしたくなったら、ローカルのプロジェクトで次を実行するだけです。

```bash
git push dokku main
```

リポジトリが GitHub にもある場合は、デプロイ後に通常通り push しておきましょう。

```bash
git push
```

---

**その他：**

1. 私のドメインは Cloudflare で管理しているため、Cloudflare のプロキシ機能を使って https を提供しています。そのため Dokku 側の https 機能は利用していません。
