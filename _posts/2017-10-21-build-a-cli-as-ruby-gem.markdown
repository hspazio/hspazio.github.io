---
layout: post
title: "Build a CLI as Ruby Gem"
date: 2017-10-16 18:23:32
tags: [ruby, cli]
comments: true
---

Ruby is known for having a great ecosystem of code sharing within the community. Rubygems and Bundler have done an amazing job in simplifying this process and dependency management, even influencing modern package managers for other languages. 

Every developer that is new to Ruby also knows how to `require` a file and/or an installed Gem. However, one of the most under estimated features of a Ruby Gem is the capacity of running as a standalone command-line executable.

Today we are going to build a simple weather CLI using plain Ruby with the help of [Bundler][bundler].
The full project is also available on [GitHub][weather_repo].

First of all let's create a new Gem project boilerplate using Bundler. 

{% highlight bash %}
bundle gem weather --test=minitest
{% endhighlight %}

> For more details on how to build and publish a gem to an internal gemserver checkout [my presentation here][bundler_geminabox_presentation].

## Customise the gemspec
We need to make initial modifications to `weather.gemspec` file. First, editing any fields marked with a TODO.

{% highlight ruby %}
spec.summary = "simple weather CLI"
spec.description = "Simple weather CLI using OpenWeatherMap API"
spec.homepage = "http://idonthaveanybutyoumayhaveone.io"
{% endhighlight %}

Add the following line to define which file will be executed when we will invoke `weather` command in the terminal. Note that the command will take the name of the "default executable", not the Gem name.
{% highlight ruby %}
spec.default_executable = "weather"
{% endhighlight %}

Then we will need to create the file `exe/weather`. You may need to first create the "exe" directory. We will fill the content of this file later.

## Set up the weather API account
For this tutorial we are going to use [OpenWeatherMap API][weather_api_site] as they provide nice features with the free account. We need to register [here][weather_api_key] to get an app ID.

I've obtained an app ID and set it as environment variable `WEATHER_APP_ID` so that we don't commit it to the source control system. Hour gem is going to read it from the global `ENV` variable.

## Time to implement!
Using Test-driven development we design our weather client. We start by designing a `Weather::Gateway` that we will use to encapsulate the details of the API provider, so that if we decide to user another provider, our changes are constrained within this class.

{% highlight ruby %}
require 'test_helper'

describe Weather do
  it 'has a version number' do
    refute_nil Weather::VERSION
  end
end

describe Weather::Gateway do
  before do
    app_id = ENV['WEATHER_APP_ID']
    abort "set WEATHER_APP_ID env variable" unless app_id
    @gateway = Weather::Gateway.new(app_id)
  end

  it 'gets forecast for New York' do
    samples = @gateway.forecast('New York')

    assert samples.any?
    samples.each do |sample|
      assert_respond_to sample, :date
      assert_respond_to sample, :description
      assert_respond_to sample, :humidity
      assert_respond_to sample, :temp_min
      assert_respond_to sample, :temp_max
    end
  end
end
{% endhighlight %}

Here is a basic implementation that satisfies the test. We do it in a file called `lib/weather.rb`
{% highlight ruby %}
require 'net/http'
require 'json'
require 'weather/version'

module Weather
  Sample = Struct.new(:date, :temp_min, :temp_max, :humidity, :description)

  class Gateway
    ROOT_URL = 'http://api.openweathermap.org/data/2.5'

    def initialize(app_id)
      @app_id = app_id
    end

    def forecast(query)
      uri = forecast_uri_for(query)
      response = Net::HTTP.get_response(uri)

      if response.code == '200'
        sample_list_from_response(response)
      else
        raise response.body
      end
    end

    private 

    def sample_list_from_response(response)
      data = JSON.parse(response.body, symbolize_names: true)

      data[:list].map do |sample|
        sample_from_json(sample)
      end
    end

    def sample_from_json(json)
      Sample.new(
        Time.at(json[:dt]).strftime('%Y-%m-%d %H:%M'),
        json[:main][:temp_min],
        json[:main][:temp_max],
        json[:main][:humidity],
        json[:weather].first[:description]
      )
    end

    def forecast_uri_for(query)
      params = { 
        units: 'metric', 
        appid: @app_id,
        q: query 
      }
      uri = URI(ROOT_URL + '/forecast')
      uri.query = URI.encode_www_form(params)
      uri
    end
  end
end
{% endhighlight %}

## Pulling it all together
Time to edit `exe/weather` to make use of the `Weather::Gateway` class and print out the results to the terminal.

{% highlight ruby %}
#!/usr/bin/env ruby
require 'weather'

app_id = ENV['WEATHER_APP_ID']
city   = ARGV[0]

abort "WEATHER_APP_ID environment variable is not set" unless app_id
abort "Provide a city as argument" unless city

gateway = Weather::Gateway.new(app_id)
samples = gateway.forecast(city)

puts "Weather forecast for #{city}"
puts "DATE\tTEMP(min)\tTEMP(max)\tDESCRIPTION"
samples.each do |sample|
  puts "#{sample.date}\t#{sample.temp_min}\t#{sample.temp_max}\t#{sample.description}"
end
{% endhighlight %}

The first line (shebang) is crucial for the executable to work. It tells Rubygem how to run this executable and lets us not to worry about cross-platform compatibility.
In fact, while on a Unix environment it's sufficient to have the shebang and executable permission on the file to run, when installing the gem on Windows, Rubygem will automatically create a `weather.bat` file that runs the `exe/wather.rb` file using the currently active Ruby installation. Great, isn't it?

Now we can finally build and install the Gem to see it working!
{% highlight bash %}
gem build weather.gemspec
gem install weather-0.1.0.gem

weather Rome
# Weather forecast for Rome
# DATE    TEMP(min)       TEMP(max)       DESCRIPTION
# 2017-10-17 22:00        18.42   20.25   clear sky
# 2017-10-18 01:00        12.68   14.06   clear sky
# 2017-10-18 04:00        8.89    9.81    clear sky
# 2017-10-18 07:00        6.92    7.38    clear sky
# 2017-10-18 10:00        5.63    5.63    clear sky
# 2017-10-18 13:00        4.55    4.55    clear sky
# 2017-10-18 16:00        14.24   14.24   clear sky
# 2017-10-18 19:00        18.48   18.48   clear sky
# 2017-10-18 22:00        18.82   18.82   clear sky
# 2017-10-19 01:00        12.24   12.24   clear sky
# 2017-10-19 04:00        7.87    7.87    clear sky
# 2017-10-19 07:00        5.94    5.94    clear sky
# 2017-10-19 10:00        4.3     4.3     clear sky
# 2017-10-19 13:00        3.31    3.31    clear sky
# 2017-10-19 16:00        14.65   14.65   clear sky
# 2017-10-19 19:00        19.29   19.29   clear sky
# 2017-10-19 22:00        19.85   19.85   clear sky
# 2017-10-20 01:00        12.4    12.4    clear sky
# 2017-10-20 04:00        7.56    7.56    clear sky
# 2017-10-20 07:00        5.42    5.42    clear sky
# 2017-10-20 10:00        4.16    4.16    clear sky
# 2017-10-20 13:00        3.29    3.29    clear sky
# 2017-10-20 16:00        15.68   15.68   clear sky
# 2017-10-20 19:00        21.14   21.14   clear sky
# 2017-10-20 22:00        21.53   21.53   clear sky
# 2017-10-21 01:00        13.54   13.54   clear sky
# 2017-10-21 04:00        9.2     9.2     few clouds
# 2017-10-21 07:00        7.66    7.66    few clouds
# 2017-10-21 10:00        7.06    7.06    broken clouds
# 2017-10-21 13:00        7.31    7.31    overcast clouds
# 2017-10-21 16:00        15.92   15.92   broken clouds
# 2017-10-21 19:00        20.25   20.25   broken clouds
# 2017-10-21 22:00        21.02   21.02   clear sky
# 2017-10-22 01:00        15.7    15.7    scattered clouds
# 2017-10-22 04:00        13.66   13.66   few clouds
# 2017-10-22 07:00        13.4    13.4    scattered clouds
# 2017-10-22 10:00        14.6    14.6    scattered clouds
# 2017-10-22 13:00        15.25   15.25   overcast clouds
# 2017-10-22 16:00        18.75   18.75   broken clouds
# 2017-10-22 19:00        22.79   22.79   broken clouds
{% endhighlight %}

## Conclusion
In the past I've used libraries like [Ocra][ocra] to compile command-line tools from Ruby into executable Windows binaries. However things started to get complex after a while as we had to support a combination of Windows and Unix 32 and 64 bits executables. Unnecessary complexity started to appear considering that our team's primary focus was not to build command line tools.

Ocra is a good solution if the users of your command line tool are not expected to have an installation of Ruby on their environment or if asking the users to install it as dependency is unacceptable. For the rest, building an executable gem is a great, cheap and more maintainable solution.

[bundler]: http://bundler.io
[bundler_geminabox_presentation]: http://slides.com/fabiopitino/bundler-and-geminabox
[weather_api_site]: https://openweathermap.org
[weather_api_key]: https://openweathermap.org/appid
[weather_repo]: https://github.com/hspazio/weather
[ocra]: https://github.com/larsch/ocra
