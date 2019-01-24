---
layout: post
title: "Enumerate through paginated REST API resources"
date: 2019-01-24
tags: [ruby, api]
comments: true
---

When working with a JSON API that exposes a list of resources we typically are in presence of paginated results.

A typical response body would contain a limited list of items, based on the size of the page and a `meta` node that contains information on how to paginate through all the items. This section might contain the current `page` number and the `total_pages` number or a URL to the `next` page, if any.

In the example below we have an endpoint that provides only the current page and the total number of pages, which are sufficient for the purpose of this post. The page size is 3, to keep the example short, but common choices may be 10, 30 or 50.

```javascript
GET /users

{
  "items": [
    { "id": 1, "name": "Jane", "age": 30 },
    { "id": 2, "name": "Mattew", "age": 24 },
    { "id": 3, "name": "Julia", "age": 28 }
  ],
  "meta": { 
    "page": 1, 
    "total_pages": 3,
    "next": 2
    // alternatively as url:
    // "next": "/users?page=2"
  }
}
```

When writing the client library for this API we would like to be able to iterate through all the users as we would normally do with a simple Array object. After all, the purpose of the client is to hide the complexity of dealing with multiple HTTP requests.

Our ultimate goal is to be able to use all `Enumerable` [methods](https://ruby-doc.org/core-2.6/Enumerable.html). For example:

```ruby
client.users.each { |user| puts user }

client.users.find { |user| user.name == 'Fabio' }

client.users.max_by { |user| user.age }
```

## A naive approach

At first we decide to 
1. leverage the `total_pages` to loop through all the pages
2. For each fetched page we extract the items and append them to a list
3. Return all the items when the `next` page is `nil` or when the current page reaches the total pages

```ruby
class Api::Client
  def users
    items = []
    page = 1

    loop do
      result = get("/users?page=#{page}")
      items += result['items'].map { |item| User.new(item) }

      break if page >= result['meta']['total_pages']
      page += 1
    end

    items
  end
end
```

Here is how this client will be used:

```ruby
client = Api::Client.new
client.users
# GET /users?page=1
# GET /users?page=2
# GET /users?page=3
> [
>   #<User:0x00007fa1ff926b40 @id=1, @name="Jane", @age=30>, 
>   #<User:0x00007fa1ff926aa0 @id=2, @name="Mattew", @age=24>, 
>   #<User:0x00007fa1ff9269d8 @id=3, @name="Julia", @age=28>, 
>   #<User:0x00007fa1ff926370 @id=4, @name="Alex", @age=28>, 
>   #<User:0x00007fa1ff926230 @id=5, @name="Bob", @age=27>, 
>   #<User:0x00007fa1ff926190 @id=6, @name="Alice", @age=29>, 
>   #<User:0x00007fa1ff925c18 @id=7, @name="Charlie", @age=35>, 
>   #<User:0x00007fa1ff925b78 @id=8, @name="Sarah", @age=33>
> ]
```

__The problem with this approach is that all the pages will be fetched regardless whether we need all the items or not.__

Consider the scenario where we need to find the user named "Bob" but we don't know in which page it will be. With the current approach we will fetch all the users, then we will use `find` on the returned Array. Not really efficient!

```ruby
client.users.find { |u| u.name == "Bob" }
# GET /users?page=1
# GET /users?page=2
# GET /users?page=3
# => #<User:0x00007f9e528d7a78 @id=5, @name="Bob", @age=27>
```

## Enumerator to the rescue!

A better approach that saves unnecessary requests is to use [Ruby's Enumerator](https://ruby-doc.org/core-2.6/Enumerator.html) which is very powerful tool that enables any possible ways to lazily iterate through a list.

The `Enumerator` yields a `yielder` object which is used to provide the next object in the list. Inside the `Enumerator`'s block we fetch the next page and add each item in the page to the `yielder`. Once the caller has iterated through all the items provided by the `yielder` the next cycle of the loop is triggered until there are no more pages to load.

```ruby
class Api::Client
  def users
    Enumerator.new do |yielder|
      page = 1

      loop do
        result = get("/users?page=#{page}")
        result['items'].each { |item| yielder << User.new(item) }

        break if page >= result['meta']['total_pages']
        page += 1
      end
    end
  end
end
```

The power of this approach is that we only fetch the pages we need, at the moment we need them. It's especially efficient when working with many pages.

Example: if we are looking for a user named "Bob", which is in page 2, the finder exits the loop as soon as the user if found, without fetching page 3.

```ruby
client.users.find { |user| user.name == "Bob" }
# GET /users?page=1
# GET /users?page=2
# => #<User:0x00007f9e528d7a78 @id=5, @name="Bob", @age=27>
```

# Generalization

Finally, this logic can be easily extracted into a more generic behavior to be used to paginate any resource, not just `User`:

```ruby
class Api::Client
  def users
    paginate_through('/users', factory: ->(item_params) { User.new(item_params) })
  end

  def posts
    paginate_through('/posts', factory: ->(item_params) { Post.new(item_params) })
  end

  def paginate_through(path, factory:)
    Enumerator.new do |yielder|
      page = 1

      loop do
        result = get("#{path}?page=#{page}")
        result[:data].each { |item| yielder << factory.call(item) }

        break if page >= result[:meta][:total_pages]
        page += 1
      end
    end
  end
end
```

This is a constant pattern I keep in my toolbox for building REST API clients. I hope you also found it useful.
