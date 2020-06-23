---
title: Affordance for Errors, part 3
date: 2020-02-18 23:00:00 +0400
description: In this third and final post, we work through a practical example of API design, to see how to remove the affordance for errors.
categories:
- Ruby
aliases:
- "/ruby/2020/02/18/affordance-for-errors-pt3.html"
---

In the [first post]({{< ref "posts/affordance-for-errors-pt1" >}}) of this series, I showed how a few APIs afford errors to their users. In the [second post]({{< ref "posts/affordance-for-errors-pt2" >}}), I showed a few examples of how other APIs or languages have avoided or solved the same problems. In this third and final post, we will work through an example of designing an API.


When I design APIs, I try to think of all the ways it could possibly be misused, and remove as many ways as possible. Sometimes, this comes at the cost of _some_ ergonomy, but how much I'm willing to sacrifice depends on a few factors: the criticality of the errors, the impact on ergonomy, the (handwavy) likelihood that it will occur, to name a few.

## Design Example: Reservation Manager

I was recently writing an API to our reservation manager (the system holds reservation for carts, and disallows selling more items than are available). From the outside, the functionality is simple. You can reserve the items of a cart. Once reserved, the reservation can be claimed (ex: after the payment is successful) or unreserved (ex: if the payment failed).

### Who Are The Users?

I think the very first step of API design is empathy. Yearn for your users to succeed in their task. Understand where they're coming from. Make sure your APIs can be composed with the other tools they use.

The most important question you must ask yourself: "who will the users be?". Are they interns, senior developers, or principal engineers? Will they use your API every day, once per quarter, or once per decade? Is your API the cornerstone of their feature, or are they using it as an afterthought?

Whenever possible, aim for the lowest common denominator. If an intern unfamiliar with the language using this API for the first time can succeed, the probability that a principal engineer using it for the 10th time will too is very high. When it's less practical, understanding your users will help you make the right tradeoffs between the different factors.

In my case, this API is likely to be used by junior developers, and very rarely. They will definitely plan ahead as this will be integral to what they're building.

### Idiomatic API

I started by jotting down a first draft of the API, that handled all the cases necessary, in an idiomatic Ruby fashion. It looked something like this:

```ruby
manager = ReservationManager.new
is_reserved = manager.reserve(cart)
if is_reserved
  if process_payments(cart)
    manager.claim(cart)
  else
    manager.unreserve(cart)
  end
else
  # handle error
end
```

However, as we've seen previously, this API affords a lot of errors.

### Identifying Affordance for Errors

Using the learnings from the first post, we can quickly identify a few problems:

1. It's possible to ignore the return value of `reserve` and treat all reservations like they succeeded.
2. Additionally to #1, they can call `claim` and `unreserve` without having reserved in the first place.
3. Is it legal to use different instances of `ReservationManager` for `reserve` and `claim` or `unreserve`?
4. It's impossible to know when the reservation has exceeded its lifetime (ex, it has been garbage collected), and it's impossible to force the user to consume (`claim` or `unreserve`) the reservation.

For example, if the user had wrong assumptions about their system, they could write this implementation:

```ruby
manager.reserve(cart)
if process_payments(cart)
  manager.claim(cart)
end
```

Knowing that the users of this system will use this API approximately once in their entire career, I strongly favour reducing the surface for errors over idiomacy or ergonomy. Let's see how we can solve this problem.

### Removing the Affordances

To partially remove the affordance #1, we need to give an incentive to the user to use the return value. In fact, we will give them no choice. We can start by making `reserve` return a `Reservation` object, which is now the owner of `claim` and `unreserve` methods. In doing so, we also solved #3; our users _cannot_ use a different instance of `ReservationManager` to `claim` or `unreserve.`

```ruby
reservation = manager.reserve(cart)
if reservation.success?
  if process_payments(cart)
    reservation.claim
  else
    reservation.unreserve
  end
else
  # handle error
end
```

With this API, we only partially solved #1, and we haven't solved #2 at all, however. One can still fail to check if the reservation was successful. To solve this, we can wrap our `Reservation` in a `Result` object (aka, a kind of Either monad), forcing the user to at least check for success.

The usage of the `Result` object could be surprising to some Ruby developers, but in our codebase they are very common, and they wouldn't startle anyone.

```ruby
result = manager.reserve(cart)
result
  .on_success do |reservation|
    if process_payments(cart)
      reservation.claim
    else
      reservation.unreserve
    end
  end
  .on_error do |error|
    # handle error
  end
```

Using this version, the users will have no choice but to consider whether the reservation was successful before they proceed further with it.

Depending on the various factors, we could decide to stop here; we're already in a much better state than the original API. However for this API, I really wanted to make sure I solved #4 and force the user to `claim` or `unreserve`. We can do so by trading the return value of `reserve` for a mandatory block that will receive the reservation result.

```ruby
manager.reserve(cart) do |result|
  result
    .on_success do |reservation|
      if process_payments(cart)
        reservation.claim
      end
    end
    .on_error do |error|
      # handle error
    end
end
```

This allows us to `unreserve` the reservation if it hasn't been consumed by the end of the block. Depending on the specifics, we can also choose to `raise` if the reservation hasn't been consumed, to notify the users that their code has a problem.

With this API in place, we have dramatically reduced the surface for error. The objects in play will naturally guide the our users towards the correct way to use our API, and even if they don't, it won't compromise the correctness of our system.

Opinions may vary, but in mine, we haven't sacrificed ergonomics by the slightest to get here, either.

## Conclusion

In the first post of this series, I showed a few ways in which common APIs allow their users to make mistakes. My goal was to help you take notice of the problem, so that you can find similar problems (and more) in your own APIs. In the second post, we saw how others have solved the same problems, to help you see ways to remove the affordance you give. In this final post, I went through an example, and explained how I design APIs, and how I try to make it easy for my users to make no error. I sincerely hope you can take something away from this series, and that you start taking notice of, and start removing, the affordance for errors in your APIs.

[Comment or Like](https://github.com/gmalette/gmalette.github.io-src/pull/10)

