---
layout: post
title: Affordance for Errors, part 2
date: 2020-02-16 15:00:00 +0400
description: In this second post of three, I show how other APIs and languages have avoided some of the problems shown in the first post.
categories:
- Ruby
---

In the first part of this series of three posts, I outlined a few common Ruby and Rails API, and explained the errors they afford their users. In this second post, I'll discuss how other tools have avoided these problems, and how those solutions may apply to Ruby and Rails. I'll be reusing the examples from the previous post. If you haven't read it, I suggest you start there: [Affordance for Errors, part 1]({% post_url 2020-01-29-affordance-for-errors-pt1 %}).

## How Others Have Solved Similar Problems

Jump right ahead:
- [ActiveRecord's Errors API](#activerecords-errors-api)
- [N + 1](#n--1)
- [Illegal States](#illegal-states)
- [Ruby’s Visibility Modifiers](#rubys-visibility-modifiers)
- [Ignored Values](#ignored-values)
- [Missing Coupling](#missing-coupling)

### ActiveRecord's Errors API

Quick reminder: Some APIs force the user to invoke methods in a particular order. Ex: ActiveRecord forces you to use the `valid?` method before calling the `errors` method [(link)]({% post_url 2020-01-29-affordance-for-errors-pt1 %}#activerecords-errors-api).

```ruby
user = User.new
user.errors.any?
# => false

user.valid?
# => false

user.errors.any?
# => true
```

An obvious way to remove this surface for error would be to generate `errors` when the method is called.

Another, perhaps more interesting way, is [Ecto's `Changeset` API](https://hexdocs.pm/ecto/Ecto.Changeset.html). It completely eliminates the problem by using `Changeset` which are either valid or invalid, and are the vehicle for change. If it were Rails, you would perhaps write it this way:

```ruby
changes = user.change(name: "Guillaume")
if changes.valid?
  user.save(changes)
end
```

### N + 1

Quick reminder: Some APIs's default usage mode is what users consider a mistake. Ex: ActiveRecord allows mistakenly loading records in a serial manner, rather than as a batch [(link)]({% post_url 2020-01-29-affordance-for-errors-pt1 %}#n--1).

_I can already hear [Alex Snaps](https://github.com/alexsnaps) complaining about ORMs. ORMs have their fair share of issues, I get it!  I guess this problem exposes a conumdrum: either we allow the prevalent\* N+1 issue, or we force coupling between unrelated pieces of code._

All the approaches to solving this problem I can think of consist of considering that loading an association of a record loaded through a `has_many` association without precising _how_, is invalid. Let's dissect that.

```ruby
investor = Investor.find(id)

investor.accounts.each do |account|
  puts account.fund.name
end
```

In the previous code, `accounts` is a `has_many` association on `investor`, so any association of it cannot be loaded unless we specify how. This would prevent the `account.fund` from working. However, if an `Account` was loaded outside an association, it could still be considered valid, ex:

```ruby
account = Account.find(id)

puts account.fund.name # this is fine
```

Rails already allows specifying how to load associations, with `includes`, `eager_load`, or `preload`, ex:

```ruby
investor = Investor.includes(accounts: :fund).find(id)

investor.accounts.each do |account|
  puts account.fund.name
end
```

This is sufficient to prevent the problem. We could also imagine a different API achieving the same results:

```ruby
investor = Investor.includes(accounts: :fund).find(id)

investor.accounts(preload: :fund).each do |account|
  puts account.fund.name
end

# or

investor.accounts.each do |account|
  puts account.load_fund.name
end
```

The point is to make situations that produce N+1 only achievable explictly. This is the path [`Ecto`](https://hexdocs.pm/ecto/Ecto.html#module-associations) chose.

\* You don't have to take my word for it Google "[Rails](https://www.mskog.com/posts/42-performance-tips-for-ruby-on-rails/#eliminate-n-1-queries) [performance](https://yalantis.com/blog/how-to-speed-up-your-ruby-on-rails-app/#h.6d0kbb4x1751) [tips](https://www.rorexpertsindia.com/blog/top-3-tips-boost-performance-ruby-rails-application/) [and tricks](https://www.codewithjason.com/rails-performance-tips/)". Nearly all of them will mention N + 1 queries.

### Illegal States

Quick reminder: if no illegal states can be represented, we never have to worry about the valitity of our objects.

I don't think I can express this better than [Yaron Minksy](https://blog.janestreet.com/effective-ml-revisited/), but I'll try to bring it into the Ruby world, if only because this is my favourite topic.

#### `String`, `Hash`, and other primitives

More often than not, when we use the language's primitives, we give up an opportunity to express the domain model of our application. When we use a `String` to represent an `Email`, we haven't encoded the fact that it needs to satisfy some constraints (ex: have an `@`, or satisfy a regular expression, or have a finite length). When we use `Hash` to represent structured data, we allow the possibility that some keys will be unset, or that some other keys will have values. We can fix all these problems by using classes specific for our use-cases.

#### State Machines

Finding examples for illegal states in state machines is almost trivial. A good indicator is the presence of many `nullable` attributes. Let's take for example a `Transaction` state machine, that models the exchange of money. Without proper modelling, it could have this shape (simplified, showing only the bare minimum):

```ruby
class Transaction
  # The monetary value of the transaction
  attr_reader(:amount)
  # Can be `:pending`, `:success`, `:failed`
  attr_reader(:status)
  # Only present if the transaction has
  # failed, `nil` otherwise
  attr_reader(:failure_reason)
  # Time at which the Transaction became
  # `:success` or `:failed`, `nil` otherwise
  attr_reader(:completed_at)
end
```

We can see in this example that some states are illegal, for example is `status` is `:success` but `failure_reason` is present, or if `status` is `pending`, but `completed_at` is present.

We can instead express the same object in a way that does not allow these illegal states. We can introduce classes for each possible state, that have the methods valid only for those states. The `Transaction` class will contain only the methods that are transcendental to states.

```ruby
class TransactionPending
  # Nothing perticular here
end

class TransactionSuccess
  attr_reader(:completed_at)
end

class TransactionFailure
  attr_reader(:completed_at)
  attr_reader(:failure_reason)
end

class Transaction
  attr_reader(:money)
  # One of `TransactionPending`, `TransactionSuccess`,
  # or `TransactionFailure`
  attr_reader(:state)
end
```

With this change, only valid states can be represented. It's impossible to have a `TransactionSuccess` with a `failure_reason`, or a `TransactionPending` with a `completed_at`.

#### Incomplete Booleans

In Part 1, I also gave examples of ["Incomplete Booleans"]({% post_url 2020-01-29-affordance-for-errors-pt1 %}#incomplete-booleans) that can be combined to express illegal states. The example given was of two booleans: `inventory_is_managed` and `allow_negative`. The illegal states manifests itself when `inventory_is_managed` is `false`, because any value of `allow_negative` makes no sense. The problem is that there are 3 possible values, but the structure allows us to represent 4 values.

We already know many ways to express a finite-but-larger-than-two number of values. Some people will use `Symbol`, `String`, or `Integer` values to that effect, and validate that the value is part of a finite list.

```ruby
# can also be :managed_lax or :unmanaged
product.inventory_policy = :managed_strict
```

The way I prefer structuring it is by using objects:

```ruby
class InventoryPolicy
end

MANAGED_STRICT = InventoryPolicy.new
MANAGED_LAX = InventoryPolicy.new
UNMANAGED = InventoryPolicy.new

product.inventory_policy = MANAGED_STRICT
```

The first reason is that it makes type checking easier (`is_a?(InventoryPolicy)`). It also makes it possible to give behaviour to the different `InventoryPolicy` if the need arises, by adding methods to the objects. When that becomes the case, it will also be trivial to do dependency injection, for example by creating a test-only `InventoryPolicy`. In general, in the worse case, there is no difference between that and using symbols, and in the best case, it's a much more extensible design.

### Ruby's Visibility Modifiers

Quick Reminder: Some APIs do not meet the user's expectations. Ex: Ruby's visibility modifiers (`private`, `public`, `protected`) have no effect when using `def self.method_name`.

This specific problem is such a common error pains me the utmost, because it is so unnecessary. Approximately 100% of other languages do not have this problem, by having visibility modifiers at the method definition level. In fact, it even works in Ruby, and many large companies have adopted this style. It may be inelegant, but it removes this opportunity for errors altogether:

```ruby
class Foo
  private def baz
    :baz
  end
end
```

My preferred way is the method of singleton class opening, because it's symmetric with normal class definitions:

```ruby
class Foo
  class << self
    def baz
      :baz
    end
  end
end
```

I don't really want to dwell on this too much. The key lesson is: be emphatetic towards your users, understand what they expect, and make sure your APIs match their expectations.

### Ignored Values

Quick reminder: Some APIs return critical values that are easy to ignore. Ex: ActiveRecord's `save` method returns `false` when a record could not be saved.

In Ruby, this affordance is very hard to remove without resorting to exceptions. For APIs where handling errors is critical, it can be enforced by requiring a lambda injection:

```ruby
user.save(on_error: -> (e) { ... })
```

While I do believe that using exceptions for fatal cases is great, I personally dislike using exceptions for cases that should be handled by the user, for many reasons. I'm not sure I need to list them, and I don't think I can do so exhaustively, but in case it's helpful, here are a few reasons:

- Exceptions add a second way methods can return, adding complexity.
- Unlike in some other languages, in Ruby it's impossible to know the entire set of exceptions a method can raise, making it very hard to know which exceptions should be expected, and of those, which should be handled.
- More often than not, an exception raised will leave inconsistent state in the program, in stackframes that were not equiped to deal with it.

As a result, I generally prefer returning object that explain the error that occurred, instead of using exceptions. This makes it important to make sure the returned values are not ignored.

Some languages have solved this problem in interesting ways; some have implemented type systems in which values of certain types cannot produced if they are not consumed. See [substructural type systems on Wiki](https://en.wikipedia.org/wiki/Substructural_type_system).

In [Rust](https://www.rust-lang.org/) specifically, API developers can mark return values as `must_use`, meaning that the return value __must be consumed__ (or checked). Failing to do so would result in a compiler warning. [ATS](http://www.ats-lang.org/) achieves similar results via a concept of "proofs" where unconsumed proofs prevent compilation.

In languages where destructors are controlled, it would be possible to panic if the return value has not been consumed.

In Ruby, you could _technically_ do it by using `define_finalizer(obj) { raise }` and `undefine_finalizer(obj)` :trollface:.

### Missing Coupling

Quick reminder: Some APIs require things that go together, without expressing the coupling. Ex: `Hash` requires the methods `hash` and `eql?` to be coupled, without expressing it. Moreover, the `Object` class defines both methods, making it easy to define one without defining the other, leading to subtle bugs.

The ergonomics around this are hard to get right. One possibility would be to remove `hash` and `eql?` from all objects. This would be a large inconvenience, and I don't think the added correctness would be worth it. Another would be to force reimplementing one method if the other is redefined. I think this second approach would be viable; we wouldn't sacrifice much ergonomics, yet we'd largely prevent this type of error.

This, of course, is where interfaces (or traits) shine, by expliciting the coupling between all the required pieces an API needs.

## Conclusion

In this post, we covered the various affordances given by common Ruby and Rails APIs, as well as common mistakes I've seen in codebases I worked on. We explored ways to spot them and avoid making them in designing our own APIs. I hope you learned something, or that this post helped you put words on ideas you already had, such that you can discuss them with the people on your teams, such that we can all build better APIs, with less affordance for errors. In Part 3, I will be going over how I approached and designed a specific API.
