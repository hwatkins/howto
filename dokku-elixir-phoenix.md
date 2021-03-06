# Elixir and Phoenix on Dokku on Digital Ocean

Create a new [Vultr](https://www.vultr.com/) box:

* Ubuntu 18.04 x64
* $5 per month / 1GB / 1 CPU
* Atlanta location (or what ever is closest)
* $2 per month Auto Backups (do not choose IPv6)
* Add SSH key

![](https://rpl.cat/uploads/LMEqcxZ0H2HhEfcdK2ojJKBe77PzGv7hD37FhvxkrH4/public.png)

Choose a hostname (if you want to send email you should make it match your domain as an FQDN: `yourappname.com`)

Launch the server. It is fast. Open an SSH session (you'll see the new IP when you launch the server):

    ssh root@111.222.333.444
    
Install the first five minutes security (https://github.com/jeffrafter/server):

    apt-get install git-core
    
Then:

    cd ${HOME}
    git clone git://github.com/jeffrafter/server.git server    
    cd ${HOME}/server
    cp scripts/env.sh.example scripts/env.sh
    nano scripts/env.sh

Run it:

    scripts/setup.sh
    
### Setting up dokku    
    
Open a new SSH session as the `deploy` user:

    ssh deploy@111.222.333.444
        
Next install dokku (http://dokku.viewdocs.io/dokku/getting-started/installation/) check version here (http://dokku.viewdocs.io/dokku/)

    wget https://raw.githubusercontent.com/dokku/dokku/v0.7.2/bootstrap.sh
    sudo DOKKU_TAG=v0.7.2 bash bootstrap.sh

This takes about 5 minutes. Once complete open a browser pointed at your IP. Change the hostname to match your hostname and create.

Next run these dokku commands (still SSH as `deploy`):
    
    dokku apps:create yourappname
    sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
    dokku postgres:create yourappname-database
    dokku postgres:link yourappname-database yourappname   
        
## Phoenix setup

To deploy Phoenix to `dokku` you need three things:

* Elixir buildpacks
* A deployable `prod.secret.exs`
* A new `SECRET_KEY`

### Buildpacks

Create a new file in your repo called `.buildpacks`:

    https://github.com/HashNuke/heroku-buildpack-elixir.git
    https://github.com/gjaldon/heroku-buildpack-phoenix-static.git

Also create `elixir_buildpack.config`:

    elixir_version=1.9.4
    erlang_version=22.2.1

And `phoenix_static_buildpack.config`:

    phoenix_relative_path=.
    node_version=10.16.3
    npm_version=6.12.0
    yarn_version=1.19.1

    clean_cache=false
    
> **Note**: you do not need the phoenix static buildpack if you do not have assets (i.e., this is a REST API or you compiled your assets elsewhere).

And `app.json`:

    {
      "scripts": {
        "dokku": {
          "postdeploy": "mix ecto.migrate"
        }
      }
    }
    
### Safe `prod.exs`    

```elixir
config :textmewhen, Textmewhen.Endpoint,
  http: [port: {:system, "PORT"}],
  url: [host: "example.com", port: 80],
  cache_static_manifest: "priv/static/manifest.json",
  secret_key_base: System.get_env("SECRET_KEY_BASE")

# Do not print debug messages in production
config :logger, level: :info

# Configure your database
config :textmewhen, Textmewhen.Repo,
  adapter: Ecto.Adapters.Postgres,
  url: System.get_env("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "20"),
  ssl: true
```

Delete the line: `import_config "prod.secret.exs"`

Remove the old file

    rm config/prod.secret.exs

Procfile:

    web: mix phoenix.server

From your Vultr VM:  
  
    dokku config:set SECRET_KEY_BASE="`mix phoenix.gen.secret`"
    dokku config:set MIX_ENV=prod
    dokku config:set PORT=5000
    dokku domains:add yourappname domainname
    

## SSL 

Via ssh:

    sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git

From your app:

```
dokku domains
dokku domains:add yourserver.com
dokku domains:add api.yourserver.com
dokku domains:add click.yourserver.com
dokku letsencrypt:cron-job --add
dokku config:set --no-restart DOKKU_LETSENCRYPT_EMAIL=your@email.tld
dokku letsencrypt
```



...

# Resources

https://github.com/phoenixframework/phoenix_guides/blob/master/deployment/E_heroku.md
https://dev.to/mplatts/deploying-phoenix-via-dokku-1gip
http://www.cspags.com/deploying-elixir-phoenix-to-heroku/
http://philippkueng.ch/deploy-a-phoenix-application-on-heroku.html
