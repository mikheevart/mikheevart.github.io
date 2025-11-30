---
title: Rails 7 - легкие пути инсталляции Bootstrap 5
tags: [RUBY ON RAILS, RUBY]
style: fill
color: danger
description: Rails 7 - легкие пути инсталляции Bootstrap 5 без esbuild, node, yarn
---
### Первый способ

```
$ rails new my_app
$ cd my_app
```
Проверяем, включен ли importmaps в ваш проект:
```
$ cat config/importmap.rb
```
, если нет:
```
$ rails importmap:install
$ bin/importmap pin bootstrap
```
, после чего можно снова взглянуть:
```
$ cat config/importmap.rb
```
Добавим в app/javascript/application.js:
```
import 'bootstrap'
```
Добавим в Gemfile:
```
gem 'bootstrap', '~> 5.1.3'
```
Добавим в app/assets/stylesheets/application.css:
```
@import "bootstrap";
```
и переименуем в app/assets/stylesheets/application.scss
```
$ bundle install
$ rails s
```
### Второй способ

```
$ rails new my_app
$ cd my_app
```
Добавим в Gemfile
```
gem 'bootstrap', '~> 5.1.3'
gem "sassc-rails"
```
Добавим в app/assets/stylesheets/application.css
```
@import "bootstrap";
```
и переименуем в app/assets/stylesheets/application.scss
Добавим в config/initializers/assets.rb
```
Rails.application.config.assets.precompile += %w( bootstrap.min.js popper.js )
```
Далее:
```
rails assets:precompile
```
Добавим в config/importmap.rb:
```
pin "popper", to: 'popper.js', preload: true
pin "bootstrap", to: 'bootstrap.min.js', preload: true
```
Добавим в app/javascript/application.js:
```
import "popper"
import "bootstrap"
```
Далее:
```
$ bundle install
$ rails s
```
