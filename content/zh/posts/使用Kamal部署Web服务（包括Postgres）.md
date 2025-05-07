+++
date = '2025-05-03T13:45:19+09:00'
draft = false
title = '使用 Kamal 部署 Web 服务（包括 Postgres ）'
categories= ['Programming']
tags= ['Kamal', 'Deploy']
+++

DHH 的公司借助在 Kamal 的帮助下，将公司的所有产品从云服务商迁出，[每年节省 200 万美元](https://world.hey.com/dhh/our-cloud-exit-savings-will-now-top-ten-million-over-five-years-c7d9b5bd)。这让我对 Kamal 产生了兴趣。

初次相识，看上去还不错：

1. 构建在 Docker 之上，所以拥有所有容器化技术的优势。
2. 理解和使用起来都相对简单，不会带来多少学习成本（但是文档仍旧过于简单，因此遇到意外问题时需要从网上翻找答案，但我觉得随着互联网上的相关内容越来越多，以及官方团队对文档的完善，这里不会有什么问题）
3. 提供了：零停机部署、滚动重启、资源桥接、远程构建、辅助服务管理以及使用 Docker 在生产环境中部署和管理 Web 应用所需的一切功能。

这篇文章是对 Kamal 文档的一个补充。

即把文档中一笔带过的配置描述按照最接地气的方式重新解释了一遍，又顺便讲了很多文档中没有提到，但实际部署服务可能需要了解到的知识。
同时也顺便解释了程序、配置、命令背后的实现原理。因为只有理解了原理才不容易搞错，才不会把它当成神奇的东西，出了问题才能知道大概是哪里的问题。

---

## 前置条件：

1. 一个可以在本地运行的 Wen 项目（包含 Postgres 数据库）
2. 一个服务器，你可以通过命令行 ssh 命令登录（不管是通过密码还是密钥文件）
3. 安装 [Kamal](https://kamal-deploy.org)，并且已经按照文档初始化好（`kamal init`）
4. 到 Dockerhub 上创建一个 access token

`kamal init` 会在项目目录下生成一些文件，但主要用到的是两个：

1. `.kamal/secrets`
2. `config/deploy.yml`

当然，Kamal 文档里有介绍。但是过于简单，很多配置的解释一笔带过，似乎作者在假设用户已经对服务部署、Docker、数据库、网络很了解，一看就明白怎么回事了。因此，一旦遇到了预料之外的问题时，就比较难受。我在网上查询资料以及反复测试上浪费了大量的时间。

下面开始解释配置。

## .kamal/secrets：

[原文档](https://kamal-deploy.org/docs/configuration/environment-variables/)

```yml
# 上面创建的 Dockerhub 的 access token，Kamal 构建了本地镜像后，会利用这个 Key + 你的用户名，上传到你指定的 Dockerhub 仓库里。
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# XAI 的 API Key
XAI_API_KEY=$XAI_API_KEY
```

这是存放项目会用到的环境变量的地方。

而且里面的每一行只是变量的映射。比如：`KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD`。它的意思是：Kamal 从当前的系统环境变量下寻找一个叫`KAMAL_REGISTRY_PASSWORD`的变量，拿到它的数据后，赋值给左边的`KAMAL_REGISTRY_PASSWORD` 变量。变量的使用在哪里呢？在 `config/deploy.yml` 文件里。

除了从环境变量里获取数据，也可以从借助 Linux 命令从任何其他地方获取，还可以从密码管理器里获取。这部分使用方法参考文档。

其他需要说明的：

1. 这个文件里最好只存放密钥等机密环境变量的配置，当然也可以把任何明文环境变量放在这里，比如：`TEST_MODE=123`，但和设计初衷不同，违反直觉的事情不要做。这类环境变量应该放在 `config/deploy.yml` 里的 `env.clear` 下面。
2. 这个文件可以上传到 Github，因为毕竟它里面没有机密信息的明文。

## config/deploy.yml：

这是最重要的配置文件

```yaml
# 服务名称。用来标识服务，因为 Kamal 支持在一台机器上部署多个程序，所以需要个标识。
# 而且这个 service 名字也会出现在数据库 Docker 服务的命名里。
service: test

# 必须和你 Dockerhub 下的仓库名字一致。
# 注意，如果你不想要公开镜像，需要到 Dockerhub 上面手动设置为 private 。但不管怎么样，都不影响 Kamal 部署
image: shuirong/test

# 如果你不想每次使用 Kamal 时都手动输入服务器的登录密码，就反注释这几行配置。
# 这样，Kamal 在需要操作服务器时，都会自动读取这里的私钥（不一定时pem结尾，只要是有效的私钥文件即可）
# ssh:
#   keys:
#     - ~/.ssh/xxxx.pem

# 你想部署的程序的配置，一般大家部署的都是 Web 服务
servers:
  # 这里可以使用默认名称：web
  web:
    # 你想部署的服务器的 IPV4 地址
    - 185.194.123.12

# 你的程序前面会有一个网关程序，这里是对网关进行一些配置。
# Kamal 使用的是 Traefik 来作为网关。
proxy:
  # 通过 Let’s Encrypt 实现签发 SSL 自动证书，也就是说实现 https 访问。
  # 如果使用多个 Web 服务器，请移除此部分，并确保在负载均衡器处终止 SSL。
  # 注意：如果你的域名托管在 Cloudflare，或者域名的 DNS 托管在 Cloudflare，
  # 需要到域名的 DNS 设置那里，确保 Proxy Status的值是： DNS Only，
  # 这样访问域名就不会出现：「将您重定向的次数过多」问题。因为两边的 SSL 服务 只能启用一个。
  # 关掉这边的 ssl，然后 Cloudflare 那边选择 Proxied 也是可以的。
  ssl: true
  # 项目的域名。一台机器上部署的 Web 服务的 host 不能相同。
  # 如果未来一台服务器上部署多个Web服务的话，网关需要根据用户是通过哪个域名进来的，
  # 来了解需要将请求代理到哪个 Web 程序里。
  host: iyaku.ai
  # 程序暴露的端口。如果不设置，默认值是 80
  app_port: 8080
  # 如果您设置ssl为true，网关将停止将请求头转发到您的应用，除非明确设置forward_headers: true。
  # 这个设置一般用不上，但如果有一天想通过解析 X-Forwarded-For 请求头来确认用户的 IP，
  # 然后发现获取不了这个请求头时，就会用上这个设置。
  # 可以默认开着。
  forward_headers: true

# Dockerhub 相关配置
# 当然，也可以选择其他的镜像托管服务。那就参考文档，我这里只说 Dockerhub
registry:
  # Dockerhub 账户的用户名
  username: xxxx

  # 这就是前面提到的 Dockerhub 账户的 access token，上传镜像时使用。
  password:
    - KAMAL_REGISTRY_PASSWORD

# Kamal 会在本地构建镜像，所以你需要提前启动 Docker Desktop 软件。
builder:
  # 制定服务器的架构，这样 Docker 才能构建出来符合架构的镜像。
  # 一般的 Linux 服务器都是 amd 架构，所以默认值即可。如果是其他架构，记得调整这个值。
  # 如果你的电脑跟我一样，是苹果 M 系列的 arm 架构的电脑，那打包 amd 架构的镜像时，可能会出现奇怪问题，迫于篇幅不在这里讲了
  arch: amd64

# 上面的 Web 服务会用到的所有环境变量，都声明在这里。
# 只有声明了，Kamal 才会注入到 Web 服务的容器环境变量里，
# 不是说写在 `.kamal/secrets` 文件里就可以了。
env:
  # 项目会用到的所有非机密信息的环境变量都可以写在这里，如果不想直接写在源码里的话。
  # 这里是否需要声明一些变量，其实最终取决于你的程序是否需要。如果程序不需要从外部的环境变量里读取什么，
  # 那这里全部是空的，不需要设置什么。
  # 所以你的环境变量也估计跟我的不一样。
  clear:
    # 设置 PHX_HOST 为 0.0.0.0 ，这样就可以从容器外访问 Phoenix 应用。
    PHX_HOST: 0.0.0.0
  secret:
    - XAI_API_KEY
    - DATABASE_URL

# 给 Kamal 添加别名，也就是一些长命令的简短命令
aliases:
  # 直接进到 web 服务那个 Docker 容器的命令行里
  shell: app exec --interactive --reuse "bash"
  # 查看 web 服务和下面的数据库服务的日志。
  logs: app logs

# 这个词语的意思是配件，也就是 Web 服务的配套服务，而且也是不需要从服务器外部直接访问到的服务。
accessories:
  # 这里是服务在 Kamal 里的名字，我也起成了 postgres，当然可以用别的。
  # 这里我强调是「在 Kamal 里的名字」，是因为如果你到服务器上，然后通过 `docker ps` 查看有什么程序，
  # 会发现数据库的 Docker 容器名称不是这个，而是开头的 <service name> + postgres，比如：test-postgres
  # test-postgres 这个是 Docker 容器服务的名字，这个很重要。因为你要从 Web 服务器连接数据库服务，
  # 就是通过这个地址来连接的，而不是 postgres。
  postgres:
    # 需要安装的数据库镜像版本
    image: postgres:15
    # 数据库需要安装的服务器地址。如果你想将数据库部署到和 Web 服务一样的机器上，就填一样的地址。
    # 这个配置一开始让我愣了一下，因为我潜意识里认为肯定是部署在同一台机器上，为啥还需要我额外设置。
    host: 185.194.123.12
    # postgres 的 Docker 容器需要映射的端口，就写这个。
    port: '127.0.0.1:5432:5432'
    # postgres 需要的环境变量信息。其他数据库不一样需求相同，具体参考对应数据库文档。
    env:
      clear:
        POSTGRES_USER: 'postgres'
        POSTGRES_DB: 'production_db'
      secret:
        # 和前面的 env 配置一样，这里使用的前提是，必须在 `.kamal/secrets` 里已经声明了变量。
        - POSTGRES_PASSWORD
    # postgres 容器里数据文件的映射，必须将其映射到宿主机器，不然容器一旦重启，数据将会全部丢失。
    # 其他数据库容器的数据路径不一定相同，具体看对应文档。
    directories:
      - data:/var/lib/postgresql/data
```

最后，关于 Kamal 的部署，如果有人想要使用 Kamal 直接部署 Dockerhub 上的镜像，而不想从本地构建，那么请参考这个 [issue](https://github.com/basecamp/kamal/discussions/1132#discussioncomment-13022584)
