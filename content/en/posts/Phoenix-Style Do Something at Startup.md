+++
date = '2025-10-19T01:11:04+09:00'
draft = false
title = 'Phoenix-Style "Do Something at Startup"'
categories= ['Programming']
tags= ['Elixir', 'Phoenix', 'Cache', 'GenServer']
+++

Every now and then we need to run some work as soon as the system boots—say, warming a cache by loading hot data from the database into memory.

Different languages and frameworks approach this in different ways.

### Java: Spring Boot

---

You typically implement `ApplicationListener<ContextRefreshedEvent>` and do the work inside the `onApplicationEvent` callback.

```java
// https://juejin.cn/post/7459021463329865728
@Component
public class CachePrewarmListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private CacheService cacheService; // Example cache service

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // Cache warm-up logic
        System.out.println("Starting cache warm-up...");
        List<String> hotData = cacheService.getHotDataFromDatabase(); // Simulate loading hot data
        for (String data : hotData) {
            cacheService.putIntoCache(data); // Write it to the cache
        }
        System.out.println("Cache warm-up finished!");
    }
}
```

### Node.js: Fastify

---

Fastify runs it in the app’s `ready` callback.

```JavaScript
app.ready(async (err) => {
  if (err) throw err
  app.log.info('Starting cache warm-up...')
  const hotData = await cacheService.getHotDataFromDatabase()
  for (const data of hotData) {
    await cacheService.putIntoCache(data)
  }
  app.log.info('Cache warm-up finished!')
})
```

### Elixir: Phoenix

---

Phoenix takes a different tack: add a child process under the application’s supervision tree to handle the cache warm-up.

A running Elixir/Phoenix app is a tree of processes (in the BEAM sense, not OS processes). In a typical Phoenix project the root node is the application, and the default children include telemetry/metrics collection, the database integration, routing, node discovery, and so on.

Over time I’ve added custom children for caching (via an open-source library), background jobs (another library), and now a warm-up worker named `Dokuya.Translation.DictionaryLoader`.

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

Each child must be a GenServer, so `Dokuya.Translation.DictionaryLoader` looks like this:

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
    Logger.info("Starting cache warm-up")

    dictionaries =
      Dictionary.load_all()
      |> Enum.map(fn dict ->
        {dict.text, dict}
      end)

    Cachex.put_many!(:translation_dictionary_cache, dictionaries)
    Logger.info("Loaded dictionary data into memory")
    {:ok, init_arg}
  end
end
```

The logic is straightforward boilerplate—the warm-up lives entirely in the `init` callback.
