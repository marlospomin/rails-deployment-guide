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

For extra help configuring Mina, head over to the documentation page at https://github.com/mina-deploy/mina.

## Deploy

After the configuration process is done run `mina setup` to setup the folder structure on your server.

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
