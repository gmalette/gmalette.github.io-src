---
layout: post
title: The Ruby Object Model
date: 2019-02-01 13:36:00 +0400
categories:
- General
---

# Ruby's Object Model

Recently I was called out. I hypothesized that some bad patterns in Ruby are caused by developers misunderstanding Ruby's Object Model, without providing the learning resources. And so, here it is.

---

This post is not aimed at newcomers to OO languages, and I'll assume you have some understanding of what objects and classes are. 
If you want to learn more about Ruby, or if this post has you itching for more, I must recommend [Ruby Under The Microscope](http://patshaughnessy.net/ruby-under-a-microscope) by Pat Shaughnessy.

Without further ado, let's get started.

## Objects

```ruby
class Animal
end

class Cat < Animal
  def initialize(name)
    @name = name
  end

  def pet
    puts("purrr")
  end
end

coco = Cat.new("coco")
```

You'll hear it often: all values in Ruby are objects, and all expressions return values. It is thus commonly said that "Everything in Ruby is an Object". 

Objects in Ruby are just a bag holding state, with a class attached to it.

```ruby
coco.class
# => Cat
```

[Drawing of bag with @name = coco state and Cat class]

The word "Class" in Ruby can mean multiple things. To clearly distinguish the different facets of Classes, I will avoid using the word "Class" whenever possible, and introduce precise vocabulary for each facet. When talking about objects we generally intend to describe the methods an object has. For this purpose, we will use the word _"behaviour"_ from now on. As such, the class of an object provides that object's behaviour.

## Classes

Ruby Classes play many roles. We showed previously that one role is providing the behaviour of objects. This is done by setting up an ancestor chain (also called "inheritance chain"). They can be inspected at runtime using the `ancestors` method:

```ruby
coco.class.ancestors
# => [Cat, Animal, Object, Kernel, BasicObject]
```

Whenever you call the `pet` method on `coco`, the Ruby VM will find the behaviour in `coco`'s class (`Cat`) ancestor chain, from left to right: starting with `Cat`, then `Animal`, and so on, until the method is found.

To know which method are defined on each piece in the inheritance chain, you use the `instance_methods` method.

```ruby
Cat.instance_methods(false)
# => [:pet]
```

In Ruby, Classes are also values, which means they are also objects. This is interesting; it means that Classes are also a bag holding state, with a reference to a class. Classes are, in fact, instances of the `Class` class.

```ruby
Cat.class
# => Class
```

For this role (objects which are instances of the `Class` class), we will be talking about _"Class object"_.

Class objects have a method named `new`, which comes from the behaviour defined by the `Class` class.

```ruby
Cat.method(:new).owner
# => Class
```

The default behaviour of the `new` method is to return a new object whose class is the receiver–instantiate the class.

```ruby
coco = Cat.new("coco")
coco.class
# => Cat
```

While this method can be changed in by any class in the ancestor chain, this privilege is rarely abused, and would be frowned upon, but one _could_ do it.

For some, it may be surprising that the `Class` class is a subclass of the `Module` class, meaning that all Class objects are also Modules.

```ruby
Cat.class.ancestors
# => [..., Module, ...]
```

## Modules

```ruby
module Quadruped
  def feet_count
    4
  end
end
```

Modules, like classes, are both values (and objects), as well as ancestors to other objects. Unlike Class objects, Module objects do not have a method named `new`, and cannot be instantiated directly.

```ruby
Module.new.method(:new)
Traceback (most recent call last):
        1: from (irb):1:in `method'
NameError (undefined method `new' for class `Module')
```

Since `Module`s cannot be instantiated nor subclassed, they need another way to become an object's ancestor. This is done by using the `include` and `extend` methods on `Module` objects (and by extension, `Class` objects which also are descendants of `Module`).

When using `include`, the module is inserted in the ancestors of an object. It's methods are made available on it's descendants. The `extend` method will be described later.

```ruby
class Cat
  include(Quadruped)
end

# Or the equivalent: `Cat.include(Quadruped)`
```

The previous code will add `Quadruped` as an ancestor to `Cat`, making it's methods available to all instances of `Cat`

```ruby
coco.class.ancestors
# => [Cat, Quadruped, ...]

> coco.feet_count
# => 4
```

## Singleton Classes

```ruby
dora = Cat.new("dora")
# Dora has mitten paws
def dora.number_of_toes
  22
end

dora.number_of_toes
# => 22
```

Earlier in this post, I said that objects are the combination of state and a Class. This is slightly less than accurate. In fact, two objects instantiated from the same class, are not necessarily of the same type. This is (you probably guessed it from the section title) because of the singleton class. The truth is, every object in Ruby possesses it's very own class, of which it is a singleton. You may also have seen "metaclass" or "eigenclass", both terms used to describe the singleton class, which shouldn't be used anymore in Ruby. In our code, `dora` and `coco` don't have the same class, which explains why `coco` does not have the `number_of_toes` method:

```ruby
coco.number_of_toes
Traceback (most recent call last):
        1: from (irb)
NoMethodError (undefined method `number_of_toes' for #<Cat:0x00007fe5201b0f78 @name="coco">)
```

As such, the "true" class of an object is it's singleton class.

```ruby
dora.singleton_class.instance_methods(false)
# => [:number_of_toes]
dora.singleton_class.ancestors
# => [#<Class:#<Cat:0x00007fe52016e3a8>>, Cat, Quadruped, ...]
```

The weird `#<Class:#<Cat...>>` up there means "the singleton class of the Cat object `0x00007fe52016e3a8`", which is `dora`:

```ruby
dora
# => #<Cat:0x00007fe52016e3a8 @name="dora"> # notice the `0x00007fe52016e3a8`
```

Astute readers will correctly understand that, in many cases, an object's singleton class can be elided by the virtual machine. In fact, if the VM _did not_ elide most of them, no Ruby program would be able to run. This is because **Class objects also have singleton classes**. Singleton classes are also Class objects, which themselves also have singleton classes... and so on recursively.

Methods defined on a Class object's singleton class are sometimes erroneously called "static methods". This nomenclature is misleading; we should avoid it. Using it sets us up for expectations the Ruby VM cannot meet. The reason is, they are behaviour defined on the Class object's singleton class, and the Class object is like all other objects: it has state.

There are several ways to define methods on a class's singleton class.

```ruby
def Cat.feline? # Like the `number_of_toes` example
  true
end

class Cat
  def self.feline? # `self` is `Cat` here, so this is exactly the same as the previous example
    true
  end
end

class Cat
  class << self
    def feline?
      true
    end
  end
end
```

In this last example, `class << self` is a special syntax which opens `self`'s singleton class, in this case, `Cat`'s, similar to how `class Cat` opens the `Cat` class. Everything that can be done in `Cat` to affect it's instances can be done within `class << self` to affect the only instance of `Cat`'s singleton class, which is `Cat` itself.

Of these 3 forms however, there is a clear winner in terms of which should be used: `class << self`. Other means will fail the programmer's expectations more often than not, e.g. around method visibility:

```ruby
class Cat
  private

  def self.are_the_best?
    true
  end
end
```

One would expect the `are_the_best?` method defined by `Cat`'s singleton class to be private, but it is not.

```ruby
Cat.are_the_best?
# => true
```

I didn't intend to talk about method visibility in this object model post, but let's just briefly go there. The reason for this unexpected behaviour is that `private` is not a keyword: it is a method. In this case, it will be received by `Cat`, allowing to make any methods defined to it's behaviour as private. `Cat`'s singleton class however, does not receive this method call. It is not aware that it should change the visibility of methods that will be defined.

Had we used the `class << self` syntax instead, it would have worked as expected:

```ruby
class Cat
  class << self
    private

    def are_the_best?
      true
    end
  end
end

Cat.are_the_best?
Traceback (most recent call last):
        1: from (irb)
NoMethodError (private method `are_the_best?' called for Cat:Class)
```
