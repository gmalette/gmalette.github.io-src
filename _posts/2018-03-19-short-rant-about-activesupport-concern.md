---
layout: post
title: Short Rant about ActiveSupport::Concern
date: 2018-03-19 21:00:00 +0400
categories:
- Ruby
---

This is a copy/paste of a comment I made on an internal pull request. I had been tagged on it because on my hard stance against `ActiveSupport::Concern`, asking me to clarify my objections.
I’ve reworded my reply where it made sense.

Here is the code on which I was asked to comment:

```ruby
module Component
  module AcquiredInformation
    extend ActiveSupport::Concern

    included do
      class << self
        class_attribute :component

        method = method(:component)
        self.component =
          Component::Discovery.from_method(method) ||
          Component::Discovery.fallback_component
      end
    end

    module ClassMethods
      def inherited(klass)
        klass.component =
          Component::Discovery.from_caller(caller) ||
          Component::Discovery.fallback_component

        super
      end
    end
  end
end
```

## My Reply

My main problem is that `ActiveSupport::Concern` (later `AS::C`) encourages bad object-oriented design, while simultaneously having programmers forfeit their understanding of the object model.

I don’t want to dwell on what Ruby’s object model is; it’s the topic of an other discussion and most of you already know it. One thing is important to understand is that for `class MyClass`, the instance returned by `MyClass.new` and the class `MyClass` itself both are proper objects (later denoted instance and class, respectively). Great care must be applied in designing both, and both must not be confused for the other.

### OO Design
`AS::C` conflates the class and the instance design by encouraging a single module that acts upon both.

In all cases where instances are used, the class itself is a factory. Adding more functionality to the class must take that fact into account to not violate the [SRP](https://en.wikipedia.org/wiki/Single_responsibility_principle).

Furthermore, for those that don’t know me, here’s one thing you must know: I fundamentally believe that naming things correctly is a prerequisite for good design. Naming concepts and objects is the best way we have to communicate meaning to one another, it’s basically one of the greatest decomposition technique at our disposal.

`AS::C` robs us of this ability to name the things by imposing a `ClassMethods` name to add features to the class object. The `ClassMethods` suggests that the module is just a bag of methods without necessitating a common, named concept.

Consider the `CarrierService` class (ed: an internal class), which, let’s remind ourself, is a factory generating `CarrierService` instances. It apparently has many other roles, of which most aren’t properly named.

```
> CarrierService.singleton_class.ancestors
[
 IdentityCache::ShouldUseCache::ClassMethods,
 IdentityCache::QueryAPI::ClassMethods,
 IdentityCache::ConfigurationDSL::ClassMethods,
 IdentityCache::CacheKeyGeneration::ClassMethods,
 IdentityCache::BelongsToCaching::ClassMethods,
 EncryptableAttributes::ClassMethods,
 ...
```

What does it mean to say that `CarrierService` is a `IdentityCache::ShouldUseCache::ClassMethods`? Is it a grab-bag of methods related to `IdentityCache`, or is it a behaviour compatible with the factory role?

### Object Model
`AS::C` has programmers forfeit their understanding of what it means for an object to either `include` or `extend` a module, by having them always `include`, and use the `self` module to act upon the instance’s class, and `ClassMethods` module to act upon the class’s class, even in cases where only the latter is necessary.

Taking the code-at-hand as an example. The only behaviour we want to change it the class’s; the instance should remain unmodified. To do so, the code using `AS::C` does many things:

- Define a `Component::AcquiredInformation` module and makes it inherit from `AS::C`
- Defines a `ClassMethod` module
- Upon defining instances as inheriting from `Component::AcquiredInformation`, it reopens that instances’s class to define an attribute on it
- Upon defining a class that inherits from `Component::AcquiredInformation::ClassMethods`, it modifies the class to overwrite the attribute

Now consider a module built for the same purpose but that doesn’t conflate instance and class. It could use the `extended` hook to define the attribute, and the `inherited` hook to overwrite the attribute.

```ruby
module Component
  module AcquiredInformation
    def self.extended(base)
      base.class_attribute :component

      method = base.method(:component)
      base.component =
        Component::Discovery.from_method(method) ||
        Component::Discovery.fallback_component
    end

    def inherited(klass)
      klass.component =
        Component::Discovery.from_caller(caller) ||
        Component::Discovery.fallback_component

      super
    end
  end
end
```

I agree that this still isn’t the greatest OO design, but at least it’s radically simpler.

Because I struggle to find a single case in which using `AS::C` actually provides a better design than not using it, I am a hardliner of never using it.

[Comment or React](https://github.com/gmalette/gmalette.github.io/pull/4)
