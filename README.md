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

Nginx is also required, to install it use `apt install nginx` on your server if it's not installed already.

Pro Tip: To check if nginx is installed type `which nginx` or `nginx -v`.

## Nginx
## Puma
## Mina

## Contributing

If you wish to contribute please make a pull request or open an issue.

## License

Code released under the [MIT](LICENSE) license.
