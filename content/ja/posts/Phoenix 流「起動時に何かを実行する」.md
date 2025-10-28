+++
date = '2025-10-19T01:11:04+09:00'
draft = false
title = 'Phoenix 流「起動時に何かを実行する」'
categories= ['Programming']
tags= ['Elixir', 'Phoenix', 'Cache', 'GenServer']
+++

システムの起動時に何か処理を挟みたいときがあります。たとえばホットデータをデータベースから読み込み、メモリキャッシュをウォームアップするような場合です。

言語やフレームワークによって実現方法はさまざまです。

### Java：Spring Boot

---

一般的には `implements ApplicationListener<ContextRefreshedEvent>` を実装し、`onApplicationEvent` コールバックで処理を行います。

```java
// https://juejin.cn/post/7459021463329865728
@Component
public class CachePrewarmListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private CacheService cacheService; // これはキャッシュサービスの例

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // キャッシュのウォームアップ処理
        System.out.println("キャッシュのウォームアップを開始...");
        List<String> hotData = cacheService.getHotDataFromDatabase(); // データベースからホットデータを取得
        for (String data : hotData) {
            cacheService.putIntoCache(data); // データをキャッシュへ書き込む
        }
        System.out.println("キャッシュのウォームアップが完了しました！");
    }
}
```

### Node.js：Fastify

---

アプリの `ready` コールバックで処理するのが一般的です。

```JavaScript
app.ready(async (err) => {
  if (err) throw err
  app.log.info('キャッシュのウォームアップを開始...')
  const hotData = await cacheService.getHotDataFromDatabase()
  for (const data of hotData) {
    await cacheService.putIntoCache(data)
  }
  app.log.info('キャッシュのウォームアップが完了しました！')
})
```

### Elixir：Phoenix

---

Phoenix では、ルートノードの子プロセスとしてキャッシュローダーを追加する形で実現します。少し変わっています。

実行中の Elixir/Phoenix アプリケーションは木構造のノードで構成され、それぞれがプロセス（BEAM 仮想マシン上の概念であり、OS プロセスとは異なる）です。典型的な Phoenix アプリでは、Application という名前のルートノード配下に、テレメトリー、メトリクス収集、データベース統合、ルーティング、ノード自動検出などの子ノードがぶら下がっています。

これまでにもキャッシュ用ライブラリやジョブキュー用ライブラリを子ノードとして追加してきましたが、今回さらにキャッシュのウォームアップ担当である `Dokuya.Translation.DictionaryLoader` を追加しました。

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

ここで登録するモジュールはいずれも GenServer である必要があります。そのため `Dokuya.Translation.DictionaryLoader` は次のように実装しました。

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
    Logger.info("キャッシュのウォームアップを開始")

    dictionaries =
      Dictionary.load_all()
      |> Enum.map(fn dict ->
        {dict.text, dict}
      end)

    Cachex.put_many!(:translation_dictionary_cache, dictionaries)
    Logger.info("辞書データをメモリへロードしました")
    {:ok, init_arg}
  end
end
```

ロジック自体はシンプルで、ほぼボイラープレートです。`init` コールバック内でウォームアップ処理を行っています。
