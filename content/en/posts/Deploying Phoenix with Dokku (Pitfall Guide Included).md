+++
date = '2025-08-21T22:41:51+09:00'
draft = false
title = 'Deploying Phoenix with Dokku (Pitfall Guide Included)'
categories= ['Programming']
tags= ['Elixir', 'Deploy', 'Phoenix', 'Dokku', 'Postgres', 'Cloudflare']
+++

After spending some time [deploying web services with Kamal](<./Deploying Web Services with Kamal (Including Postgres).md>), I eventually decided to move on. Here is why:

1. The official documentation is extremely terse. Many topics are brushed over in a sentence, as if every user were already a seasoned deployment engineer. Whenever something went wrong I had to scour the internet or even read the source.
2. In Phoenix projects the environment-variable flow felt unnecessarily convoluted. Variables travel through four hops before they reach the code: system environment → `.kamal/secrets` → `configs/deploy.yaml` → `configs/runtime.exs`.

After a bunch of research and comparisons I switched to [Dokku](https://dokku.com/), the open-source cousin of Heroku. The ecosystem is richer than Kamal’s and the documentation is far more complete.

## How Dokku Works

---

Let me first sketch how Dokku operates. Understanding the pipeline makes troubleshooting much easier. If you already know, feel free to jump to “Deployment Steps”.

1. Dokku runs on your server and bundles a Git service (plus several other modules). When you push code from a different repository, Dokku receives the Git hook and kicks off the deployment process.
2. Your application runs as a Docker container on the server. Dokku builds the Docker image after it receives your push. If there is a Dockerfile, it uses that; otherwise it falls back to Herokuish (roughly equivalent to Heroku buildpacks—a collection of scripts for building Docker images).

## Deployment Steps

---

**Goal:** deploy a Phoenix application wired to a Postgres database, fronted by a domain managed on Cloudflare.

The steps below are stitched together from official docs, community blog posts, and a series of mishaps.

0. Follow the [official installation guide](https://dokku.com/docs/getting-started/installation/) to install Dokku on the server, configure SSH keys, and set up the domain.
1. Create an application on the server (use the same name as your local project if you like).

```bash
# Run on the Dokku host
dokku apps:create ruby-getting-started
```

2. Install the dokku-postgres plugin (feel free to rename `railsdatabase`).

```bash
# Run on the Dokku host
# Install the Postgres plugin
# Plugin installation requires root, hence sudo
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git0

# Create a Postgres service named railsdatabase
dokku postgres:create railsdatabase
```

3. Link the application with the database so it receives the connection URL.

`postgres:link` essentially does this:

```bash
# Run on the Dokku host
# Every official datastore offers a `link` command to attach a service to an app
dokku postgres:link railsdatabase ruby-getting-started
```

4. Append the Herokuish buildpacks.

```bash
echo "https://github.com/HashNuke/heroku-buildpack-elixir.git" >> .buildpacks
# The gigalixir buildpack fixes https://github.com/gjaldon/heroku-buildpack-phoenix-static/issues/127
echo "https://github.com/gigalixir/gigalixir-buildpack-phoenix-static.git" >> .buildpacks
```

4. Configure the buildpacks.

```bash
# Create elixir_buildpack.config at the project root
touch elixir_buildpack.config
```

Add the following content.

Adjust `config_vars_to_export` for your own project. This is my setup:

```txt
# The version string must contain a dot and match the list entry exactly.
# I once used 27 and it failed.
erlang_version=27.0

# Elixir version
elixir_version=1.18.1

# Always rebuild from scratch on every deploy?
always_rebuild=true

# Export heroku config vars
config_vars_to_export=(DATABASE_URL GMAIL_PASSWORD GMAIL_USERNAME SECRET_KEY_BASE)
```

```bash
# Create phoenix_static_buildpack.config at the project root
touch phoenix_static_buildpack.config
```

Then add:

```txt
# Specify a recent Node version; otherwise you’ll get a very old one and the build fails.
node_version=23.11.0
npm_version=10.9.2
```

5. Downgrade the Dokku Heroku stack.  
   The buildpack we added—[HashNuke/heroku-buildpack-elixir](https://github.com/HashNuke/heroku-buildpack-elixir?tab=readme-ov-file#version-support)—does not support the latest stack, `heroku-24`. Dokku uses `heroku-24` by default, so deploys will fail with an error like this:

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

To fix it, [downgrade the stack to `heroku-22`](https://stackoverflow.com/questions/76957214/dokku-how-to-change-heroku-stack-to-heroku-20-or-heroku-22-from-heroku-18). The buildpack supports that stack, and it includes [recent Erlang builds](https://builds.hex.pm/builds/otp/ubuntu-22.04/builds.txt).

```bash
# Check which stack your app is using
# Replace my-app with your application name
dokku buildpacks:report my-app
# Switch to heroku-22
dokku buildpacks:set-property my-app stack gliderlabs/herokuish:latest-22
```

6. Preconfigure environment variables.

Run this on the server:

```bash
# You can set multiple variables at once.
# Each change restarts the app. It’s fine if the app isn’t deployed yet.
dokku config:set my-app ENV=prod COMPILE_ASSETS=1
```

7. Run database migrations automatically.

My Phoenix project needs database migrations. I wanted them to run after every deploy, so I created an `app.json` at the project root.

```elixir
{
  "scripts": {
    "dokku": {
      "postdeploy": "mix ecto.migrate"
    }
  }
}
```

Dokku provides other deployment hooks—check the [documentation](https://dokku.com/docs/appendices/file-formats/app-json/?h=postde#scripts) for details.

8. Add a new Git remote.

```bash
git remote add dokku dokku@<mydomain.com>:<app_name>
```

Upload your Git public key to the Dokku server.

If you already use SSH for GitHub, your key is usually at `~/.ssh/id_rsa.pub`. Append its contents to the server’s `~/.ssh/authorized_keys`.

9. Deploy the app.

From now on, deployments are as simple as running:

```bash
git push dokku main
```

If your repository lives on GitHub, remember to push there as well:

```bash
git push
```

---

**Other notes**

1. My domain is managed on Cloudflare, so I rely on their proxy for HTTPS. I didn’t enable Dokku’s HTTPS features.
