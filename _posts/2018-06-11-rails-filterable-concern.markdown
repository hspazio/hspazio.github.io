---
layout: post
title: "Enhanced Filterable concern for Rails models"
date: 2018-06-07
tags: [ruby, rails, concerns]
comments: true
---

If you have ever tried to filter Rails models via a RESTful controller, you've probably read [Justin Weiss](https://twitter.com/justinweiss)'s great [article][filterable_post] about making models filterable without bloating the controllers. 

I owe him most of the credits for this post because not only did his idea greatly helped to simplify codebases I worked on but it became a must-have Concern for almost all Rails applications I've been working on since. 🙏 Thank you!

The idea is to allow a list of parameters, provided via the controller, to be used for filtering ActiveRecord models. This behavior is defined in a single-purpose `Filterable` concern as below (*Justin's code*):

```ruby
# app/models/concerns/filterable.rb
module Filterable
  extend ActiveSupport::Concern

  module ClassMethods
    def filter(filtering_params)
      results = self.where(nil)
      filtering_params.each do |key, value|
        results = results.public_send(key, value) if value.present?
      end
      results
    end
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Filterable
  ...
end
```

And then used in the controller with a single method:

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = Post.filter(params.slice(:status, :published_after, :author))
end
```

This approach worked greatly until one day I happened to work on an advanced search page for an application. We were allowing users to execute fine grained searches on a number of domain models using various input fields.

Every time we wanted to add a new filter to the page we had to:

1. Create a scope in the dedicated model class.
2. Permit the scope in the controller's parameters for the `filter` method.
3. Add the relative view code such as input tags, select tags, etc.

__It quickly became difficult to maintain the filters__ especially remembering to go through all the 3 steps. 

In all honesty, __step #1__ is hard to miss because it's normally the first behavior to be implemented and it's also the core of the feature.

__Step #3__ might be optional if implementing JSON API or, if in presence of a frontend app, we may have a wireframe and strong user expectations.

__Step #2__, permitting the parameter in the controllers, from my experience, was easy to forget.

I just found it hard to remember! 😅

I use Test-Driven Development so I always end up having tests in place for the scopes and the view. However, I was not going to create controller tests for each scope we were exposing via the advanced search page. Are these controller tests, which are often slow, really needed?

## What if we could automatically update the permitted filters in the controller?

I wanted the controller to use some sort of class method that provides only the allowed parameters for the searching. I called it `search_params` and I thought we could splat them inside the `params.slice` method in the controller.

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = Post.filter(filter_params)
end

private

def filter_params
  params.slice(*Post.search_params)
end
```

At this point we could make the `Filterable` concern able to hold a list of scopes of a certain type, which we could call `search_scopes`.
We could define a class method `search_scope` which delegates to the ActiveRecord's `scope` and also register itself for later use, in the controller.

```ruby
# app/models/concerns/filterable.rb
module Filterable
  extend ActiveSupport::Concern

  included do
    @search_scopes ||= []
  end

  module ClassMethods
    attr_reader :search_scopes

    def search_scope(name, *args)
      scope name, *args
      @search_scopes << name
    end

    def filter(params) 
      # same as before
    end
  end
end
```

With this simple change we could easily migrate the model scopes that we wanted to expose via the controller. This was possible by simply changing `scope` with `search_scope`.

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  search_scope :status, -> (status) {
    where(status: status)
  }

  search_scope :published_after, -> (date) {
    where('published_at >= ?', date)
  }

  # some other scopes we do not want to expose via the controller
  scope :private, -> { where(private: true) }

  ...
end
```

## Conclusions

Not only did this enhancement remove any changes in the controllers, it also brought other positive side effects:

1. There was no need to create extra controller tests.
2. It helped to enforce security as only the search scopes were automatically permitted.
3. It became a signpost for all other developers as the scope needed to be treated with a certain attention (like prevent SQL injection).

[filterable_gist]: https://gist.github.com/justinweiss/9065666 
[filterable_post]: https://www.justinweiss.com/articles/search-and-filter-rails-models-without-bloating-your-controller
