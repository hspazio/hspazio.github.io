---
layout: post
title: "Introducing ElasticNotifier"
date: 2017-11-28
tags: [ruby, elasticsearch, rails]
comments: true
---

[ElasticNotifier][elastic_notifier_github] is a gem that provides a simple API to send error notifications to an [ElasticSearch](https://www.elastic.co/) instance.

As you rescue errors in your application you can send them to an ElasticSearch index to be used later for analytics, reports and dashboards.

Alternatively, ElasticNotifier is also compatible to [exception_notification](https://github.com/smartinez87/exception_notification) gem as Notifier plug-in.
ExceptionNotification is a Rack middleware that intercepts any unhandled errors from a Rails (Sinatra or any other Rack-based applications) and sends sends notifications using various configurable notifiers.

### Getting started

Add the gem to your application's `Gemfile` and run `bundle` command.

Configure the notifier:

```ruby
NOTIFIER = ElasticNotifier.new(
  url: "http://myserver.com:9200", # default is http://localhost:9200
  index: "my_custom_index",        # default is :elastic_notifier
  type: "my_document_type"         # default is :signals
)
```

For __Rails__ applications you can add the code above to `config/initializers/elastic_notifier.rb` so it will be available throghout the app.

Then send error notifications as you rescue errors:

```ruby
begin
  # some code that raises an exception
rescue => error
  NOTIFIER.notify_error(error)
end
```

### How can I use it with ExceptionNotification gem?

In `config/initializers/elastic_notifier.rb`, after initializing the notifier object as described above, you need to register it [as documented here](https://github.com/smartinez87/exception_notification#custom-notifier):

```ruby
notifier = ElasticNotifier.new(url: "http://myserver.com:9200")
ExceptionNotifier.add_notifier :elastic_search, notifier
```

Now any unhandled failures from your app will result in a document sent to ElasticSearch.

For background processes, you can leverage all registered notifiers (email, Elastic Search, Slack, etc.) with a single command!

```ruby
begin
  # some code that raises an exception
rescue => exception
  ExceptionNotifier.notify_exception(exception)
end
```

### What information is being sent?

At the time the notifier is invoked it collects some information from the environment, serializes it together with the exception details and sent it to the Elastic instance.

```ruby
{
  severity: "error",
  timestamp: "2017-12-31 23:59:59",
  program_name: "my_app.rb",
  pid: 1345,
  hostname: "myservicename",
  ip: "123.123.123.123",
  data: {
    name: "NoMethodError",
    message: "undefined method `test` for nil:NilClass",
    backtrace: [...]
  }
}
```

It is possible to override parameters such as `program_name` which will remain static for the notifier instance.

```ruby
notifier = ElasticNotifier.new(url: "http://myserver.com:9200", program_name: "custom_name")
```

### Do you fancy contributing?

Please get in touch! Bug reports and PR as well as simple feedback are very welcome! 
The code is available on [GitHub][elastic_notifier_github] and the gem is officially deployed to [Rubygems.org](https://rubygems.org/gems/elastic_notifier).

[elastic_notifier_github]: https://github.com/hspazio/elastic_notifier
