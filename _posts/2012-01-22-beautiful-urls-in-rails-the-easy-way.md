---
layout: post
title: Beautiful URLs in Rails. The easy way.
date: 2012-01-22
excerpt: A really simple approach of having readable URLs on Rails without using FriendlyID.
categories:
- rails
- url design
- metaprogramming
---

**Disclaimer.** I know [FriendlyID](http://rubygems.org/gems/friendly_id) exists,
and you should probably use it. This approach is for something that was done a
*long* time ago and doesn't require the `slug` to be persisted on the database.

---

I've always been in love with [beautiful URL design][url-design]. Specially for 'public' parts of the site. In Rails, it's pretty common to use the ID of the object in the URL. Well, that **really** sucks.

My first approach to solve this problem was started by [Federico Builes][federico] a long time ago with a plugin ([which I now maintain](https://github.com/nhocki/make_permalink) as a gem) called [Make Permalink](http://rubygems.org/gems/make_permalink). It's really easy to use and will (up to some point) solve the problem.

```ruby
# https://gist.github.com/nhocki/1660555
class Post < ActiveRecord::Base
  make_permalink :title

  # Override this method to have nice looking
  # URLs. You still use `Post.find` in the controller
  # and will get URL like this:
  #
  #    /posts/1-look-mom-i-rock
  def to_param
    permalink
  end
end
```

The good thing about having `/users/1-nhocki` as the URL is that is way better than `/users/1` **AND** you don't have to change any of your app for that (you can still use `find` in your controllers).

I was really glad with this but not too long ago, someone asked me for something similar but for Mongoid. And hell! Mongoid ids are ugly as shit! So, they wanted to have _**just**_ the username. I said, well, why don't you just change the `to_param` method to return the username and use `User.find_by_username` everywhere?

Even though this actually works, I would be very tired from writing `find_by_username` over and over again. So I told him to write a `fetch` method where he could get an user either by the `id` or the `username`. I think this is pretty convenient and nice, but I would _**hate**_ to write that fetch method in every model. So, with some metaprogramming we can easily do this.

First, I've created a `fetchable_on` method to use it in your models. This will define a class method called `fetch` that will let you get the object by querying on the `attribute` **OR** on the `id`. Right now, it won't work if your attribute is a numeric value (like the number of bathrooms you have in your house), but most of the times, it's ok. Here's the code. Place it in your initializers folder.

```ruby
# https://gist.github.com/nhocki/1657758
# config/initializers/fetchable.rb

# encoding: UTF-8

module Extensions
  module Fetchable
    # Adding `fetchable_on` to a model will define the `fetch` method
    # that will allow you to query that model with the `attribute` or `id`.
    #
    # Having the following:
    #
    #    class User << ActiveRecord::Base
    #      fetchable_on :username
    #    end
    #
    # Will let you fetch users by username or id.
    #
    #    User.fetch('nhocki').id # => 1
    #    User.fetch(1).username # => nhocki
    def fetchable_on(attribute)
      define_singleton_method("fetch") do |*args|
        looked_value = args.shift
        begin
          find(looked_value.to_i, *args)
        rescue ActiveRecord::RecordNotFound
          send("find_by_#{attribute}!", looked_value.to_s, *args)
        end
      end
    end
  end
end

ActiveRecord::Base.extend(Extensions::Fetchable)
```

That way, you just need to call `fetchable_on :attribute` in your model and it'll adjust the `fetch` method to query on that `attribute` or the `id`. Here's an example:

```ruby
class User < ActiveRecord::Base
  fetchable_on :username

  # URL Style: /users/nhocki
  def to_param
    username
  end
end

# Gives you the `fetch` method.
# It takes the username or the id.
# Use fetch in your controller

User.fetch('nhocki').id # => 1
User.fetch(1).username  # => 'nhocki'
```

**Note that I use `find_by_attribute!` to throw an exception if the object is not found. Using the method without a `!` will return `nil` if there's no object in the DB.**

How would you implement this? Any good solutions?

[url-design]: http://warpspire.com/posts/url-design
[federico]: http://mheroin.com
