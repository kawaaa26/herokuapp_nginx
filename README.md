# README

Simple CRUD application for deploying to heroku with Nginx and Unicorn.

The application is just created by using scaffold.

## Versions

* Ruby 2.6.3
* Ruby on Rails 5.2.3
* psql (PostgreSQL) 11.3

## Deploy Settings

* Create a heroku app
```sh
$ heroku create
```

* Ruby buildpack settings
```sh
$ heroku bildpacks:add heroku/ruby
```

### Nginx Settings

* Nginx buildpack settings
```sh
$ heroku buildpacks:add https://github.com/heroku/heroku-buildpack-nginx
```

* Create an nginx configuration file

`config/nginx.conf`
```
# Heroku dynos have at least 4 cores.
worker_processes <%= ENV['NGINX_WORKERS'] || 4 %>;

events {
  use epoll;
  accept_mutex on;
  worker_connections 1024;
}

http {
  gzip on;
  gzip_comp_level 3;
  gzip_min_length 150;
  gzip_proxied any;
  gzip_types text/plain text/css text/json text/javascript
    application/javascript application/x-javascript application/json
    application/rss+xml application/vnd.ms-fontobject application/x-font-ttf
    application/xml font/opentype image/svg+xml text/xml;

  server_tokens off;

  log_format l2met 'measure#nginx.service=$request_time request_id=$http_x_request_id';
  access_log logs/nginx/access.log l2met;
  error_log logs/nginx/error.log;

  include mime.types;
  default_type application/octet-stream;
  sendfile on;

  # Must read the body in 5 seconds.
  client_body_timeout 5;

  upstream app_server {
    server unix:/tmp/nginx.socket fail_timeout=0;
  }

  server {
    listen <%= ENV["PORT"] %>;
    server_name _;
    keepalive_timeout 5;

    root /app/public; # path to your app

    location / {
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect off;
      proxy_pass http://app_server;
    }

    location ~* ^/documentation/(.*) {
      set $s3_bucket        'my_bucket.s3-website-us-east-1.amazonaws.com';
      set $url_full         '$1';
      resolver              8.8.8.8 valid=300s;
      resolver_timeout      10s;

      index index.html;

      proxy_hide_header       x-amz-id-2;
      proxy_hide_header       x-amz-request-id;
      proxy_hide_header       Set-Cookie;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        Host $s3_bucket;
      proxy_ignore_headers    "Set-Cookie";
      proxy_buffering         off;
      proxy_intercept_errors  on;
      proxy_redirect          off;
      proxy_pass              http://$s3_bucket/$url_full;
    }

    location ~* \.(eot|oft|svg|ttf|woff)$ {
      add_header Access-Control-Allow-Origin *;
      expires max;
      log_not_found off;
      access_log off;
      add_header Cache-Control public;
    }

    location ~* ^/assets/ {
      gzip_static on;

      # Per RFC2616 - 1 year maximum expiry
      expires 1y;
      add_header Cache-Control public;

      # Some browsers still send conditional-GET requests if there's a
      # Last-Modified header or an ETag header even if they haven't
      # reached the expiry date sent in the Expires header.
      add_header Last-Modified "";
      add_header ETag "";
      break;
    }
  }
}

```

## Unicorn Settings

* remove puma and add unicorn to the Gemfile

`Gemfile`
```rb
# gem 'puma', '~> 3.11' # remove or comment

gem 'unicorn' # Add unicorn
```

* run `$ bundle` to update the `Gemfile.lock`

* remove `config/puma.rb` Due to using `unicorn.rb` instead.

* create an Unicorn configuration file

`config/unicorn.rb`
```rb
worker_processes Integer(ENV["WEB_CONCURRENCY"] || 3)
timeout 15
preload_app true
worker_processes 4
listen 'unix:///tmp/nginx.socket', backlog: 1024

before_fork do |server,worker|
    FileUtils.touch('/tmp/app-initialized')
end
```

* create a Procfile to call `config/unicorn.rb`

`Procfile`
```rb
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb

# web: bin/start-nginx bundle exec puma -c config/puma.rb # If using puma instead.
```

## Deploy to Heroku

* run the commands below to deploy

```sh
$ git add -A

$ git cz

$ git push heroku master
```

* create a DB onto the Heroku App

```sh
$ heroku run rake db:migrate RAILS_ENV=production
```

The deploy is done. check if the app works properly on heroku.

* run the command below to check Nginx on the App.

```sh
$ wget -S --spider <HEROKU_APP_URL>
```

The command will perhaps return like...
```sh
Spider mode enabled. Check if remote file exists.
--2019-07-14 17:29:09-- <HEROKU_APP_URL>
Resolving  <HEROKU_APP_URL>...
Connecting to  <HEROKU_APP_URL>|... connected.
HTTP request sent, awaiting response...
  HTTP/1.1 200 OK
  Connection: keep-alive
  Server: nginx
  Date: Sun, 14 Jul 2019 08:29:09 GMT
  Content-Type: text/html; charset=utf-8
  X-Frame-Options: SAMEORIGIN
  X-Xss-Protection: 1; mode=block
  X-Content-Type-Options: nosniff
  X-Download-Options: noopen
  X-Permitted-Cross-Domain-Policies: none
  Referrer-Policy: strict-origin-when-cross-origin
  Cache-Control: max-age=0, private, must-revalidate
  X-Runtime: 0.090119
  Via: 1.1 vegur
Length: unspecified [text/html]
Remote file exists and could contain further links,
but recursion is disabled -- not retrieving.
```

* if the respnse says `Server: nginx`, the application successfully works on the Nginx Server!

* if the command `wget` is unrecognized, try installing `wget` using brew.

<br>
