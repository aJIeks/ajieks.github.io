---
layout: post
title:  "JRuby + Ratpack"
date:   2016-10-20 00:00:00
comments: false
categories: jekyll update
tags:
- jruby
---

JRuby + Ratpack = ❤️
==

Многие разработчики на Ruby знают как обстоят дела с асинхронным выполнением кода на на имеющихся серверах.
Либо вы используете что-то на EventMachine, либо колдуете с Ruby::Concurrent, Celuloid.
В любом случае время это работает не сильно эффективно из-за GIL (ждем, надеемся и верим в Ruby 3).
Но есть реализации свободные от этой проблемы, одна из них реализация поверх JVM - JRuby.
Много расписывать про него не буду, думаю все как минимум слышали про него.
Главной особенностью данной реализации является легкая интеграция с любой библиотекой на JVM.
Это открывает нам большой простор в выборе библиотек и готовых инструментов.

Так в мире Java есть библиотека, избавляющая нас от использования стандартной конкурентной Java модели на Executor, реализуя ее на Акторах.
Звать библиотеку эту Netty. Позже на ее основе были разработаны других, например Ratpack.

Ratpack асинхронный веб сервер, под капотом находится Netty, следовательно достаточно эффективно работает с подключения и в целом с IO.
Имеет в себе все необходимое для построения производительного сервиса.

Поэтому используя возможности Ratpack и гибкость и простоту Ruby (JRuby), сделаем простейший сервис, который будет разворачивать нам короткие ссылки.
В интернете есть ряд примеров, но они заканчиваются на то как запустить все это и получить простой ответ.
Есть еще пример с подключением метрик (способ из документации абсолютно не пригоден для JRuby).

В данном примере рассмотрим:

- подключение jar файлов
- создание сервера
- подключение метрик
- асинхронное выполнение запросов к внешним ресурсам
- тестирование нашего сервиса
- и зальем все это на heroku

Подключение JAR файлов
--
Каждый Ruby программист пользуется bundler, жизнь без него была грустна и полна плясок, грабель и других приключений.
В мире Java есть различные сборщики, которые подтянул указанные зависимости и соберут приложением, но это не Ruby way.
Так появился jbundler. Выполняет тужа самую функцию что и bundler, но для java библиотек. После чего при загрузке они становятся доступны из JRuby. Красота!

И так, нам нужно подключить ratpack к нашему приложению. Нам достаточно будет только core. Остальное мы не используем.

__Gemfile:__
{% highlight ruby %}
source 'https://rubygems.org'

ruby '2.3.0', :engine => 'jruby', :engine_version => '9.1.2.0'

gem 'rake'
gem 'activesupport', '4.2.5'
gem 'jruby-openssl'

gem 'jbundler', '0.9.2'
gem 'jrjackson'

group :test, :development do
  gem 'pry'
end

group :test do
  gem 'rspec'
  gem 'simplecov', require: false
end
{% endhighlight %}

__Jarfile:__
{% highlight ruby %}
jar 'io.ratpack:ratpack-core', '1.4.2'
jar 'org.slf4j:slf4j-simple', '1.7.10'
{% endhighlight %}

В консоли выполняем
{% highlight bash %}
bundle install
bundle exec jbundle install
{% endhighlight %}

В дальнейшем добавим еще пару библиотек, но пока остановимся на этом.

Создание сервера
==
Загрузив все зависимости создаем базовый сервер, проверим что все работает. Так как у нас нет rack, то маршрутизаицю будем будем писать используя штатные средства.

Для начала импортируем необходимые Java классы.
{% highlight ruby %}
require 'java'

java_import 'ratpack.server.RatpackServer'
java_import 'ratpack.server.ServerConfig'
java_import 'java.net.InetAddress'
{% endhighlight %}

И объявим наш класс сервера:
{% highlight ruby %}
module UrlExpander
  Server = Struct.new(:host, :port) do
    def self.run
      new('0.0.0.0', ENV['PORT'] || 3000).tap(&:run)
    end

    def run
      @server  = RatpackServer.of do |s|
        s.serverConfig(config)
        s.handlers do |chain|
          chain.get  'status', Handler::Status
          chain.all            Handler::Default
        end
      end
      @server.start
    end

    def shutdown
      @server.stop
    end

    private

    def config
      ServerConfig.embedded
                  .port(port.to_i)
                  .address(InetAddress.getByName(host))
                  .development(ENV['RACK_ENV'] == 'development')
                  .base_dir(BaseDir.find)
                  .props("application.properties")
    end
  end
end
{% endhighlight %}

Для нашего сервиса создали endpoint: status.
Первый позволяет нам проверить жив ли наш сервер в принципе, а второй будет выполнять основную задачу, разворачивания ссылок.
Метод handlers, принимает блок, в которой передается интерфейс [Chain](https://ratpack.io/manual/current/api/ratpack/handling/Chain.html),
который позволяет определить маршрутизацию. Для объявления status испольуем метод get, эквивалентный HTTP методу.
Вторым аргументом передается объект реализующий интерфейс [Handler](https://ratpack.io/manual/current/api/ratpack/handling/Handler.html).
В нашем случае это модуль, в котором объявлен метод handle, принимающий текущий контекст.
Как видите все достаточно просто и понятно. Никаких трехэтажных фабрик, или чего-то подобного.

Собственно сам обработчик достаточно простой:
{% highlight ruby %}
module UrlExpander
  module Handler
    class Status
      def self.handle(ctx)
        ctx.render 'OK'
      end
    end
  end
end
{% endhighlight %}

Подключение метрик
==
Отслеживать статус нашего сервиса мы теперь можем, но хорошо бы еще узнать что с ним внутри, время ответа, количество запросов и другие показатели.
Для этого нам нужны метрики. Ratpack имеет интеграцию с Dropwizard, для этого нужно добавить в наш Jarfile пару пакетов и установить их
{% highlight ruby %}
jar 'io.ratpack:ratpack-guice', '1.4.2'
jar 'io.ratpack:ratpack-dropwizard-metrics', '1.4.2'
{% endhighlight %}

Далее подключим его к нашему серверу, выполняется это достаточно просто, достаточно лишь модифицировать несколько участков.
{% highlight ruby %}
java_import 'ratpack.guice.Guice'
java_import 'ratpack.dropwizard.metrics.DropwizardMetricsConfig'
java_import 'ratpack.dropwizard.metrics.DropwizardMetricsModule'
java_import 'ratpack.dropwizard.metrics.MetricsWebsocketBroadcastHandler'
{% endhighlight %}
Зарегистрируем модуль в нашем Registry:
{% highlight ruby %}
        s.serverConfig(config)

        s.registry(Guice.registry { |g|
                     g.module(DropwizardMetricsModule.new)
                   })
{% endhighlight %}

И загрузим его конфигурацию:
{% highlight ruby %}
    def config
      ServerConfig.embedded
                  .port(port.to_i)
                  .address(InetAddress.getByName(host))
                  .development(ENV['RACK_ENV'] == 'development')
                  .base_dir(BaseDir.find)
                  .props("application.properties")
                  .require("/metrics", DropwizardMetricsConfig.java_class)
    end
{% endhighlight %}

А еще мы хотим получать наши метрики через WebSocket, добавим handler для этого:
{% highlight ruby %}
        s.handlers do |chain|
          chain.get  'status', Handler::Status
          chain.get 'metrics-report', MetricsWebsocketBroadcastHandler.new
          chain.all  Handler::Default
        end
{% endhighlight %}

Готово, так же можно подключить выгрузку метрик в консоль либо в StatsD. Позднее добавим к этому небольшой интерфейс.

Асинхронное выполнение запросов к внешним ресурсам
==
Сервер у нас заводится, слушает указанный порт и отвечает на запросы. Далее добавим endpoint, который будет возвращать нам все url,
через которые проходит наша короткая ссылка. алгоритм будет простейший, будем на каждом redirect сохранять новый Location в массив, после чего вернем его.
{% highlight ruby %}
s.handlers do |chain|
  chain.get 'status',  Handler::Status
  chain.path 'expand', Handler::Expander
  chain.all            Handler::Default
end
{% endhighlight %}
Добавленный endpoint будет принимать как POST, так и GET запросы.

Если бы у нас был только блокирующий API, каждый запрос обрабатывался в своем потоке, ну как обрабатывался, 90% времени он бы ждал ответа от сервера,
т.к. полезных вычислений у нас минимум. Но к нашему счастью, Ratpack является асинхронным сервером и предоставляет полный набор компонентов,
в том числе асинхронный http клиент и Promise, какая асинхронщина без них.
И так, создадим для каждой исходной ссылки Promise, который при удачном завершении вернет нам массив Location.
Внутри же, запускаем get по нашему URL и вешаем callback на получение нового location от сервера.
Таким образом вы поместим в наш массив целевой URL  и все промежуточные.

{% highlight ruby %}
module UrlExpander
  module Handler
    class Expander < Base
      java_import 'ratpack.exec.util.ParallelBatch'
      java_import 'ratpack.http.client.HttpClient'

      def execute
        data = request.present? ? JrJackson::Json.load(request) : {}
{% endhighlight %}
Создаем HttpClient, которым будем собирать наши ссылки
{% highlight ruby %}
        httpClient = ctx.get HttpClient.java_class
{% endhighlight %}
Собираем все URL, переданные нам и если ничего нет, то сразу возвращаем Promise с пустой мапой.
{% highlight ruby %}
        urls = [*data['urls'], *data['url'], *params['url']].compact
        unless urls.present?
          return Promise.value({})
        end
{% endhighlight %}
Создаем параллельные запросы по всем переданным ссылкам:
{% highlight ruby %}
        tasks = urls.map do |url|
          Promise.async do |down|
            uri = Java.java.net.URI.new(url)
            locations = [url]
            httpClient.get(uri) do |spec|
              spec.onRedirect do |resposne, action|
                locations << resposne.getHeaders.get('Location')
                action
              end
            end .then do |_resp|
              down.success(locations);
            end
          end
        end
{% endhighlight %}
Дождавшись их выполнения собираем результат и возвращаем его:
{% highlight ruby %}
        ParallelBatch.of(tasks).yieldAll.flatMap do |results|
          response = results.each_with_object({}) do |result, locations|
            result.value.try do |list|
              locations[list.first] = list[1..-1]
            end
          end

          Promise.value response
        end
      end
    end
  end
end
{% endhighlight %}

В итоге получили цепочку из Promise, которая асинхронно выполнит наш код.

Тестирование нашего сервиса
==