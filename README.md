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

## Nginx
## Puma
## Mina

After installing mina's gem add a new file under `/config` folder called `deploy.rb`, in that file add the following lines:

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

Run `mina deploy` to deploy.

Done.

## Troubleshooting

If your server assets are not displaying properly log into your server using `ssh`, grep puma's master process id using `ps aux` and kill it `kill -9 PID`, then `cd /var/www/your_app_name` and re launch puma with `rails s`.

Pro Tip: You can also enable puma's PID to output into a file so you can quickly restart your server with a custom mina command/launch task.

## Contributing

If you wish to contribute please make a pull request or open an issue.

## License

Code released under the [MIT](LICENSE) license.
