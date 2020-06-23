---
title: Refactoring Common ActiveSupport::Concern Patterns
date: 2019-02-25 20:00:00 +0400
description: "Following my Rant on ActiveSupport::Concern, I've been asked how to refactor the code to make it stop using it, or write new code without it. I thought I'd share the patterns that have worked for me so far."
categories:
- Ruby
aliases:
- "ruby/2019/02/25/refactoring-common-activesupport-concern-patterns.html"
---

Following my [Short Rant on `ActiveSupport::Concern`]({{< ref "posts/short-rant-about-activesupport-concern" >}}), I've been asked how to refactor the code to make it stop using it, or write new code without it. I thought I'd share the patterns that have worked for me so far.

## What is `ActiveSupport::Concern`?

`ActiveSupport::Concern` is a module which other modules can `extend` to get its features. It allows for easily (note: _easy_, not _simple_) applying modifications to the receiver module and its singleton class, as well as triggering hooks when the module gets `include`'d.

The following code contains all the marks which identify `ActiveSupport::Concern` (later `AS::C`):

```ruby
module Concern
  extend(ActiveSupport::Concern)

  included do # 1
    puts("Hello from #{self}")
  end

  module ClassMethods # 2
    def baz
    end
  end

  class_methods do # 3
    def bar
    end
  end
end

module ReceiverModule
  include(Concern) # 4
end
```

The previous code uses all the `AS::C` features, which are (annotated in the code):
1. Setting up a callback, for when the module is included (4). The callback is `class_eval`'d on the receiver module, in our case `ReceiverModule`.
2. Open (or reopen) the `ClassMethods` module. When `Concern` is included in `ReceiverModule`, `AS::C` will also do `ReceiverModule.extend(Concern::ClassMethods)`.
3. Same as 2, different style.
4. `ReceiverModule` uses our `Concern` module. The result is that `ReceiverModule`'s singleton class will have both `baz` and `bar`, and the code will print `"Hello from ReceiverModule"`.

Another pattern you may see is the following:

```ruby
module AnotherConcern
  include(Concern)
  extend(ActiveSupport::Concern)

  def toto
  end
end
```

While `AnotherConcern` appears to not use any of `AS::C`'s features, it's "reexporting" the hooks from `Concern`. This is why I often say that metaprogramming breeds metaprogramming: `Concern`'s use of metaprogramming forced `AnotherConcern` to also use metaprogramming, otherwise it would not be able to offer `Concern`'s functionality.

Now that we've identified the telling signs of `AS::C` (`included` callback, `ClassMethods` module, and `class_methods` block), we can go about refactoring away from it!

## Useless Extend

Believe it or not, extending `AS::C` for no reason nor benefit is _extremely_ common. It also has a nice property of being trivial to remove.
You'll be able to tell that it's uselessly extended when the receiver module has none of its telling signs, and does not include any other modules which use `AS::C`.

If the module has none of these, it's safe to remove `ActiveSupport::Concern`, given this code:

```ruby
module ConcernModule
  extend(ActiveSupport::Concern)

  def my_method
  end
end
```

You can apply these changes:

```diff
module ConcernModule
- extend(ActiveSupport::Concern)

  def my_method
  end
end
```

## `ClassMethods` only, please

Modules that use `ActiveSupport::Concern` with the only benefit of having the receiver subsequently `extend` the module's `ClassMethods`.
You'll notice this pattern when the only interesting piece of code in the module is the `ClassMethods` module.
I suspect this happens when folks find it hard to understand the difference between `include` and `extend`–which is admittedly quite hard to make–, leading them to cargo-paste this code from elsewhere.

_Sidenote: to learn more about `include` and `extend`, I suggest reading my previous blog post:_ [The Ruby Object Model]({% post_url 2019-02-01-the-ruby-object-model %}).

The code you'd be looking at would look something like this:

```ruby
module ConcernModule
  extend(ActiveSupport::Concern)

  module ClassMethods
    def a_class_method
    end
  end
end

module ReceiverModule
  include(ConcernModule)
end
```

The good news is that this is also trivial to fix. You can remove the `ClassMethods` module and move its behaviour to `ConcernModule`. Then replace `include(ConcernModule)` with `extend(ConcernModule)`, so that the behaviour is applied on the receiver's singleton class instead of the receiver itself.

```ruby
module ConcernModule
  def a_class_method
  end
end
```

```diff
module ReceiverModule
-  include(ConcernModule)
+  extend(ConcernModule)
end
```

Another variant of this is to use a `class_methods` block instead of the `ClassMethods` module. The same refactor applies.

## Callback Methods

Sometimes, `AS::C` is used for its `included` callback only. Sometimes, it will coincide that the block behaviour can be implemented in another, simpler way.

```ruby
module TestHelper
  extend(ActiveSupport::Concern)

  included do
    teardown do
      clean_up_after_tests
    end
  end
end
```

This previous example uses the `included` callback to call the `teardown` method in Minitest. Instead of this, we can _define_ a teardown method.

```ruby
module TestHelper
  def teardown
    clean_up_after_tests
    super
  end
end
```

This will unfortunately only work if the hooks can also be implemented using methods.

## Metaprogramming the Metaprogramming

Another variation of the previous pattern is defining the methods in the hooks, for no clear reasons. This means that every time `ModuleDefiningMethods` is included, it will define new versions of the `foo` and `bar` methods. Every time I've seen this, it was unwanted.

```ruby
module ModuleDefiningMethods
  extend(ActiveSupport::Concern)

  included do
    def foo
    end

    def bar
    end
  end
end
```

In this case, we can simply remove the `included` block but keep the contents. The modules which `include` `ModuleDefiningMethods` don't even have to change.

```ruby
module ModuleDefiningMethods
  def foo
  end

  def bar
  end
end
```

Now that we've gotten our hands dirty refactoring the easy cases, let's see what we can do about the more involved ones.

## Coupling Class and Instances

I've often seen this kind of offense, where one tries to add behaviour to the instances of a class, and other behaviour (often configuration) to the class itself. Example:

```ruby
module ParameterFiltering
  extend(ActiveSupport::Concern)

  included do
    class_attribute(:allowed_params) # 1
  end

  def filtered_params # 2
    ParameterFilter.new(self.class.allowed_params).filter(params)
  end

  module ClassMethods
    def allow_params(*param_names) # 3
      self.allowed_params = param_names
    end
  end
end

class FooController
  include(ParameterFiltering)

  allow_params(:foo, :bar, :baz)

  def create
    Foo.create(filter_params)
  end
end
```

What this code does:
1. Sets up a class attribute named `allowed_params`.
2. Defines an instance method to return the filtered params, based on which are allowed.
3. Defines a class-level method to set the `allowed_params` attribute in an "idiomatic macro".

This code unnecessarily couples the class (`FooController`) with its instances. Additionally, the overuse of metaprogramming in this code makes it harder to change, say, how parameters are actually filtered.

An easy change we can make to keep the code as convenient while reducing the complexity is to move `allowed_params` from a "macro" to a method. The resulting code will be slightly less idiomatic, but much simpler. I think that's a good thing.

```ruby
module ParameterFiltering
  def filtered_params
    ParameterFilter.new(allowed_params).filter(params)
  end

  def allowed_params
    raise NotImplementedError # Signal to your users that they must implement this method
  end
end

class FooController
  include(ParameterFiltering)

  def create
    Foo.create(filter_params)
  end

  private

  def allowed_params
    [:foo, :bar, :baz]
  end
end
```

Another option we have would be to make the `allowed_params` an argument to the `filter_params` method. This would increase the convenience, as callsites can filter on different arguments.

Finally, yet another refactor would be to get rid of the `ParameterFiltering` module entirely. Instead, we can instantiate a `ParameterFilter`, bind it to a constant, and reuse the same instance of `ParameterFilter` for all instances of the controller (provided it doesn't have mutable state). Object Oriented Programming has taught us favour composition over inheritance, and this approach subscribes to that philosophy.

```ruby
class FooController
  ParamFilter = ParameterFilter.new([:foo, :bar, :baz])

  def create
    Foo.create(ParamFilter.filter(params))
  end
end
```

Unfortunately, this option is too often overlooked, even though it has powerful arguments in its favour. In our case, since the instance of `ParameterFilter` can be reused, we'll be allocating less objects. The resulting code is also both simpler and easier to understand. Finally, we'll easily be able to compose objects to obtain the behaviour we want, a task which would be much harder if we used any of the previous patterns. Ex:

```ruby
class FooController
  ParamFilter =
    RejectLongParameterValues.new(ParameterFilter.new([:foo, :bar, :baz]), max_length: 30)

  def create
    Foo.create(ParamFilter.filter(params))
  end
end
```

## Class Macro, Instance Functions

This is a slight variation on the "Coupling Class and Instances" pattern, except that the main behaviour of the module is adding a "macro" to define instance methods. This pattern tends to happen when the defined method becomes long, and developers want to extract parts of it into functions. Having nowhere obvious to put the extracted methods, we resort to also using instance methods (it's the easiest).

```ruby
module ValidatedAttribute
  extend(ActiveSupport::Concern)

  module ClassMethods
    # The `attribute` method is a "macro". The `valid?` method has been extracted from it.
    def attribute(name)
      define_method(name) do
        # work
        valid_attribute?(thing)
        # work some more
      end
    end
  end

  def valid_attribute?(attribute)
    # work
  end
end
```

Refactoring this is quite easy in most cases: we can make them functions. This will allow us to get rid of the `ClassMethods` module. It also has nice benefits: the `valid_attribute?` function becomes much easier to test, we don't risk clobbering other methods with the same name, and the instances have one less level of inheritance.

```ruby
module ValidatedAttribute
  # The `attribute` method is a "macro". The `valid?` method has been extracted from it.
  def attribute(name)
    define_method(name) do
      # work
      ValidatedAttribute.valid_attribute?(thing)
      # work some more
    end
  end

  class << self
    def valid_attribute?(attribute)
      # work
    end
  end
end
```

## A Pattern I Don't Know How To Refactor

There are still a few usage patterns of `AS::C` which I don't know how to easily refactor, they would require fundamental changes. These have always been around adding another layer metaprogramming, often under the guise of convenience. Here's an example, involving metaprogramming and `ActiveRecord`. The `CanBeActive` module adds a scope to the class, and a method to the instances. While I don't consider this to be good practice, I don't see a more convenient way to achieve the same results.

```ruby
module CanBeActive
  extend(ActiveSupport::Concern)

  included do
    scope(:active, -> { where(active: true) })
  end

  def inactive?
    !!active?
  end
end

class User < ActiveRecord::Base
  include(CanBeActive)
end
```

I personally don't worry too much about the byte size of the codebase, and I would probably skip the module in this case, and implement the same logic in all classes which use it, unless that is, the `CanBeActive` is actually a concept my application uses, ex: if I had a `ActivationController` which accepts _any_ `CanBeActive` resource. That is, I would add this module not as a way to share code, but as a way to identify a common concept.

## Conclusion

Hopefully this post has proven that removing occurrences of `ActiveSupport::Concern` is generally easy, if not trivial. Often it can be removed without further changes. Other times, it just requires understanding the difference between `include` and `extend`, or knowing if methods can be used instead of "macros". Sometimes it's about getting to a better OOP design, but mostly, it's about combining objects rathern than piling more layers of metaprogramming.

And sometimes, you just have to give up and add another layer of metaprogramming, in which case feel free to use `AS::C`.

[Comment or React](https://github.com/gmalette/gmalette.github.io/pull/6)
