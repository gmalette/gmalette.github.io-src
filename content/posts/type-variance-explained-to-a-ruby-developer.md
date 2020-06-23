---
title: Type Variance Explained To a Ruby Developer
date: 2020-06-21 20:00:00 +0400
categories:
- Ruby
---

A few months ago, I found myself trying to explain type variance to a coworker whose experience is mainly Ruby. Dynamically typed languages such as Ruby don't ask the developers to specify the type variance. I found that while they develop an instinct of what usages are acceptable and which aren't, these developers don't codify it in the same way developers in statically typed languages do. In trying to explain, I had great difficulty driving home the difference between covariance and contravariance. Their good instinct made it hard for me to fill the gap in their understanding. After a while, I remembered the way Simon Génier first explained it to me, and that way was a success.

This post is how I would explain type variance to a Ruby developer, without throwing the entire Computer Science manual at them. Throughout the post, we will suppose a typical `Animal` type hierarchy, with `Dog` and `Cat` as subtypes.

```ruby
class Animal
  def age
    @age
  end
end

class Cat
  def purr
    puts "Purr"
  end
end

class Dog
  def fetch(ball)
    puts "Fetching the ball"
  end
end
```

## Type Variance

In computer programming, we can divide types into simple types (ex: `Integer`, `String`, `Symbol`), and types composed of other members (ex: `Array`, `Hash`, `Enumerable`). This composition is often called _parameterization_. An `Array<Integer>` (an `Array` of `Integer`) said to be parameterized by the `Integer` type.

Type variance allows us to describe the relationship between the composed type and its members. Intuitively, we know that `Enumerable<Cat>` is a subtype of `Enumerable<Animal>`, but it's harder to understand that `Logger<Animal>` is a subtype of `Logger<Cat>`. I'll explain why that's the case, but first, let's talk about Arrays.

## The Curious Case of Arrays

In dynamically typed languages as well as in languages that added generics after-the-fact, Arrays are weird.

Let's examine a `print_age` method takes an Array of Animals and prints their age.

```ruby
# @params animals Array<Animal>
def print_age(animals)
  animals.each { |a| puts(a.age) }
end
```

The `print_age` method can be called an array containing any animal:

```ruby
# Array<Dog>
dog_array = [Dog.new, Dog.new]
print_age(dog_array)

# Array<Cat>
cat_array = [Cat.new, Cat.new]
print_age(cat_array)

# Array<Animal>
animal_array = [Dog.new, Cat.new]
print_age(animal_array)
```

We can also have another method named `adopt`  that adds animals to the list:

```ruby
# @params into_animals Array<Animal>
def adopt(into_animals)
  into_animals.push(Dog.new)
end
```

Now can you `adopt` in the same arrays? Let's see.

```ruby
# Array<Animal>
animal_array = [Dog.new, Cat.new]
adopt(animal_array)
# animal_array is still an `Array<Animal>`, no problem

# Array<Cat>
cat_array = [Cat.new, Cat.new]
adopt(cat_array)
# OH NO!!! [Cat, Cat, Dog]. This is no longer an `Array<Cat>`
```

The answer is no: `adopt` cannot use the same arrays as `print_age`.  However, the `adopt` has a property that `print_age` did not have, it accepts `Array<Object>`.

```ruby
# Array<Object>
object_array = [Dog.new, Object.new]
adopt(object_array)
# This is fine too! [Dog, Object] is a valid `Array<Object>`.
```

We can also notice that we cannot send `object_array` into `print_age`, because `Object` doesn't define the `age` method.

What can we learn from this? Depending if you read from the array or write to it, the type doesn't behave the same.

## Sources are Covariant

Composed types from which you can exclusively read or take stuff are called sources. Examples include `Enumerable` and `Reader`. These become more specific as their type members become more specific. As a result of `Dog` being a subtype of `Animal`, `Enumerable<Dog>` is a subtype of `Enumerable<Animal>`; you can pass an `Enumerable<Dog>` to any method that expects an `Enumerable<Animal>`.

## Sinks are Contravariant

Sinks are the opposite of sources: they're types to which you can exclusively write or put stuff in. Examples include `Logger` or `Writer`. Perhaps counter-intuitively, these become more specific as their type member become less specific! That means that __`Logger<Animal>` is in fact a subtype of `Logger<Dog>`__ (because `Logger<Animal>` will be able to log for any `Dog` as well.

## Read And Write, Put And Take

What if a method needs to do both? Read and write, put and take? It would appear that they are at the intersection of covariance and contravariance... and that's exactly right: they're __invariant__. No subtyping is possible.

In an ideal world, `Array<Dog>` is neither a subtype nor a supertype of `Array<Animal>`. In practice, even if `Array`, `HashMap` and other read/write containers __should be invariant__, many languages will consider them covariant because of historic reasons. 

## Conclusion

Hopefully this piece will help you understand type variance better, or explain it to a curious coworker.
