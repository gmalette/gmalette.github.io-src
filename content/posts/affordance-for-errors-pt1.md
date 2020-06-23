---
title: Affordance for Errors, part 1
date: 2020-01-29 23:00:00 +0400
description: In this first posts of three, I highlight a few examples of affordance for errors in common APIs.
categories:
- Ruby
aliases:
- "ruby/2020/01/29/affordance-for-errors-pt1.html"
---

A few years ago, I read [Sandi Metz's blog post about affordances](https://www.sandimetz.com/blog/2018/21/what-does-oo-afford) and it stuck with me. I started minding how the APIs I write can be misused; I started noticing the surface I leave for the users of my APIs to make errors. Soon after I started noticing how the APIs we use every day are full of affordance for error.

In this first of three posts, I want to show a few examples of how common APIs (mostly in Ruby and Rails) can be accidentally misused. Maybe this will spark a discussion, or will incite people to share other issues they've seen. The second post will explore solutions to the problems from this post. In the final post of this series, I will show how I approach API design to minimize the affordance for errors while offering the best ergonomics I can.

## Why Bother?

You may be wondering why this matters, _why can't just write better code, without mistake_. After all, we're smart people, we should be able to learn these tools and to not make mistakes... If you are the kind of developer that can write bug-free code, this blog post is even more so for you, as your coworkers cannot. Developers are humans. We get tired, we forget, our attention lapses, we're juggling umpteen other things in our heads. For many reasons, we end up making mistakes. It is then our shared responsibility to make your APIs human-friendly.

Moreover, good APIs minimize the amount of mistakes we can make, they remove the pain and frustration of using them. They lower the cost of entry. They don't require you to understand how things are implemented "on the metal". Providing good APIs with minimal affordance for errors is about making the life of your fellow developers just a little better, making them just a little more productive, easing their cognitive load just a little.

Good APIs will naturally guide developers towards the best ways to use them, in the same subtle yet important way the bumps on the `F` and `J` keys guide your hands towards the good hand placement.

My hope is that after reading this post, you too will start taking notice of what errors you afford your users, and that we can all use this as a starting point to write better, human-friendlier APIs.

## Examples of Surfaces for Errors

Jump right ahead:
- [ActiveRecord's Errors API](#activerecords-errors-api)
- [N + 1](#n--1)
- [Illegal States](#illegal-states)
- [Ruby’s Visibility Modifiers](#rubys-visibility-modifiers)
- [Ignored Values](#ignored-values)
- [Missing Coupling](#missing-coupling)
- [Extending Self](#extending-self)
- [`super` vs `super()`](#super-vs-super)

### ActiveRecord's Errors API

I want to start this off with an easy, approachable example: ActiveRecord's `errors` API, which requires a specific incantation.

```ruby
class User < ApplicationRecord
  validates(:name, presence: true) # 1
end

user = User.new # 2
user.errors.any? # 3
# => false

user.valid? # 4
# => false

user.errors.any? # 5
# => true
```

For those unfamiliar with Rails, in the previous example, we
- We create a `User` class which specifies that the `name` attribute must be set, otherwise it's not valid (#1)
- We then instantiate a `User`, without giving it a `name` (#2); we can then expect it to be invalid
- However, we can see that it contains no errors (#3)
- This is because ActiveRecord requires the programmer to use `valid?` (#4) before calling `errors`, which mutates the `User` to set `errors` (#5)

APIs like this can often be exposed in ways that completely suppress the problems, so much so that I've always been stricken that Rails kept choosing not to do anything (reminder: exploring solutions will be the topic of the second post).

### N + 1

Some APIs, like ActiveRecord, allow mistakenly loading data serially instead of in a batch manner, without guiding the user towards the more performant approach.

```ruby
investor = Investor.first

investor.accounts.each do |account|
  puts account.fund.name
end

# Fund Load (1.2ms)  SELECT  "funds".* FROM "funds" WHERE "funds"."id" = $1 LIMIT $2  [["id", 6], ["LIMIT", 1]]
# Fund Load (1.4ms)  SELECT  "funds".* FROM "funds" WHERE "funds"."id" = $1 LIMIT $2  [["id", 7], ["LIMIT", 1]]
# ... and so on for each account
```

Because the code that loaded the `investor` object did not `preload` the `fund`, every iteration of the loop will issue a query to load one `fund`.

If you have used Rails with ActiveRecord for more than a few days, the probability that you've encountered or written code with this mistake are high. This problem is so common that [a gem (Bullet)](https://github.com/flyerhzm/bullet) was written to help developers detect when they cause it.

### Illegal States

If I were to choose, this would be the most important one.

> Make illegal states unrepresentable

Attributed to [a 2011 post by Yaron Minksy](https://blog.janestreet.com/effective-ml-revisited/) (which I became aware of only much later), this quote completely changed the way I approach software design. This practice eliminates so many possible mistakes that it's hard to even explain the implications.

__If no illegal state can be represented, we never have to worry about the validity of objects__.

More often than not, however, we write code that allows representing invalid state. ActiveRecord initialization is a good example, but there are so many others. All APIs that use `Hash`es to represent state immediately allow illegal state. There are many, many variations of APIs offering this kind of surface for errors, I encourage you to keep your eyes peeled and take notice.

#### Incomplete Booleans

One example, I've recently used an APIs for inventory management using a number of booleans to represent possible states, but the number of allowed states is less than the booleans combined values allow. Inventory items can be configured with the boolean values `inventory_is_managed` (the inventory counter can be set and is decremented after a sale), and `allow_negative` (the item can continue selling once after it the counter reached zero).

The problem is that this API has 4 configurations, but only 3 are valid:
- `(inventory_is_managed: true, allow_negative: true)` the inventory is managed, but can go negative
- `(inventory_is_managed: true, allow_negative: false)` the inventory is managed, and cannot go below zero
- `(inventory_is_managed: false, allow_negative: _)` the inventory is not managed; whether or not it can go below zero is meaningless

Users of this API should not be able to set any value for `allow_negative` if `inventory_is_managed` is `false`.

#### Email is a String, Struct is a Hash

Ruby code is [plagued with primitive obsessions](https://refactoring.guru/smells/primitive-obsession). We represent emails using `String`, and "complex" data using `Hash`. Neither of these classes allow preserving the invariants of our system (ex: the email matches a regular expression). As a result, most codebases I've seen build defenses; they validate the state more than once.

### Ruby's Visibility Modifiers

There's an entire category of errors stemming from APIs allowing you to do the wrong thing, while seemingly doing the right thing.

Visibility modifiers (`private`, `protected`, `public`) are prime examples; they are the source of many mistakes, by newcomers's mismatched expectations and veterans's lapse of attention. To say nothing of the fact that `protected` means something different than in other languages, many devs make mistakes when it comes to singleton methods, often called "class methods". Here's an example:

```ruby
class Foo
  private

  def self.bar
    :bar
  end
end
```

Many devs would expect the `Foo.bar` method to be private to `Foo`–perhaps because many call it a class method–, however it is not. The reason is that `private` is a method on `Foo`, telling it to mark the methods defined after as `private`. It does not affect `Foo`'s singleton class. To learn more about the relation between classes and their singleton classes, refer to [The Ruby Object Model]({{< ref "posts/the-ruby-object-model" >}}).

That it is such a common error pains me the utmost, because it is so unnecessary. Approximately 100% of other languages do not have this problem, by having visibility modifiers at the method definition level. In fact, it even works in Ruby, but I digress.

### Ignored Values

Some APIs will use values that are easy to ignore, once again leading to errors.

While I don't want to seem like I pick on ActiveRecord, it contains examples within reach.

```ruby
def update
  user.update(user_update_params)
  user.save

  respond_with(user)
end
```

In this example, the return value of `user.save`–a `Boolean` indicating whether the persistance was successful or not–is not checked. The error case, when `save` fails, is not handled. ActiveRecord doesn't help nor guide towards handling it, either.

If you think your code is impervious to this problem, I suggest you look at your test suite. If you're using `save` and not `save!` to setup objects, and unless you have very high code hygiene, chances that at least one started failing since you wrote it.

### Missing Coupling

APIs sometimes need things to be coupled, but don't express it. For example, in Ruby, any object can be a `Hash` key by defining two methods: `hash` and `eql?`. However, this is not encoded at runtime. To make things worse, the `Object` class defines an implementation based on object equality.

As a result, it's easy for developers to redefine either of the two methods but not both. This can easily lead to situations where the objects seem to be valid hash keys, until there two items end up in the same bucket, or, conversely, when two items should fold together, but don't.

### Extending Self

The usage of `extend(self)` in module is probably one of my biggest pet peeves.  It's used to add methods to the singleton class of a module, effectively allowing the module to receive the methods it defines. Ex:

```ruby
module Foo
  extend(self)

  def foo # 1
    :foo
  end
end

Foo.foo # 2
# => :foo
```

In the previous example, the module `Foo` defines the method `foo` (#1) which is available on `Foo` (#2) due to the `extend(Foo)`. Not only does this idiom produce a complicated object, the fact that it's an idiom affords users some errors.

I think people fail to consider the implications of these modules. Should `Foo` be used as a singleton, or should it be `include`'d, as modules are? In the previous example, it doesn't matter. When you start using instance variables and state, it does.

### `super` vs `super()`

In Ruby, as we know, parentheses are optional, unless they aren't. The usage of `super` is such an example, where `super` (without parentheses) is semantically different to `super()` (with parentheses).

```ruby
class Parent
end

class WithParentheses < Parent
  def initialize(var)
    @var = var
    super()
  end
end

class WithoutParentheses < Parent
  def initialize(var)
    @var = var
    super
  end
end

WithParentheses.new(1) # 1
# => #<WithParentheses:0x000...>

WithoutParentheses.new(1) # 2
# => ArgumentError: wrong number of arguments (given 1, expected 0)
```

This example shows the semantic difference between both versions.

The reason is that `super` (without parentheses) sends the arguments verbatim, whereas `super()` (with parentheses) sends no argument.

## Conclusion

I hope this post has opened your eyes on the amount of affordance for errors offered by our languages and APIs. I hope you'll start noticing them in the code you're writing, and you'll strive to remove as many as possible.

### Post Scriptum

My list contained many other affordances that didn't make the cut for this post. To name a few:

- `HashWithIndifferentAccess` blurring the line between `String` and `Symbol`
- `validates_uniqueness_of` is racy by default
- `assert(obj, message)` is easily mistaken with `assert_equal(a, b)`
- `config(default_value: :false)` (using a symbol where a boolean was expected)
- `ActiveRecord` scopes behaving like `Array` but not quite
- Using of structureless markup languages (YAML)

If you wish to contribute with more examples, please share them in the comments.

[Comment or Like](https://github.com/gmalette/gmalette.github.io-src/pull/7)
