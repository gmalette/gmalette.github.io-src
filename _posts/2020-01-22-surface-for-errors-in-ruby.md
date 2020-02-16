---
layout: post
title: Surface for Errors in Ruby
date: 2020-01-22 20:00:00 +0400
description: __CHANGE__
categories:
- Ruby
---

A few years ago, I read [Sandi Metz's blog post about affordances](https://www.sandimetz.com/blog/2018/21/what-does-oo-afford) and it stuck in the back of my mind. Every time I see an API that's ripe for misuse, I get reminded of that blog post. I've been noticing many idioms in Ruby and Rails which immediately create opportunities for mistakes. At the same time, I've been exploring how other languages and framework solve similar problems. I now feel the need to share my thoughts on what I call Surfaces for Error, which I think is equivalent to what Sandi Metz calls affordances.

## What are Surfaces for Error

The surface of a product is the sum of all ways an API can be used. Applied to an API, it would be the sum of all possibilities it offers, both intended and unintended.The surface for errors is the portion of the surface that produces invalid states, undefined or unexpected behaviours, or bugs.

I'm interested in focusing on that part. Whatever the reasons, it could be through poor design, ambiguous requirements, missing framework or language features, it could even be voluntary, we introduce ways in which the APIs we design can be misused. A great example of this is the `errors` API in Rails. Consider the following code:

## Specific Incantation

Specific Incantations are the type of APIs where one must perform the ritual–completely, and in the correct order, without guidance–, to get the desired outcome. A great example of this is the `errors` behaviour in `ActiveRecord`.

```ruby
class User < ApplicationRecord
  validates(:name, presence: true) # 1
end

User.new # 2
user.errors.any? # 3
# => false
```

In the previous example, we create a `User` class which specifies that the `name` attribute must be set, otherwise it's not valid (#1). We then instantiate a `User`, without giving it a `name` (#2); we can then expect it to be invalid, but we can see that it contains no errors (#3).

Of course, this is because `ActiveRecord` requires the programmer to use `valid?` before calling `errors`, which mutates the `User` to set `errors`.

```
user.valid? # 4
# => false

user.errors.any? # 5
# => true
```

After calling `valid?` (#4), the `User`'s `errors` are now populated (#5).

Specific Incantation APIs can often be exposed in ways that completely suppress the problems, so much so that I've always been stricken that Rails kept choosing not to do anything. An obvious way to remove this surface for error would be to generate `errors` when the method is called.

Another, perhaps more interesting way, is [Ecto's `Changeset` API](https://hexdocs.pm/ecto/Ecto.Changeset.html). It completely eliminates the problem by using `Changeset` which are either valid or invalid, and are the vehicle for change. If it were Rails, you would perhaps write it this way:

```ruby
changes = user.change(name: "Guillaume")
if changes.valid?
  user.save(changes)
end
```

While I do understand that Ecto's `Changeset` API is largely driven by Elixir's immutability, I still believe that Rails could have chosen a similar API.

## Ignored Values

Some APIs will use values that are easy to ignore, once again leading to errors.

While I don't want to seem like I pick on `ActiveRecord`, it contains examples within reach.

~~The first case I want to talk about is far from exclusive to `ActiveRecord` or Rails, and is quite prevalent in Ruby (and other languages).~~

# NO
```ruby
def find_user
  User.find_by(name: @name)
end
```

~~In this example, the return value can be `nil`, if the `~~

```ruby
def update
  user.update(user_update_params)
  user.save

  respond_with(user)
end
```

In this example, the return value of `user.save`–a `Boolean` indicating whether the persistance was successful or not–is not checked. The error case, when `save` fails, is not handled. `ActiveRecord` doesn't help nor guide towards handling it, either.

In Ruby, this surface is hard to remove. For APIs where handling errors is critical, it can be enforced through callbacks:

```ruby
user.save(on_error: -> (e) { ... })
```

For more interesting ways in which other languages have solved this problem, some languages have implemented [substructural type systems](https://en.wikipedia.org/wiki/Substructural_type_system#Linear_type_systems).

In [Rust](https://www.rust-lang.org/) specifically, API developers can mark return values as `must_use`, meaning that the return value *must be checked*. Failing to do so would result in a compiler warning. [ATS](http://www.ats-lang.org/) achieves similar results via proofs. Unconsumed proofs prevent compilation.


## Attention Auditor

The Attention Auditor is when an API makes it easy to do the wrong thing, while seemingly doing the right thing.

Visibility modifiers (`private`, `protected`, `public`) are prime Attention Auditors; they are the source of mistakes, for veterans and newcomers alike. To say nothing of the fact that `protected` means something different than in other languages, many devs make mistakes when it comes to singleton methods, often called "class methods". Here's an example:

```ruby
class Foo
  private

  def self.bar
    :bar
  end
end
```

Many devs would expect the `Foo.bar` method to be private to `Foo`, however it is not. The reason is that `private` is a method on `Foo`, telling it to mark the methods defined after as `private`. It does not affect `Foo`'s singleton class. To learn more about the relation between classes and their singleton classes, refer to [The Ruby Object Model]({% post_url 2019-02-01-the-ruby-object-model %}).

1. Explicitly call `private` when defining private methods.

    ```ruby
    class Foo
      private def baz
        :baz
      end

      private def self.bar
        :bar
      end
      # => NameError (undefined method `bar' for class `Foo')
    end
    ```

    While this is slightly more verbose, it will prevent this kind of mistakes.

2. Always define singleton methods by opening the singleton class.

    ```ruby
    class Foo
      class << self
        private

        def bar
          :bar
        end
      end
    end
    ```

    Likewise, this approach removes the entire surface for errors.

## Missing Coupling



~~What's more, while the language claims to have non-semantic whitespaces–spaces don't have meaning–, it's not always the case, and there are plenty of examples where~~


## Surface for Errors in Ruby, Specifically

~~Ruby is possibly the most complicated programming language I know. As I explained in more details in the post [Revoking the (Parentheses) Privilege]({% post_url 2018-12-03-revoking-the-parentheses-privilege %}), the language designer's choice to make parentheses optional can lead to many programmer mistakes.~~

~~As you might expect, that's not the only place Ruby gives developpers the opportunity to make mistakes.~~

__END__

, and it made me wonder: how others manage to avoid them, and more importantly, if we can steal other ways of doing and apply them to Ruby.

Let me clarify what I mean with "affordance
