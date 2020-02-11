---
layout: post
title: My Preferred Coding Style
date: 2020-02-09 20:00:00 +0400
description: __CHANGE_ME__
categories:
- Ruby
---

I use a Ruby coding style diverging from the community [Ruby Style Guide](https://rubystyle.guide/) (later called the RSG). This style has been chosen deliberately, and is often met with criticism from newcomers on my team, especially experienced Ruby programmers (rightly) claiming it isn't idiomatic. 

This post will lay out my style guide, both the differences and the similarities with the RSG. I will outline the decisions behind the divergences from the RSG, and I will reinforce some of their recommendations.

~~_Note: If your team uses an auto-formatting tool that formats code on every save, changing style is probably irrelevant._~~

~~Like for all computer all things, my choices are heavily influenced by Simon Génier, who helped shape my way of thinking.~~

## Goals

The main goal of code formatting styleguides is to make the code uniform and predictable.
They makes the code easier (but not simpler) to read: humans are masters at pattern-matching, 
and consistent and predictable code formatting helps us find patterns. 
When they cover the entirety of the language, they make the code more pleasant to read, by removing the opportunity for [bikeshedding](https://en.wiktionary.org/wiki/bikeshedding).
They also make the process writing code both easier and simpler. A proper guide can remove a lot of cognitive load.

I tried to adopt a styleguide optimizing for a few features:

- Pleasant. 

  It's acceptable that a new style requires a settlement period, but it must be pleasant after long-term adoption.
 
- Simple. 

  There must be a low amount of rules. Rules should not contradict one-another. If possible, it simplifies the language.

- Universal. 

  Everything is covered, nothing is ambiguous.
  
- Easy. 

  A developer shouldn't need a lot of prior knowledge to adopt a styleguide.
  
- Minimize churn.

  Changes should require the minimal commits possible.

## My Style Guide

### Parentheses

*Always use parentheses when calling a method with arguments.*

This is in line with the Ruby Styleguide to [use parentheses](https://rubystyle.guide/#method-invocation-parens), except that they also recommend [not using parentheses for internal DSL](https://rubystyle.guide/#method-invocation-parens-internal-dsl). Disregarding that second part makes the style simpler and easier to adopt, as one doesn't have to know what counts as a DSL, and avoids arbitrary opinions on whether something is or isn't a DSL. Always using parentheses also produces higher correctness. Read [Revoking the (Parentheses) Privilege]({% post_url 2018-12-03-revoking-the-parentheses-privilege %}) to know more.

### Indentation

Nothing radical here: indent everything by **two spaces**, relative to the expression it's part of. This means that no line will be indented by more than two spaces more than the previous line.

```ruby
# good

a = foo(
  :a, 
  :b
)
```

I've seen styles that aligned list items with the first item, or `when` at the same level as the `case`. As we'll see later, it's unlikely that you'll end up writing this code anyway.

```ruby
# bad
a = foo(:a,
        :b
) 

# bad
a = case foo
    when 1
      puts "1" 
    end
```

This is obviously bad, as it doesn't minimize the churn. Renaming the `a` binding or the `foo` method would force us to reindent the `:b` line too.

### Line Breaks

Since the way I split expressions to avoid long lines is where I diverge the most from the idiomatic Ruby style, and since this is the cornerstone of my style, it makes sense to explain it early on.

I tend towards a functional style of programming, and I don't shy away from long expressions, provided they are clear. I don't introduce intermediate bindings–nor extract single-use methods–for the sake of it. 

~~This section will be better explained by working through an example, which we'll split as we go.~~

~~~
```
results = File.readlines(root.join(index_file_name))
  .map { |path_to_download| uri = URI(CDN.location(east).with_path(path_to_download)) }
  .map { |uri| result = DownloadManager.download(uri); puts "Downloading file #{uri}: #}
```
~~~

---

I have determined a set of rules that allow me to predictably split long expressions, achieving predictable results.

This may be somewhat contrarian, but I don't particularly care about the indenting level. Where some people will have fast rules like "no more than 4 levels of indenting", I don't. I focus on the width of the text, and I try to keep my code either single-line, or column width, basically less than somewhere around 40 characters. 

Also, while most people will be eager to split their expressions on the `(` or `.` characters, the first character I split on is `=` (assignment). 

#### First Token: `=`

Take for example this code:

```ruby
paths = manifest.static_assets(domain: application.domains.primary).images
```

We can split it a number of ways.

On the `(` token:

```ruby
paths = manifest.static_assets(
  domain: application.domains.primary
).images
```

On the `.` token:

```ruby
paths = manifest
  .static_assets(domain: application.domains.primary)
  .images
```

On the `=` token:

```ruby
paths =
  manifest.static_assets(domain: application.domains.primary).images
```

For most Ruby developers, the `(` and `.` options probably feel most familiar, but I argue we should use the 3rd option: breaking on `=` first.
Generally, binding names aren't very long, and this doesn't buy a lot of real estate, but I choose to break on `=` first because of the other properties it offers: it allows preserving `if/else`, `case/when` , `do/end`, and similar constructs's indentation.

```ruby
message = 
  case error
  when NameMissing
    "name is missing"
  when DatabaseUnavailable
    "database is unavailable"
  end

# or

result = 
  if user.valid?
    :success
  else
    :error 
  end
```

It also allows chaining on them. This example will be controversial, and I can already hear the more squeamish yell that chaining on `end` is unacceptable. To those, I ask: "why not?".

```ruby
# good
paths = 
  manifest.static_assets.images.map do |img|
    CDN.location(location).path(image)
  end
  .map do |path|
    DownloadManager.download(path)
  end
  .to_set
```

Had we not split on `=` first, we'd be left with two options: using `end.method` (not so bad), or using `end\n{space}{space}.method`:


Using `end.method` is fine in terms of looks, but it makes it harder to see the start of the sub-expression.

```ruby
# bad
paths = manifest.static_assets.images.map do |img|
    CDN.location(location).path(image)
end.map do |path|
  DownloadManager.download(path)
end.to_set
```

However, using `end\n{space}{space}.method` is just chaotic evil.

```ruby
# bad
paths = manifest.static_assets.images.map do |img|
  CDN.location(location).path(image)
end
  .map do |path|
    DownloadManager.download(path)
  end
  .to_set
```

#### Second Token:  `.`

The second token on which to break expressions is `.`, before `(`. Remember to indent using two spaces relative to the start of the expression it's part of.

```ruby
paths =
  manifest
    .static_assets(domain: application.domains.primary, format: Format::PNG)
    .images
    .map { |image| CDN.location(location).path(image) }
    .to_set
```

#### Third Token: `(`, `{`, `[`

If we wanted to make the previous example narrower, we would be splitting on the `(` token.
Note that I use `do/end` instead of `{}` for multiline. Since all method calls are parenthesized, they have the same semantics, and since I'm not squeamish about chaining on `end`, it poses no issue for chaining either.


```ruby
paths =
  manifest
    .static_assets(
      domain: application.domains.primary,
      format: Format::PNG,
     )
    .images
    .map do |image| 
      CDN.location(location).path(image)
    end
    .to_set
```

#### Fourth: `,`

The comma (`,`) holds a very low priority as a token to split expressions on. As a result of the prior rules, it's impossible to write the following code, since we would be splitting on `(` before we split on `,`.

```ruby
# bad
foo(:a,
    :b,
)
```

### Item Enumeration

When enumerating a list, either all items are on a single line, or they each have their own line. This is true for `Array` and `Hash` literals, argument lists, etc.

```ruby
# good, single line
[1, 2]
{ a: 1, b: 2 }
Set[1, 2]
foo(1, b: 2)

# good, one line each
[
  1,
  2,
]
{
  a: 1,
  b: 2,
}
Set[
  1,
  2,
]
foo(
  1,
  b: 2,
)

# bad
[1,
  2]
{ a: 1,
  b: 2 }
Set[1,
  2,
]
foo(1,
  b: 2
)
```

### Trailing Comma

Speaking of enumeration, whenever possible, the trailing comma is used. This includes `Array` and `Hash` literals, argument lists. Parameter lists are excluded as the syntax does not allow it.

```ruby
# good
[ 
  1,
]
{
  a: :b,
}
foo(
  1,
  b,
)

# bad
[ 
  1
]
{
  a: :b
}
foo(
  1,
  b
)
```

This minimizes the churn in the code.

### (Double) Quoting

Always favour double quotes (`""`) for strings ([somewhat inline with the RSG](https://rubystyle.guide/#consistent-string-literals-single-quote)). The reason is: they work predictably every time. One doesn't have to decide whether a they need interpolation, or special characters. I've seen too many times people using `'\n'` and not understanding why it doesn't work. 

Using exclusively double quotes simplifies the language.

### Percent-Literals

Percent-literals are disallowed. They introduce opportunity for mistakes (like using a comma to delimit items, only to have it become part of the item `%w( a, b ) #=> ["a,", "b"]`), and increase the burden on the reader. They introduce more complexity than convenience, and they can all be expressed using simpler constructs. The simpler constructs can also be composed, unlike percent-literals.

Removing percent literals simplifies the language.

I would allow the `%()` string literal only when the amount of escaping in a double-quoted string becomes overwhelming.

### Numbers

For numbers, use the underscore (`_`) semantically. If you're writing plain numbers, split on thousands: `1_200_000`. However, if you're encoding monies with two subunits (like canadian dollars), underscoring cents is a good idea: `200_00`. 

### Singleton Methods

The only acceptable way of defining singleton methods (commonly called class methods) is to use the `class << self` construct. 
I know it's more characters, but again, it works predictably every time, and it minimizes the 
[affordance for errors]({% post_url 2020-01-29-affordance-for-errors-pt1 %}). Using only `class << self` greatly simplifies the language.

```ruby
# good
class A
  class << self
    def foo
      :foo
    end
  end
end

A.foo
#=> :foo
 
# bad
class A
  def self.foo
    :foo
  end
end

# bad
class A
  def foo
    :foo
  end
  module_function(:foo)
end
```

### `extend(self)`

Using `extend(self)` is forbidden, for reasons explained in [affordance for errors – extending self]({% post_url 2020-01-29-affordance-for-errors-pt1 %}#extending-self).

## Sidenote On RuboCop

I really dislike Rubocop because it goes beyond its mandate.
I love it as a code formatting tool, but that's the end of it, and it shouldn't try to go beyond that, for the very simple reason that it doesn't have any knowledge of the code semantics.

- It doesn't know that a rescued `Exception` is fine if it gets re-raised
- It doesn't know that a given `Hash` is from user-input so the keys should be `String`
- It doesn't know that a value is a `BasicObject` and doesn't have the `nil?` method, or the `is_a?` method.    

ON CORRECTNESS:
The [Ruby style guide](https://rubystyle.guide/) unfortunately makes recommendations on the methods to use (and RuboCop enforces them) without knowing the types of the receiver can receive those methods. A good example of this is the [Predicate sectio](https://rubystyle.guide/#predicate-methods) recommending the usage of `x.nil?` over `x == nil`. Given `BasicObject` doesn't have `nil?` and that any class can inherit from `BasicObject`, WE CANT RECOMMEND THAT.




