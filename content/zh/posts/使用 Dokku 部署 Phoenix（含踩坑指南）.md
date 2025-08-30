+++
date = '2025-08-21T22:41:51+09:00'
draft = false
title = '使用 Dokku 部署 Phoenix（含踩坑指南）'
categories= ['Programming']
tags= ['Elixir', 'Deploy', 'Phoenix', 'Dokku', 'Postgres', 'Cloudflare']
+++

[通过 Kamal 部署 Web 服务](./使用Kamal部署Web服务（包括Postgres）.md) 的一段时间后，还是决定放弃 Kamal，理由如下：

1. 官方文档过于简洁，很多地方只是一句带过，似乎它预设用户是一个部署经验丰富的工程师。导致遇到问题就得去网上自己找翻找答案、甚至翻源码。
2. 在 Phoenix 项目中使用时，环境变量的处理流程过于复杂（4 步才能流到代码里，多出来了中间两个环节）：系统环境变量 --> `.kamal/secrets` --> `configs/deploy.yaml` --> `configs/runtime.exs`

一番研究和对比后，决定转向 [Dokku](https://dokku.com/)，Heroku 的开源竞品。生态远比 Kamal 丰富，文档也很齐全。

## Dokku 工作原理

---

首先得解释下 Dokku 大概的工作原理，这对于搞清楚状况、排查问题很有帮助，如果已经清楚，可以直接跳到下面看「部署步骤」：

1. 它是一个安装在服务器上的程序，运行了一个 Git 服务器（当然还有别的模块）。当你在其他地方的代码仓库里向该 Git 服务器推送代码时，它会收到通知（Git Hook）然后运行部署流程。
2. 你的程序在服务器上是通过 Docker 实例的方式运行的。而 Docker Image 是在你推送代码到服务器上后，Dokku 在服务器上构建出来的。构建时优先使用 Dockefile，没有的话就用 Herokuish（约等于 Heroku 的 Buildpack，即用来打包 Docker Image 的一系列脚本）

## 部署步骤

---

**首先，最终部署的成果：** Phoenix 程序，连接 Postgres 数据库，使用 Cloudflare 托管的域名。

下面是看官方文档 + 网友文章 + 踩坑后整理出来的步骤：

0. 参考[官方文档](https://dokku.com/docs/getting-started/installation/) 在服务器上安装好 Dokku、设置好 SSH 密钥和域名
1. 在服务器上，创建一个应用程序（和你本地开发的程序同名即可）

```bash
# on the Dokku host
dokku apps:create ruby-getting-started
```

2. 安装 dokku postgres 插件（railsdatabase 名字可以修改）

```bash
# on the Dokku host
# install the postgres plugin
# plugin installation requires root, hence the user change
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git0

# create a postgres service with the name railsdatabase
dokku postgres:create railsdatabase
```

3. 给应用追加数据库访问 URL 的配置

postgres:link 的原理就是

```bash
# on the Dokku host
# each official datastore offers a `link` method to link a service to any application
dokku postgres:link railsdatabase ruby-getting-started
```

4. 追加 Herokuish（buildpacks）

```bash
echo "https://github.com/HashNuke/heroku-buildpack-elixir.git" >> .buildpacks
# 注意，使用 gigalixir 的，它解决了 https://github.com/gjaldon/heroku-buildpack-phoenix-static.git 仓库里的一个问题：https://github.com/gjaldon/heroku-buildpack-phoenix-static/issues/127
echo "https://github.com/gigalixir/gigalixir-buildpack-phoenix-static.git" >> .buildpacks
```

4. 配置 buildpacks

```bash
# 根目录下创建 elixir_buildpack.config
touch elixir_buildpack.config
```

将下面内容添加进去。

注意根据你自己的项目的实际需要调整环境变量名（config_vars_to_export），下面是我自己的版本

```txt
# 版本名字必须包含小数点，也就是跟链接里的出现的一模一样才行，我之前设置的27，也不行
erlang_version=27.0

# Elixir version
elixir_version=1.18.1

# Always rebuild from scratch on every deploy?
always_rebuild=true

# Export heroku config vars
config_vars_to_export=(DATABASE_URL GMAIL_PASSWORD GMAIL_USERNAME SECRET_KEY_BASE)
```

```bash
# 根目录下创建 phoenix_static_buildpack.config
touch phoenix_static_buildpack.config
```

将下面内容添加进去

```txt
# 声明一个比较高的node版本，否则会安装一个非常低的node版本，并且最后报错
node_version=23.11.0
npm_version=10.9.2
```

5. 降级 Dokku 上的 Heroku Stack 配置
   前面安装的 [HashNuke/heroku-buildpack-elixir](https://github.com/HashNuke/heroku-buildpack-elixir?tab=readme-ov-file#version-support) 不支持最新的 stack heroku-24。但在服务器上安装的 Dokku 已经默认使用的是最新的 stack heroku-24，
   所以会导致在部署时报错

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

要解决这个问题的话，就需要[降级 Dokku 支持的 Heroku Stack 为 heroku-22](https://stackoverflow.com/questions/76957214/dokku-how-to-change-heroku-stack-to-heroku-20-or-heroku-22-from-heroku-18)，因为上面这个库支持它，而且里面也有最[新版本的 erlang](https://builds.hex.pm/builds/otp/ubuntu-22.04/builds.txt)

```bash
# 检查应用当前使用的 stack
# 记得修改 my-app 成你的应用名称
dokku buildpacks:report my-app
# 切换到 heroku-22
dokku buildpacks:set-property my-app stack gliderlabs/herokuish:latest-22
```

6. 给程序提前设置好环境变量

在服务器上使用

```bash
# 可以同时设置多个环境变量
# 每次设置后，都会触发重启应用程序。如果程序还没部署也没关系，尽管设置
dokku config:set my-app ENV=prod COMPILE_ASSETS=1
```

7. 自动更新数据库迁移文件

我的项目需要操作数据库，本地的 Phoenix 程序中存在一些迁移文件，我希望在每次部署时都能自动执行迁移命令。
这需要在项目根目录下创建一个 `app.json` 文件

```elixir
{
  "scripts": {
    "dokku": {
      "postdeploy": "mix ecto.migrate"
    }
  }
}
```

Dokku 还有其他部署钩子可以用在其他用途，可以参考[Dokku 文档](https://dokku.com/docs/appendices/file-formats/app-json/?h=postde#scripts)。

8. 设置新的远程 git origin

```bash
git remote add dokku dokku@<mydomain.com>:<app_name>
```

将你的 git 公钥添加到 Dokku 服务器上

公钥地址（假如你之前本地已经配置过 ssh 登录 Github 的话，一般是存在这个文件）：`~/.ssh/id_rsa.pub`
将文件内容手动追加到服务器上的 `~/.ssh/authorized_keys` 文件中

9. 部署程序

以后只要想部署一次代码，就在本地项目下运行该命令即可：

```bash
git push dokku main
```

但是可能你的仓库之前是保存在 Github 里，所以每次每次部署完，还是需要手动再推送代码到 Github ：

```bash
git push
```

---

**其他：**

1. 因为我的域名托管在 Cloudflare，所以可以直接利用它的Proxy给域名提供 https 服务。就不需要使用 Dokku 的 https 相关功能了。
