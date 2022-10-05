---
layout: post
title:  Testing with Time in Rails
date:   2022-10-05 12:00:00 -0400
---

I recently got a question from my developer asking to install the [Timecop](https://github.com/travisjeffery/timecop) gem. 
This is a fantastic gem. 
We've used it a lot in the past. 
But in our most recent project we've standardized on [TimeHelpers](https://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html), which is built into Rails.

This got me thinking, how are they different? 

## Timecop

So I dug into Timecop. We can find the magic in the [time_extension.rb](https://github.com/travisjeffery/timecop/blob/master/lib/timecop/time_extensions.rb)
file where Timecop meta-programs (monkey-patches?) `Time`, `Date`, and `DateTime` classes to alias `now`, `new`, `today`, `parse`, and `strptime`. 
When you are using Timecop, and you call these methods, you will find any fake time that Timecop has set.

As an aside, if you dig into most other methods in those classes, they all fall back to one of those methods to get the time. 
an example of this is [`Time#current`](https://github.com/rails/rails/blob/8015c2c2cf5c8718449677570f372ceb01318a32/activesupport/lib/active_support/core_ext/time/calculations.rb#L39)
which is implemented as:

```ruby
def current
  ::Time.zone ? ::Time.zone.now : ::Time.now
end
```
If you were to dig into the `Time.zone.now`, you would also hit a starting point of `Time.now`. 

As an example, here is what calling `Time#now` will get you:

```ruby
alias_method :now_without_mock_time, :now
def now_with_mock_time
  mock_time || now_without_mock_time
end
alias_method :now, :now_with_mock_time
```
If you're unfamiliar with aliasing in ruby, this is saying that `Time#now` is now called `now_without_mock_time`.
And then we reassign our new method, `new_with_mock_time` to be `now`. So when you call `Time#now` you will reach `now_with_mock_time`.
(As a fun little exercise, you can write a test which calls `Time.now_without_mock_time`. It's available.)
`mock_time` there is a simple method that just pulls the time from `Timecop` if you've set anything so far during your testing. 


## TimeHelpers

And then I dug into TimeHelpers. The implementation in Rails is a bit more convoluted to follow, but essentially the same. 
The magic for TimeHelpers is all in the [`#travel_to`](https://github.com/rails/rails/blob/48661542a2607d55f436438fe21001d262e61fec/activesupport/lib/active_support/testing/time_helpers.rb#L113)
method. It works very similar where `now` and `today` are stubbed on `Time`, `Date`, and `DateTime`.

The key bit of code is the stubbing:
```ruby
simple_stubs.stub_object(Time, :now) { at(now.to_i) }
simple_stubs.stub_object(Date, :today) { jd(now.to_date.jd) }
simple_stubs.stub_object(DateTime, :now) { jd(now.to_date.jd, now.hour, now.min, now.sec, Rational(now.utc_offset, 86400)) }
```
This is the convoluted part, but this does what you think it does. `simple_stubs` is an instance of a [`SimpleStubs`](https://github.com/rails/rails/blob/48661542a2607d55f436438fe21001d262e61fec/activesupport/lib/active_support/testing/time_helpers.rb#L10)
which is implemented in the same file. This just auto-hides the aliased method with a `__simple_stub__` prefix and supports unstubbing.
It ends up calling 
```ruby
object.singleton_class.send :alias_method, new_name, method_name
object.define_singleton_method(method_name, &block)
```
whcih means that `simple_stubs.stub_object(Time, :now) { at(now.to_i) }` is stubbing `Time#now` to be implemented as `at(now.to_i)`.
(The `now` was just a local variable set to your desired date or time for the test.)

## Differences

So what's the real difference here? Not much. At the end of the day, both implementations stub the same few methods the same core time/date libraries. 
There are a couple though:

1. `Timecop` does not support unstubbing methods. It just falls back to the default implementation if nothing is specified. 
`Timecop#return` clears the stubbed times. 
`TimeHelpers#travel_back` actually removes the stubbed methods and restores the original.
1. `Timecop` stubs more methods. `Time#new`, `Date#strptime`, `Date#parse`, and `DateTime#parse` are overridden by `Timecop`, but not `TimeHelpers`. 
This lets `Timecop` sub in stubbed times in these additional methods that might have missing pieces. 
* As an example, you can stub `Timecop.travel(Date.parse("2020-10-05")`, and then `Date.parse("Jan")` would resolve to "Sun, 05 Jan 2020" -- it remembered the year and day.
1. `Timecop` has some extra options:
* Using `Timecop.travel` doesn't automatically freeze the clock -- you have to use `Timecop.freeze` for that. `TimeHelpers` always freezes time.
* `Timecop` supports scaling time. Calling `Timecop.scale` can set the value of a second. 





