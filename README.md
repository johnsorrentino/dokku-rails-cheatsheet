# Dokku Rails Cheatsheet

This is a cheatsheet for deploying a Rails on a Digital Ocean droplet using Dokku. This deployment also includes Sidekiq processes, multiple environments, and a custom domain.

- [Local Setup](#Local-Setup)
- [Server Setup](#Server-Setup)
- [Dokku Setup](#Dokku-Setup)
  - [Dokku Plugins](#Dokku-Plugins)
  - [Dokku Services](#Dokku-Services)
  - [Dokku Domain](#Dokku-Domain)
- [Deployment Setup](#Deployment-Setup)
  - [Set Remote Repos](#Set-Remote-Repos)
  - [Set Default Branches](#Set-Default-Branches)
  - [Set Environment Variables](#Set-Environment-Variables)
- [Useful Commands](#Useful-Commands)

## Local Setup

1. Setup your Rails application.
2. Create a Procfile in the root of your Rails repos.
   ```
   web: bundle exec puma -C config/puma.rb
   worker: bundle exec sidekiq -e production -C config/sidekiq.yml
   ```
3. Create a post deploy hook in the rook of your Rails repo. This will automatically run db:migrate after deployment.
   ```
   {
     "name": "my_rails_app",
     "description": "My Rails App",
     "keywords": [],
     "scripts": {
       "dokku": {
         "postdeploy": "bundle exec rake db:migrate"
       }
     }
   }
   ```
4. Install the Dokku CLI for MacOS - `brew install dokku/repo/dokku`

## Server Setup

1. Create a Digital Ocean droplet from the official Dokku image on the marketplace.
   - You will need at least 2GB of memory for `bundle install` to avoid swap.
2. After the server initializes immediately go to the IP address and set the admin public key.
3. Add a domain to the Digital Ocean project. Set the name servers and an A record pointing to the droplet.

## Dokku Setup

These commands are run while you are SSH'd into the droplet.

### Dokku Plugins

Install the Postgres, Redis, and Let's Encrypt plugins.

```
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
sudo dokku plugin:install https://github.com/dokku/dokku-redis.git
sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
```

### Dokku Services

Create the Dokku services for the Rails application, Redis, and Postgres.

```
dokku apps:create my_rails_app
dokku postgres:create my_rails_app-postgres
dokku postgres:link my_rails_app-postgres my_rails_app
dokku redis:create my_rails_app-redis
dokku redis:link my_rails_app-redis my_rails_app
```

### Dokku Domain

```
dokku domains:clear-global
dokku domains:set my_rails_app my-custom-domain.com
dokku config:set --no-restart my_rails_app DOKKU_LETSENCRYPT_EMAIL=admin@example.com
dokku letsencrypt my_rails_app
dokku letsencrypt:cron-job --add
```

## Deployment Setup

Setup the remote staging and production repos on the Dokku server. There are two separate Dokku deployments on Digital Ocean with separate IP addresses.

### Set Remote Repos

```
git remote add staging dokku@<STAGING_IP_ADDRESS>:my_rails_app
git remote add production dokku@<PRODUCTION_IP_ADDRESS>:my_rails_app
```

### Set Default Branches

We set the default branch for staging to `develop` and the default branch for production to `main`.

```
dokku --remote staging git:set deploy-branch develop
dokku --remote production git:set deploy-branch main
```

### Set Environment Variables

Set your environment variables. Note that we're using the `--remote staging` and `--remote production` flags which choose which Dokku deployment to run the commands against.

```
dokku --remote staging config:set RAILS_MASTER_KEY=`cat config/credentials/production.key`
dokku --remote production config:set RAILS_MASTER_KEY=`cat config/credentials/production.key`
```

## Useful Commands

Run the Rails console remotely.

```
dokku --remote <environment> run rails c
```

Scale your workers.

```
dokku --remote <environment> ps:scale worker=2
```

Stop your workers.

```
dokku --remote <environment> ps:stop
```

Deploy your application.

```
git push staging develop
git push production main
```

Reset your database.

```
dokku --remote <environment> run bundle exec rake db:reset
```
