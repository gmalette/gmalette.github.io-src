---
title: The Ruby Object Model
date: 2019-02-01 13:36:00 +0400
description: "A post in which I try to explain Ruby's Object Model: Objects, Classes, Singleton Classes, Modules..."
categories:
- Ruby
aliases:
- "ruby/2019/02/01/the-ruby-object-model.html"
---

Recently I was called out. I hypothesized that some bad patterns in Ruby are caused by developers misunderstanding Ruby's Object Model, but I had not provided them with any learning resources. In fact, I couldn't find anything satisfying, so I decided to write this post.

**Edit:** I've been sent other resources about Ruby's Object Model:
- [All I'd Wanted to Know about Ruby's Object Model Starting Out...and Mooar!!!](https://www.youtube.com/watch?v=268UU4EpTew) in which Jun Qi Tan explains what I tried to in this post, in a much more eloquent talk.
- [Unraveling Classes, Instances and Metaclasses in Ruby](https://blog.appsignal.com/2019/02/05/ruby-magic-classes-instances-and-metaclasses.html) by Jeff Kreeftmeijer in the excellent Ruby Magic series.
- [Metaprogramming in Ruby](https://pragprog.com/book/ppmetr2/metaprogramming-ruby-2) by Paolo Perrotta has a section on the object model.

---

This post is aimed at developers familiar with Ruby and Object Oriented Programming. I'll assume you have some understanding of what objects and classes are, and that you can read Ruby code without commentary. If you want to learn more about Ruby, or if this post has you itching for more, I must recommend [Ruby Under The Microscope](http://patshaughnessy.net/ruby-under-a-microscope) by Pat Shaughnessy. Chapter 5 goes in-depth in the data structures the Ruby VM (MRI) uses for objects and classes.

If you'd rather skip the wall of text, visit the [**TL; DR**](#conclusion-or-tl-dr). Otherwise, let's get started!

## Objects

You'll hear it often: all values in Ruby are objects, and all expressions return values. It is thus commonly said that "Everything in Ruby is an Object". Let's add some objects to our program. We'll use the this code for the entire section:

```ruby
class Animal
  def initialize
    @affection = 0
  end

  def pet
    @affection += 1
  end
end

class Cat < Animal
  def initialize(name)
    @name = name
    super()
  end

  def pet
    puts("purrr")
    super
  end
end

coco = Cat.new("coco")
coco.pet
# => prints "purrr"
```

Objects in Ruby are just a combination of state—the values of instance variables—, with a class attached to it. The class of an object defines the methods the object has access to. Ruby is easily introspectable, so we can ask it about the state and class of `coco`.

```ruby
# State
coco.instance_variables
# => [:@name, :@affection]

# Class
coco.class
# => Cat
```

`coco`'s state contains two instance variables: `@name` and `@affection`. Its class is `Cat`. Nothing surprising so far.

![Diagram of coco and the Cat class](/images/posts/the-ruby-object-model/fig-1-coco-and-cat.svg)

The word "class" in Ruby can mean multiple things: the instantiatable class which provides behaviour to its instances, the class object, or both. When talking about an object's class, we generally intend to describe the methods defined by that class. For this purpose, we will use the word _behaviour_ from now on.

## Classes

In the [Objects section](#objects) we learned that classes have multiple facets, and one of them is providing the behaviour of instances of that class. This is done by setting up an ancestor chain (also called "inheritance chain"). They can be inspected at runtime using the `ancestors` method. This allows us to clearly see that `coco` contains the behaviours defined by `Cat`, `Animal`, `Object`, and so on.

```ruby
coco.class.ancestors
# => [Cat, Animal, Object, Kernel, BasicObject]
```

The ancestor chain is a linked list. For `Cat`, it looks like this:

![Diagram of Cat's ancestor chain](/images/posts/the-ruby-object-model/simple-ancestor-chain.svg)

Whenever you call the `pet` method on `coco`, the Ruby VM will find the behaviour in `coco`'s class's ancestor chain (`Cat`'s, that is), from left to right: starting with `Cat`, then `Animal`, and so on, until the method is found. When calling `super` within that method, Ruby would find the next ancestor defining a behaviour for the `pet` method, and invoke it, in this case, the one defined in `Animal`.

![Diagram of coco and the Cat, with Cat showing methods](/images/posts/the-ruby-object-model/fig-2-coco-ancestors.svg)

<sub>In the previous figure, ancestors after `Animal` (`Kernel` and `BasicObject`) have been omitted.</sub>

To know which methods are defined on each ancestor in the ancestor chain, use the `instance_methods` method. The boolean argument specifies whether to also include the methods defined by the class's other ancestors.

```ruby
Cat.instance_methods(false)
# => [:pet]
```

In Ruby, classes are also values, which means they are also objects. This is interesting; it means that in addition to defining methods, classes are also the combination of state and a reference to a class.

![Diagram of coco and the Cat class, with classes showing state](/images/posts/the-ruby-object-model/cat-is-a-class.svg)

For this facet (objects which are instances of the `Class` class), we will be talking about _class objects_.

Class objects have a method named `new`, which comes from the behaviour defined by the `Class` class.

```ruby
Cat.method(:new).owner
# => Class
```

The default behaviour of the `new` method is to return a new object: it instantiates the class.

```ruby
coco = Cat.new("coco")
coco.class
# => Cat
```

While this method can be changed by any class in the ancestor chain, this privilege is rarely abused and is frowned upon, but know that one _could_ do it.

For some, it may be surprising that the `Class` class is a subclass of the `Module` class, meaning that all class objects are also Modules. In the next section, we will talk about modules; remember that many things will also apply to class objects.

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

Modules, like classes, have two facets: they are values (thus, objects), and they can define behaviour for other objects. Unlike class objects however, Module objects do not have a method named `new`, and cannot be instantiated directly.

```ruby
Quadruped.new
# Traceback (most recent call last):
#         1: from (irb)
# NameError (undefined method `new' for class `Module')
```

Since `Module`s cannot be instantiated nor subclassed, they need another way to be used as an object's behaviour. This is done by using the `include`, `prepend`, and `extend` methods on Module objects (and on class objects!). I'll describe `include` and `prepend` now, but keep `extend` for later.

### Include

When using `include(A)`, the module (`A`) is inserted in the ancestor chain of the receiver module. Its methods are thus made available on instances of the receiver.

```ruby
class Cat < Animal
  include(Quadruped)
end

# Or the equivalent: `Cat.include(Quadruped)`
```

The previous code will add `Quadruped` as an ancestor to `Cat`, making its methods available to all instances of `Cat`.

```ruby
coco.class.ancestors
# => [Cat, Quadruped, ...]

> coco.feet_count
# => 4
```

Note that `include` adds the module to the ancestor chain *after* the receiver; `Quadruped` appears after `Cat`.

![Diagram of coco and the Cat class](/images/posts/the-ruby-object-model/cat-include-quadruped.svg)

In the [Objects section](#objects), we said that the ancestor chain's order is used to determine which method gets called first. Using another example, we can confirm this:

```ruby
module A
  def foo
    puts("in a")
  end
end

class B
  include(A)

  def foo
    puts("in b")
    super
  end
end

B.new.foo
# prints:
#    in b
#    in a
```

Some people are surprised that this behaviour remains unchanged if we move `include(A)` to after the method definition. The important thing to remember is that an object's behaviour is defined by its class's ancestor chain; moving `include(A)` lower down has no impact on the ancestor chain.

### Prepend

Prepend is very similar to `include` with one very important distinction: it adds the module *before* the receiver.

```ruby
class Dog
  prepend(Quadruped)
end

Dog.ancestors
# => [Quadruped, Dog, ...]
```

This means that behaviours defined by `Quadruped` will have precedence over those defined by `Dog`. The following figure shows the distinction: `prepend` adds the module to the beginning of the ancestor chain, while `include` adds it right after the receiving module.

![Diagram of coco and the Cat class](/images/posts/the-ruby-object-model/prepend-before-include-after.svg)

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

Earlier in this post, I said that objects are the combination of state and a class. This is slightly less than accurate. In fact, two objects instantiated from the same class, are not necessarily of the same type. This is (you probably guessed it from the section title) because of the singleton class. The truth is, every object in Ruby possesses its very own class, of which it is a singleton. You may also have seen "eigenclass"—another word for singleton class—, or "metaclass", which is specifically a Class object's singleton class. In our code, `dora` and `coco` don't have the same class, which explains why `coco` does not have the `number_of_toes` method:

```ruby
coco.number_of_toes
# Traceback (most recent call last):
#         1: from (irb)
# NoMethodError (undefined method `number_of_toes' for #<Cat:0x00007fe5201b0f78 @name="coco">)
```

As such, the "true" class of an object is its singleton class.

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


The following diagram shows the hierarchy of both `coco` and `dora`, with their respective singleton classes; only `dora`'s singleton class defines the `number_of_toes` method.

![Diagram of coco and the Cat class](/images/posts/the-ruby-object-model/coco-and-dora-hierarchy.svg)

Astute readers will correctly understand that, in many cases, an object's singleton class can be elided (omitted) by the virtual machine. In fact, if the VM _did not_ elide most of them, no Ruby program would be able to run. This is because **class objects also have singleton classes**, and Singleton classes are also class objects, which themselves also have singleton classes... and so on recursively. In our previous example, `coco`'s singleton class can be elided since it neither defines methods, nor has state.

Methods defined on a class object's singleton class are sometimes erroneously called "static functions". This nomenclature is misleading; we should avoid it. Using it sets us up for expectations the Ruby VM cannot meet. The reason is, they are not functions, they are nothing more than methods defined on a class object's singleton class. Additionally, constants in Ruby are _not truly constant_, so these methods are not static either.

There are several ways to define methods on a class's singleton class.

```ruby
def Cat.feline? # Like the `number_of_toes` example
  true
end

class Cat < Animal
  def self.feline? # `self` is `Cat` here, so this is exactly the same as the previous example
    true
  end
end

class Cat < Animal
  class << self
    def feline?
      true
    end
  end
end
```

In this last example, `class << self` is a special syntax which opens `self`'s singleton class, in this case, `Cat`'s, similar to how `class Cat` opens the `Cat` class. Everything that can be done in `Cat` to affect its instances can be done within `class << self` to affect the only instance of `Cat`'s singleton class, which is `Cat` itself. (oh god, I'm going to lose people over this overuse of the words "class" and "singleton", aren't I?)

Of these 3 forms however, there is a clear "better way" in terms of simplicity and predictability: `class << self`. Other means will fail the programmer's expectations more often than not, e.g. around method visibility:

```ruby
class Cat < Animal
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

I didn't intend to talk about method visibility in this object model post, but let's just briefly go there, to show an example of Ruby failing to meet expectations. The reason for this unexpected behaviour is that `private` is not a keyword, it is a method. In this case, it will be received by `Cat`, allowing it to make any new method defined on it as private. `Cat`'s singleton class however, does not receive this method call to `private`. As a result, it is not aware that it should change the visibility of methods that will be defined.

Had we used the `class << self` syntax instead, it would have worked as expected:

```ruby
class Cat < Animal
  class << self
    private

    def are_the_best?
      true
    end
  end
end

Cat.are_the_best?
# Traceback (most recent call last):
#         1: from (irb)
# NoMethodError (private method `are_the_best?' called for Cat:Class)
```

---

So far, we've seen Objects, Classes, Modules, and Singleton Classes. *This is a great time to take a break!* In the next section, we'll see `extend`, and singleton class inheritance.

---

## Extend

The `extend` method confuses many people: how is it different from `include` and `prepend`? When should they use one, or the other? To answer that question, consider the following code:

```ruby
module D
  def a
    "Hello? This is D"
  end
end

class E
  include(D)
end

class F
  extend(D)
end
```

Can we predict what behaviour we can expect from `E` and `F`? If we recall from the previous sections, an object's behaviour is provided by its class's ancestor chain. Let's try it:

```ruby
E.ancestors
# => [E, D, Object, ...]
F.ancestors
# => [F, Object, ...]
```

That's interesting! We can expect instances of `E` to have `D`'s behaviour, but not instances of `F`.

```ruby
E.new.a
# => "Hello? This is D"
```

So far so good, `E` does indeed have `D`'s behavour.

```ruby
F.new.a
# Traceback (most recent call last):
#         1: from (irb)
# NoMethodError (undefined method `a' for #<F:0x00007febf409e218>)
```

Success! We correctly predicted this too. If instances of `F` don't have `D`'s behaviour, could the class object `F` have it then? To predict it, we can ask `F`'s singleton class!

```ruby
F.singleton_class.ancestors
# => [#<Class:F>, D, #<Class:Object>, ...]
```

It does have `D`!

```ruby
F.a
# => "Hello? This is D"
```

What does this tell us? It looks like `extend` applies to the *singleton class*'s the same changes which `include` does to the *class*'s. If that were the case, it means we could achieve the same results by calling `include` within the singleton class instead.

```ruby
class D
  class << self
    include(D)
  end
end

D.singleton_class.ancestors
# => [#<Class:D>, D, #<Class:Object>, ...]

D.a
# => "Hello? This is D"
```

It works as predicted!

I should note that this is not _exactly_ true, `extend` and `singleton_class.include` are slightly different. Whenever a module gets added to the ancestor chain of another module, the first module gets a callback on one of three methods: `prepended`, `included`, `extended`. Some side-effects could be expected from these methods, and you may be required to use a specific method to get these side-effects.

```ruby
module Mod
  def self.included(receiver)
    puts "Mod is included by #{receiver}"
  end

  def self.prepended(receiver)
    puts "Mod is prepended by #{receiver}"
  end

  def self.extended(receiver)
    puts "Mod is extended by #{receiver}"
  end
end

module G
  include(Mod)
end

module H
  prepend(Mod)
end

module I
  extend(Mod)
end

# prints:
#   Mod is included by G
#   Mod is prepended by H
#   Mod is extended by I
```

### Extend on All Objects

Did you know that most objects have the same `extend` method? Now that we know that `extend` modifies the behaviour of the receiver (by adding the argument to the receiver's singleton class's ancestor chain), we can predict what happens if we use `extend` directly on any object\*.

```ruby
module OwnedByGuillaume
  def owner
    "Guillaume"
  end
end

coco.extend(OwnedByGuillaume)

coco.singleton_class.ancestors
# => [..., OwnedByGuillaume, ...]

coco.owner
# => "Guillaume"
```

As predicted, the receiver (`coco`) has the behaviour offered by `OwnedByGuillaume`.

<sub>\*although calling `extend` on any object is perfectly valid Ruby, I urge you to either not use it, or use it very parsimoniously</sub>

## Singleton Class Inheritance

In Ruby, singleton classes are subclasses of their instance's original class. Heh, this is a bit hard to follow, let's do an example.

```ruby
coco.singleton_class.superclass
# => Cat
```

In the previous code, we can see that `coco`'s singleton class is a subclass of `Cat`, like `Cat` is a subclass of `Animal`. For class objects, their singleton class are subclasses of their superclass's singleton class: `Cat`'s singleton class is a subclass of `Animal`'s singleton class, which is itself a sublcass of `Object`'s singleton class, and so on.

```ruby
Cat.singleton_class.ancestors
# => [#<Class:Cat>, #<Class:Animal>, #<Class:Object>, ...]
```

The next figure illustrates the relationships between class objects, their singleton classes, and their respective superclasses.

![Diagram of coco and the Cat class](/images/posts/the-ruby-object-model/class-singleton-class-inheritance.svg)

Modules, however, are not inherited at both levels (class and singleton class behaviours). Instead, the programmer picks which ancestor chain will be impacted by using either `prepend` or `include` (class), or `extend` (singleton class).

```ruby
module Included
end

module Extended
end

class L
  include(Included)
  extend(Extended)
end

class M < L
end

M.ancestors
# => [M, L, Included, ...]

M.singleton_class.ancestors
# => [#<Class:M>, #<Class:L>, Extended, ...]
```

As you can see, while `L` appears in `M`'s ancestors, `Extended` does not, and while `L`'s singleton class appears in `M`'s singleton class's ancestors, `Included` does not.

## Conclusion (or TL; DR)

Here's what I hope you take away from this post:
1. Objects are the composition of state and a reference to a class.
2. The "real" class of any object is its singleton class.
3. Class are just normal objects, and their class named `Class`.
4. `prepend` and `include` affect the behaviour offered by the receiver.
5. `extend` affects the behaviour of the receiver, through its singleton class.

