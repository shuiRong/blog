+++
date = '2025-10-19T01:11:04+09:00'
draft = false
title = 'Phoenix 风格的「在系统启动时做点什么」'
categories= ['Programming']
tags= ['Elixir', 'Phoenix', 'Cache', 'GenServer']
+++

有时我们会想在系统启动时做点事情，比如：缓存预热（将热点数据从数据库中加载到内存缓存中）。

不同的语言、框架做这个事情的方式都不一样。

### Java：Spring Boot

---

一般是 `implements ApplicationListener<ContextRefreshedEvent>` 然后在 `onApplicationEvent` 回调函数中做这个事情。

```java
# https://juejin.cn/post/7459021463329865728
@Component
public class CachePrewarmListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private CacheService cacheService; // 假设这是一个缓存服务类

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // 缓存预热逻辑
        System.out.println("开始缓存预热...");
        List<String> hotData = cacheService.getHotDataFromDatabase(); // 模拟从数据库加载热点数据
        for (String data : hotData) {
            cacheService.putIntoCache(data); // 将数据写入缓存
        }
        System.out.println("缓存预热完成！");
    }
}
```

### Node.js：Fastify

---

一般是在 app 的 ready 回调函数中做这个事情。

```JavaScript
app.ready(async (err) => {
  if (err) throw err
  app.log.info('开始缓存预热...')
  const hotData = await cacheService.getHotDataFromDatabase()
  for (const data of hotData) {
    await cacheService.putIntoCache(data)
  }
  app.log.info('缓存预热完成！')
})
```

### Elixir：Phoenix

---

则是通过在程序的根节点下添加一个新的缓存加载子节点来实现，比较特别。

运行中的 Elixir/Phoenix 程序其实是一个树状节点，每一个节点都是一个进程（BEAM虚拟机的概念，不等于系统进程）。典型的Phoenix程序中，根节点叫 Application，默认的子节点有负责性能和指标采集模块、数据库集成模块、路由服务模块、节点自动发现模块。

之前曾追加过缓存模块（开源库）、后台任务模块（开源库）、现在又追加了缓存预热模块（`Dokuya.Translation.DictionaryLoader`）。

```elixir
@impl true
def start(_type, _args) do
  children = [
    DokuyaWeb.Telemetry,
    Dokuya.Repo,
    {DNSCluster, query: Application.get_env(:dokuya, :dns_cluster_query) || :ignore},
    {Oban, Application.fetch_env!(:dokuya, Oban)},
    {Phoenix.PubSub, name: Dokuya.PubSub},
    DokuyaWeb.Endpoint,
    {Cachex, [:translation_dictionary_cache]},
    Dokuya.Translation.DictionaryLoader
  ]

  opts = [strategy: :one_for_one, name: Dokuya.Supervisor]
  Supervisor.start_link(children, opts)
end
```

这里的模块都得是一个 GenServer，因此我的 `Dokuya.Translation.DictionaryLoader` 实现如下：

```elixir
defmodule Dokuya.Translation.DictionaryLoader do
  use GenServer
  require Logger

  alias Dokuya.Translation.Dictionary

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  @impl true
  def init(init_arg) do
    Logger.info("开始缓存预热")

    dictionaries =
      Dictionary.load_all()
      |> Enum.map(fn dict ->
        {dict.text, dict}
      end)

    Cachex.put_many!(:translation_dictionary_cache, dictionaries)
    Logger.info("将词典数据加载到内存中成功")
    {:ok, init_arg}
  end
end
```

逻辑比较简单，基本都是样板代码，在 `init` 回调函数中实现了缓存预热逻辑。
