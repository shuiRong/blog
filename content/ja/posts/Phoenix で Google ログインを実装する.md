+++
date = '2025-08-28T23:03:26+09:00'
draft = false
title = 'Phoenix で Google ログインを実装する'
categories= ['Programming']
tags= ['Elixir', 'Phoenix', 'Google', 'OAuth', 'Authentication']
+++

chriis さんの [記事](https://www.chriis.dev/opinion/implementing-google-authentication-in-a-liveview-application) に感謝します。とてもよくまとまっています。
ただ、実際に試してみるといくつかハマりポイントがあり、最終的に自分なりのやり方で解決しました。ここではその手順を記録しておきます。

> 前提：ローカルで動作する Phoenix アプリ（LiveView である必要はありません）と、既存の[認証（ユーザー）システム](https://hexdocs.pm/phoenix/1.8.0/mix_phx_gen_auth.html)が用意されていること。

### 設定と準備

---

[Google Cloud Console](https://console.cloud.google.com/apis/dashboard) にアクセスし、新しいプロジェクトを作成するか、既存プロジェクトを選択します。
![create new project](/img/phoenix_google_auth/create_project.webp)

1. 作成したプロジェクトのホームに移動します。
2. 左メニューの「OAuth 同意画面」を開きます。
3. 左メニューの「メニュー」→「開始」に進み、アプリ作成ウィザードで基本情報を入力します。

   ![create new application](/img/phoenix_google_auth/create_application.webp)

4. 「OAuth クライアントを作成」をクリックし、基本情報を入力します。ここで 2 つ重要な項目があります。
   ![create new application](/img/phoenix_google_auth/get_client_id.webp)

- 「承認済みの JavaScript 生成元」には、開発・本番の URL を設定します（例：`http://localhost:4000`、`https://xxxxx.com`）。
- 「承認済みのリダイレクト URI」には、各 URL に `/auth/google/callback`（後でコード側で定義するパスに合わせてください）を付けたものを設定します（例：`http://localhost:4000/auth/google/callback`、`https://xxxxx.com/auth/google/callback`）。

5. 画面に表示される「クライアント ID」と「クライアント シークレット」を安全な場所に控えておきます（一定時間が経つとこの画面では再表示できません）。
6. ページ下部の「作成」または「保存」を押します。

### テンプレートコードを準備する

---

続いて Phoenix プロジェクトに戻ります。

1. [Ueberauth](https://github.com/ueberauth/ueberauth) をインストールします。

```elixir
# mix.exs に追加
{:ueberauth_google, "~> 0.10.8"}
```

`mix deps.get` を実行して依存関係を取得します。

2. 設定を追加します。

まずは `config/dev.exs`。

```elixir
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, [default_scope: "email profile"]}
  ]
```

次に `config/prod.exs`。

```elixir
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, [default_scope: "email profile", callback_scheme: "https"]}
  ]
```

両者の違いは、本番環境には `callback_scheme` を追加している点です。Ueberauth のデフォルトは `http` で、開発環境（`http://localhost:4000`）には都合が良いものの、Google 側の設定と合わせるため本番では `https` を明示する必要があります。chriis さんの記事では触れられていませんが、これを設定しないと Google ログインが失敗してしまいます。

最後に `config/runtime.exs`。

```elixir
# 先ほど控えた「クライアント ID」と「クライアント シークレット」を
# システム環境変数に設定し、ここで参照します。
config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id: System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET")
```

3. ユーザーテーブルを調整します。

> 以下の `XXXXX` は自分のアプリ名に置き換えてください。

`lib/dokuya/accounts/user.ex` に以下を追加します。

```elixir
schema "users" do
  ...
  field :is_oauth_user, :boolean, default: false
  ...
end

# 実際のユーザースキーマに合わせてフィールドを調整してください。
def oauth_registration_changeset(user, attrs, opts \\ []) do
  user
  |> cast(attrs, [:email])
  |> validate_required([:email])
  |> validate_email(opts)
  |> put_change(:is_oauth_user, true)
end
```

スキーマにフィールドを追加したら、`mix ecto.gen.migration add_is_oauth_user_to_users` でマイグレーションを作成します。

生成されたマイグレーションファイルに以下を追記します。

```elixir
defmodule Dokuya.Repo.Migrations.AddOauthUser do
  use Ecto.Migration

  def change do
    alter table(:users) do
      add :is_oauth_user, :boolean, default: false
      # OAuth ユーザーはパスワード不要なので hashed_password を NULL 可にする
      modify :hashed_password, :string, null: true
    end
  end
end
```

その後 `mix ecto.migrate` を実行してスキーマを更新します。

続いて `lib/xxxxx/accounts.ex` に以下を追加。

```elixir
def register_oauth_user(attrs) do
  %User{}
  |> User.oauth_registration_changeset(attrs)
  |> Repo.insert()
end
```

4. コアとなる Controller を追加します。

`lib/xxxxxx_web/controllers/google_auth_controller.ex` を作成し、次の内容を記述します。

```elixir
defmodule XXXXWeb.GoogleAuthController do
  require Logger
  use XXXXXWeb, :controller
  plug Ueberauth

  alias XXXXX.Accounts
  alias XXXXXWeb.UserAuth

  def request(conn, _params) do
    Phoenix.Controller.redirect(conn, to: Ueberauth.Strategy.Helpers.callback_url(conn))
  end

  def callback(conn, params) do
    create(conn, params, "Welcome back!")
  end

  # Google ログイン処理
  defp create(%{assigns: %{ueberauth_auth: auth}} = conn, _params, info) do
    email = auth.info.email

    case Accounts.get_user_by_email(email) do
      nil ->
        # ユーザーが存在しないので新規作成する
        # 今回は email だけ使っていますが、auth.info にはユーザー情報がもっと含まれています。
        case Accounts.register_oauth_user(%{
               email: email
             }) do
          {:ok, user} ->
            Logger.info("Google login success: #{inspect(user)}")

            conn
            |> put_flash(:info, info)
            |> UserAuth.log_in_user(user, %{"remember_me" => "true"})

          {:error, changeset} ->
            Logger.error("Failed to create user #{inspect(changeset)}.")

            conn
            |> put_flash(:error, "Failed to create user.")
            |> redirect(to: ~p"/")
        end

      user ->
        # 既存ユーザー。必要に応じてセッションなどを更新する
        conn
        |> put_flash(:info, info)
        |> UserAuth.log_in_user(user, %{"remember_me" => "true"})
    end
  end
end
```

5. ルーティングを追加します。

`lib/xxxx_web/router.ex` に以下を追記します。

```elixir
  scope "/auth", DokuyaWeb do
    # これを忘れるとログイン時に fetch_session 関連のエラーが出ます。
    pipe_through :browser

    get "/:provider", GoogleAuthController, :request
    get "/:provider/callback", GoogleAuthController, :callback
  end
```

6. 最後に、ログインボタンを任意のページに配置します。

```elixir
<.button href={~p"/auth/google"}>Login with Google</.button>
```

**これで完了です。ユーザーは Google アカウントで簡単にログインできるようになりました。**
