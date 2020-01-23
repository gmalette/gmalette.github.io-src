---
layout: post
title: About My Ruby Coding Style
date: 2020-01-07 20:00:00 +0400
description: ""
categories:
- Ruby
---

I use a Ruby coding style diverging from the community [Ruby style guide](https://rubystyle.guide/) (later called the Styleguide). This style has been chosen deliberately, and is often met with criticism from newcomers on my team, especially experienced Ruby programmers (rightly) claiming it isn't idiomatic. This post will lay out my style guide, emphasizing on the divergences with the community style guide, and also reinforcing some recommendations of the community style guide.

_Note: If you use an auto-formatting tool that saves on every save._

Note: I really dislike Rubocop because it goes beyond its mandate. It doesn't have the knowledge required, ex: `rescue Exception` can be re-raised
- Symbobls as keys
ON CORRECTNESS:
The [Ruby style guide](https://rubystyle.guide/) unfortunately makes recommendations on the methods to use (and RuboCop enforces them) without knowing the types of the receiver can receive those methods. A good example of this is the [Predicate sectio](https://rubystyle.guide/#predicate-methods) recommending the usage of `x.nil?` over `x == nil`. Given `BasicObject` doesn't have `nil?` and that any class can inherit from `BasicObject`, WE CANT RECOMMEND THAT.


Goals:
- Simplicity. Ruby is a complicated language, and we can easily make it simpler by refusing to use parts of it, at no cost. 
- Ease. It should be obvious how to write the code. It shouldn't require years of experience.
- Correctness. 
- Preserve expressions

## My Style Guide

### Line Splitting

Since line splitting is where I diverge the most from the idiomatic Ruby style, and since this is the cornerstone of my style, it makes sense to start here. 

I tend towards a functional style of programming, and I don't shy away from long expressions, provided they are clear. I don't introduce intermediate bindings–nor extract single-use methods–for the sake of it. 

This section will be better explained by working through an example, which we'll split as we go.

```
results = File.readlines(root.join(index_file_name))
  .map { |path_to_download| uri = URI(CDN.location(east).with_path(path_to_download)) }
  .map { |uri| result = DownloadManager.download(uri); puts "Downloading file #{uri}: #}

```

I have determined a set of rules that allow me to predictably split long lines, wit a result that's predictable.

I don't particularly care about the indenting level. Where some people will have fast rules like "no more than 4 levels of indenting", I don't. I focus on the width of the text, and I try to keep my code either single-line, or column width, basically less than 40 characters. 

While most people will be eager to split their lines on the `(` or `.` characters, the first character I split on is `=` (assignment). 

Example:

```ruby
paths = 
  manifest.static_assets.images.map { |image| CDN.location(location).path(image) }.to_set
```

Generally, binding names aren't very long, and this doesn't buy a lot of real estate, but I choose to break on `=` first because of the other properties it offers: it allows preserving `if/else`, `case/when` , `do/end`, and similar constructs's indentation, and allows chaining expressions more predictably than if we split on `.` or `(`.

Example:

```ruby
paths = 
  manifest.static_assets.images.map do |image|
    CDN.location(location).path(image)
  end

# or

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

The 2nd token I'll split on is `.`, before `(`. This gives the code a nice functional style, which also works well for OOP. It works whether the initial receiver is a long constant name or a short identifier. 

```ruby
paths = 
  manifest
    .static_assets
    .images
    .map { |image| CDN.location(location).path(image) }
    .to_set
```

After that, we can split on braces and `do/end`. Note that I would not do it for this code, but for the sake of example:

```ruby
paths = 
  manifest
    .static_assets
    .images
    .map do |image| 
      CDN.location(location).path(image)
    end
    .to_set
```

Notice that the `do/end` block and the `to_set` method had a predictable indentation even before this change. Other splitting methods could've left us with a weird indentation on `.to_set`. 

### Parentheses

*Always use parentheses when calling a method with arguments.*

This is in line with the Ruby Styleguide to [use parentheses](https://rubystyle.guide/#method-invocation-parens), except that they also recommend [not using parentheses for internal DSL](https://rubystyle.guide/#method-invocation-parens-internal-dsl). Disregarding that second part makes the style easier to adopt, as one doesn't have to know what counts as a DSL, and avoids arbitrary opinions on whether something is or isn't a DSL. Always using parentheses also produces higher correctness. Read [Revoking the (Parentheses) Privilege]({% post_url 2018-12-03-revoking-the-parentheses-privilege %}) to know more.

### (Double) Quoting

Always favour double quotes (`""`) for strings. The reason is: they work predictably every time. One doesn't have to decide whether a they need interpolation, or special characters. I've seen too many times people using `'\n'` and not understanding why it doesn't work. 


THINGS I WANT TO TALK ABOUT:
- Double Quotes (https://rubystyle.guide/#consistent-string-literals-single-quote)
- Split on =, (, .
- Ternary
- Implicit `nil`
- double-bang (!!)
- extend(self))
- Literals like %W, but `%()` for strings with lots of escaping is fine
- def self. singleton methods
- module_function
- trailing commas
- hash literals (`{ a: :b }`) are acceptable


