---
layout: post
title:  "Rails template"
date:   2016-01-29 13:34:00
comments: false
categories: rails
tags:
- rails
---
При создании нового Rails app приходится указывать из раза в раз одни и теже настройки дополнительные. Хватит это терпеть!

Пора уже собрать все в один [rails template][gist]:
 
{% highlight ruby %}
file 'Procfile.dev', <<-CODE
web: bin/puma -b unix://./tmp/sockets/puma.socket  -e development  
CODE

file 'bin/dev', <<-CODE
#!/bin/sh  
bin/bundle exec foreman start --procfile=./Procfile.dev
CODE

run 'rm .gitignore'
file '.gitignore', <<-CODE
/coverage
/.localeapp
/.bundle
/.idea
/example
/public/uploads
/db/*.sqlite3
/db/*.sqlite3-journal
/log/*.log
/log
/tmp
/spec/tmp
/config/application.yml
/config/database.yml
/config/omniauth.yml
/.bundle
/vendor/bundle
# unless supporting rvm < 1.11.0 or doing something fancy, ignore this:
.rvmrc
# if using bower-rails ignore default bower_components path bower.json files
/vendor/assets/bower_components
*.bowerrc
bower.json
# Ignore pow environment settings
.powenv
# Ignore Byebug command history file.
.byebug_history
config/initializers/secret_token.rb
config/secrets.yml
CODE

run "chmod +x bin/dev"
run 'rm README.rdoc'

gem 'hamlit'
gem 'russian'
gem 'newrelic_rpm'

gem_group :development, :test do
  gem "rspec-rails"
  
  gem 'pry'
  gem 'pry-rails'
  gem 'pry-byebug'
  
  gem 'quiet_assets'
  gem 'annotate'
end

gem_group :development do
  gem 'capistrano', '~> 3.2.0'
  gem 'capistrano-rails', '~> 1.1'
  gem 'capistrano-rails-collection'
  gem 'capistrano-rails-console'

  gem 'haml-rails'
  
  gem 'capistrano3-puma', github: 'seuros/capistrano-puma'
  
  gem 'foreman'
end

gem_group :production do
  gem 'puma'
end

after_bundle do
  git :init
  git add: "."
  git commit: %Q{ -m 'Initial commit' }
end
{% endhighlight %}


[gist]: https://gist.github.com/fuCtor/9bddaca5f4999cb5c21c