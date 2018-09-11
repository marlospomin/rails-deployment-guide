# Rails Deployment Guide

This guide covers how to deploy a Rails application using Puma, Nginx and Mina.

## Getting Started

Assuming you have a Rails >= 5 app, this is how you would configure the files in order to deploy it.

### Requirements

The following gems are required in order to deploy the app, in a `Gemfile` file add the following lines:

``` ruby
gem "puma", "~> 3.11"

group :development do
  gem "mina"
end
```

Run `bundle` to install it.

Nginx is also required, to install it use `apt install nginx` on your server if it's not installed already.

Pro Tip: To check if nginx is installed type `which nginx` or `nginx -v`.

Create your app directory: `sudo mkdir /var/www/your_app_name` and make sure to `chown` it to your non-root user. (`chown your_unix_username:your_unix_username -R /path/to/folder`)

## Nginx

First, remove the default symlink under `rm /etc/nginx/sites-enabled/default` and create a new file with your site name under `/etc/nginx/sites-available`.

Under `/etc/nginx/sites-available/your-site.com` add these lines:

``` conf
upstream your_app_name {
  server unix:///var/www/your_app_name/shared/sockets/puma.sock fail_timeout=0;
}

server {
  listen 80;
  listen [::]:80;

  server_name your-site.com;
  root /var/www/your_app_name/current/public;

  location @your_app_name {
    proxy_pass http://your-site;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
  }

  location ~ ^(assets|packs)/ {
    gzip_static on;
    expires max;
    access_log off;
    add_header Cache-Control public;
    break;
  }

  try_files $uri/index.html $uri @puma;

  location ~ ^/(500|404|422).html {
    root /var/www/your_app_name/current/public;
  }

  error_page 500 522 404 /500.html;
  client_max_body_size 10m;
}
```

Now update your `nginx.conf` file under `/etc/nginx` with:

``` conf
user www-data;

worker_processes auto;

pid /run/nginx.pid;

include /etc/nginx/modules-enabled/*.conf;

events {
  worker_connections 1024;
}

http {
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 15;

  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] '
                  '"$request" $status  $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 2;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

Now symlink your config into the enabled sites `sudo ln -s /etc/nginx/sites-available/your-site.com /etc/nginx/sites-enabled/your-site.com` and restart `nginx` with `sudo service nginx restart`.

Off-Topic: Yes I know I gotta clean up this config.

## Puma

Under `/config/puma.rb` update the file contents to:

``` ruby
# Specifies the amount of thread to power the server.
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# Specifies the `environment` that Puma will run in.
environment ENV.fetch("RAILS_ENV") { "development" }

# Specifies the number of `workers` to boot in clustered mode.
workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Use `preload_app!` method when specifying a `workers` number.
preload_app!

# Daemonize the server to run in the background in production.
daemonize ENV.fetch("PUMA_DAEMONIZE") { false }

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart
```

Note: Once you run `mina setup` it will create a new shared copy of this file under `/var/www/your_app_name/shared/config`, you will have to fill that file manually.

## Mina

After installing mina add a new file under `/config` folder called `deploy.rb`, in that file add the following lines:

``` ruby
require "mina/rails"
require "mina/git"

# If you use rvm/rbenv check the documentation
# for more info on how to configure it.

set :application_name, "your_app_name"
set :domain, "your-site.com"
set :deploy_to, "/var/www/your_app_name"
set :repository, "git@github.com:your-name/your-repo.git"
set :branch, "master"
set :user, "your_unix_username"

set :shared_files, fetch(:shared_files, []).push(
  "config/master.key", "config/puma.rb")

# If you use Active Storage uncomment the line below:
# set :shared_dirs, fetch(:shared_dirs, []).push("storage")

task :setup do
  command %{touch "#{fetch(:shared_path)}/config/master.key"}
  command %{touch "#{fetch(:shared_path)}/config/puma.rb"}
  command %{mkdir -p "#{fetch(:shared_path)}/sockets"}
end

# If you are using an older version of Rails remove `master.key`
# entry from this file and replace it with `secrets.yml`.

desc "Deploys the current version to the server."

task :deploy do
  invoke :"git:ensure_pushed"

  deploy do
    invoke :"git:clone"
    invoke :"deploy:link_shared_paths"
    invoke :"bundle:install"
    # If you use `yarn` uncomment the line below:
    # command %{yarn install}
    invoke :"rails:db_migrate"
    invoke :"rails:assets_precompile"
    invoke :"deploy:cleanup"

    on :launch do
      in_path(fetch(:current_path)) do
        command %{rails restart}
      end
    end
  end
end
```

For extra help on configuring Mina, head over to the documentation page at https://github.com/mina-deploy/mina.

## Deploy

After the base configuration is done run `mina setup` to setup the folder structure on your server.

Log in your server via `ssh` and the fill the shared files under `/var/www/your_app_name/shared/config` with your local file contents. This process is a one time only, or until you run `mina setup` again.

Run `mina deploy` to deploy.

Done.

## Troubleshooting

If your server assets are not displaying properly log into your server using `ssh`, grep puma's master process id using `ps aux` and kill it `kill -9 PID`, then `cd /var/www/your_app_name` and re launch puma with `rails s`.

Pro Tip: You can also enable puma's PID to output into a file so you can quickly restart your server with a custom mina command/launch task.

## Contributing

If you wish to contribute please make a pull request or open an issue.

## License

Code released under the [MIT](LICENSE) license.
