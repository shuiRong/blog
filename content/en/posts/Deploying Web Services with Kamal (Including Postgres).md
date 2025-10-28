+++
date = '2025-05-03T13:45:19+09:00'
draft = false
title = 'Deploying Web Services with Kamal (Including Postgres)'
categories= ['Programming']
tags= ['Kamal', 'Deploy']
+++

DHH’s company moved every product off the major cloud providers with Kamal and [saves two million dollars per year](https://world.hey.com/dhh/our-cloud-exit-savings-will-now-top-ten-million-over-five-years-c7d9b5bd). That story made me curious.

I tried Kamal and came away impressed:

1. It is built on Docker, so you inherit all the benefits of containers.
2. It is comparatively easy to grasp and use. The learning curve is low (although the docs are still barebones. When you hit unexpected errors, you need to dig around the internet or experiment. As the ecosystem grows and the team improves the docs, this will become less of a problem).
3. It covers the core needs of deploying and running web apps in production: zero-downtime deploys, rolling restarts, resource bridging, remote builds, auxiliary service management, and more.

This article complements the official documentation.

Instead of repeating the terse instructions from the docs, I walk through them in a hands-on way and fill in the background knowledge that a real-world deployment requires. I also explain why certain configuration values and scripts exist—understanding the underlying mechanics prevents mistakes and makes debugging easier.

---

## Prerequisites

1. A web project that already runs locally (with a PostgreSQL database).
2. A server you can reach via `ssh` from the command line (with a password or key file).
3. [Kamal](https://kamal-deploy.org) installed and initialized with `kamal init`.
4. A personal access token generated on Docker Hub.

Running `kamal init` drops a few files into the project directory. The two we’ll use most are:

1. `.kamal/secrets`
2. `config/deploy.yml`

The docs do cover them, but only superficially. Many settings are explained in a single line, as if the reader already understood service deployment, Docker, databases, and networking. I spent hours searching for answers and experimenting. Let’s unpack the configuration together.

## `.kamal/secrets`

[Docs](https://kamal-deploy.org/docs/configuration/environment-variables/)

```yml
# Docker Hub access token. After building locally, Kamal uses this key and your username
# to push the image to the designated Docker Hub repository.
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# XAI API key
XAI_API_KEY=$XAI_API_KEY
```

This file stores the environment variables that Kamal can load into your deployment workflow.

Each line maps an environment variable. For example, `KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD` tells Kamal to look for a `KAMAL_REGISTRY_PASSWORD` variable in the current shell and assign that value to the `KAMAL_REGISTRY_PASSWORD` slot that your deployment uses. Those variables become available in `config/deploy.yml`.

You can populate values via shell environment variables, shell commands (e.g., reading from a password manager), or any of the other options in the docs.

A few tips:

1. Reserve this file for secrets—API keys, database credentials, etc. You *can* store plain settings such as `TEST_MODE=123`, but that breaks the intent of the file. Non-sensitive configuration belongs under the `env.clear` section of `config/deploy.yml`.
2. This file may live in your Git repository (including GitHub). It contains references to environment variables, not the secrets themselves.

## `config/deploy.yml`

[Docs](https://kamal-deploy.org/docs/configuration/deploy/)

```yml
service: dokuya
image: docker.io/shuirong/dokuya
# rack or exec
servers:
  - root@185.194.123.12
# SSH に使う公開鍵（省略可）
ssh:
  user: root
  keys:
    - ~/.ssh/id_ed25519
# env-file: config/deploy.env
registry:
  username: shuirong
# start, redeploy, stop
# registry: registry.digitalocean.com/railsanity

# 以下は最小構成
env:
  clear:
    # 定数向け。SSH 接続の際に、Kamal はここで列挙された変数を目標サーバーに書き込み、コンテナに注入する。
    # SU_PASSWORD: 私の SSH パスワード
    # SECRET_KEY: 私のシークレットキー
  secret:
    - SU_PASSWORD
    - SECRET_KEY

app:
  # 例：docker run -it kamal-app bash
#    cmd: bundle exec puma -C config/puma.rb
  # カスタム環境変数
  # env:
  #   clear:

### アプリ構成
  build:
    # ビルドのための SSH オプションや秘密鍵の定義など
    # secrets:
      # - dockerconfigjson:/root/.docker/config.json
    # ブラフ・コマンド:
    # - app bin/rails deploy:makedirs
  # Healthcheck: https://kamal-deploy.org/docs/configuration/healthcheck/
  # 例
  # healthcheck:
  #   path: /up
  #   port: 3000
  #   method: GET
```

Hold on—that doesn’t look like a real config. Right, this is the template Kamal generates. Let’s see what the file looks like in a working project.

```yml
service: dokuya
image: docker.io/shuirong/dokuya

servers:
  - root@194.195.83.189

ssh:
  user: root
  keys:
    - ~/.ssh/id_ed25519

registry:
  username: shuirong
  # 使用する Docker Hub アクセストークン。
  password:
    - KAMAL_REGISTRY_PASSWORD

builder:
  arch: amd64

env:
  clear:
    PHX_HOST: 0.0.0.0
  secret:
    - XAI_API_KEY
    - DATABASE_URL

aliases:
  shell: app exec --interactive --reuse "bash"
  logs: app logs

accessories:
  postgres:
    image: postgres:15
    host: 185.194.123.12
    port: '127.0.0.1:5432:5432'
    env:
      clear:
        POSTGRES_USER: 'postgres'
        POSTGRES_DB: 'production_db'
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data
```

Now it makes sense. We can tweak the settings while Kamal is running. Let’s walk through the sections:

```yml
service: dokuya
image: docker.io/shuirong/dokuya
```

- `service`: the name of the project inside Kamal. You’ll use it in commands—e.g., `kamal app logs`.
- `image`: a Docker repository. Kamal builds images locally using the configuration in this file, logs into Docker Hub with the credentials below, and pushes the images there. If you prefer, you can bake Docker images elsewhere (say, in a GitHub Actions pipeline uploading to Docker Hub) and have Kamal redeploy them—the [issue thread](https://github.com/basecamp/kamal/discussions/1132#discussioncomment-13022584) explains how.

```yml
servers:
  - root@194.195.83.189

ssh:
  user: root
  keys:
    - ~/.ssh/id_ed25519

registry:
  username: shuirong
  password:
    - KAMAL_REGISTRY_PASSWORD
```

- `servers`: the target host(s). Add as many as you need. If your database runs on a different machine, list it under `accessories`.
- `ssh`: which user and which key Kamal should use to log in.
- `registry`: Docker Hub credentials. The password stems from `.kamal/secrets`, so remember to define `KAMAL_REGISTRY_PASSWORD` there.

```yml
builder:
  arch: amd64
```

This tells the builder which CPU architecture to target when it builds the Docker image. Most Linux servers are `amd64`, so the default works. If you build on an Apple Silicon Mac, you might run into mismatched architectures; adjust this if necessary.

```yml
env:
  clear:
    PHX_HOST: 0.0.0.0
  secret:
    - XAI_API_KEY
    - DATABASE_URL
```

Declare every environment variable that your app will need at runtime. Only the variables listed here get injected into the container. Logging them in `.kamal/secrets` alone isn’t enough.

- `clear`: non-sensitive values you prefer not to hardcode in source.
- `secret`: variables defined in `.kamal/secrets`.

Again, the final list depends on your application.

```yml
aliases:
  shell: app exec --interactive --reuse "bash"
  logs: app logs
```

Command aliases that make day-to-day work smoother (`kamal shell`, `kamal logs`, etc.).

```yml
accessories:
  postgres:
    image: postgres:15
    host: 185.194.123.12
    port: '127.0.0.1:5432:5432'
    env:
      clear:
        POSTGRES_USER: 'postgres'
        POSTGRES_DB: 'production_db'
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data
```

“Accessories” are sidecar services that the main web app depends on but that nobody accesses directly from the public internet.

- `postgres` is the name used inside Kamal. When you run `docker ps` on the server you’ll see a container named `<service>-postgres` (e.g., `dokuya-postgres`). That name matters because the application connects to the database via Docker networking, using the container name instead of `localhost`.
- `image`: which database image to run.
- `host`: which server to run it on. Use the same host as the app if you want them colocated.
- `port`: how to map the service ports.
- `env`: the environment variables required by the accessory (the secrets must also be listed in `.kamal/secrets`).
- `directories`: bind mounts for data persistence. Without them you’ll lose data whenever the container restarts. Check the documentation of whatever database you use; different images may store their data elsewhere.

To deploy directly from Docker Hub without a local build, see the [discussion](https://github.com/basecamp/kamal/discussions/1132#discussioncomment-13022584).

---

## How I Deploy

1. `kamal env push` writes the environment variables defined in `config/deploy.yml` to `/root/.kamal/env/env.dokuya` on the server.
2. Update the source code.
3. `kamal deploy` builds a new Docker image on your workstation, pushes it to Docker Hub, fetches it on the server, and swaps it in with zero downtime.

With the workflow in place, I no longer worry about Phoenix/Elixir deployments.
