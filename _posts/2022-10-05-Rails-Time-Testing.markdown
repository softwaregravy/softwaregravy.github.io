---
layout: post
title:  Testing with Time in Rails
date:   2022-10-05 12:00:00 -0400
---

I recently got a request from a developer on my team to install the [Timecop](https://github.com/travisjeffery/timecop) gem. 
This is a fantastic gem. 
We've used it a lot in the past. 
In our most recent project we've standardized on [TimeHelpers](https://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html), which is built into Rails and we haven't needed Timecop yet. 

This got me thinking, __how are they different?__ Lets go down the rabbit hole.

## Quick Intro

Timecop and TimeHelpers are used to stub the current time when writing tests for [Ruby on Rails](https://rubyonrails.org/). 
Timecop is a stand-alone gem, so you could use it with plain Ruby or Sinatra or whatever; TimeHelpers is built into Rails' ActiveSupport library. 
The ability to stub time is really useful and lets us write tests at fixed points in time, move fixed intervals through time, and test edge cases. 
(Testing 11:59 pm and 12:01 am in a local time zone at week and month boundaries are good candidates. In my experience test cases across 2:00 am on Saturday nights when DST happens will always fail.)

Here's a simple [spec](https://rspec.info/) using TimeHelpers where we test that our `Farm` class correctly reports the time in it's local time zone:

```ruby
it "returns local time in farm's timezone" do
  travel_to DateTime.parse("2022-05-08T01:00:00-00:00") do
    farm = FactoryBot.build(:farm, time_zone: "Pacific Time (US & Canada)")
    expect(farm.time_at_farm.to_s).to eq("2022-05-07 18:00:00 -0700")
  end
end
```

And here's the equivalent using Timecop:
```ruby
it "returns local time in farm's timezone" do
  Timecop.freeze DateTime.parse("2022-05-08T01:00:00-00:00") do
    farm = FactoryBot.build(:farm, time_zone: "Pacific Time (US & Canada)")
    expect(farm.time_at_farm.to_s).to eq("2022-05-07 18:00:00 -0700")
  end
end
```

In both cases we are teting that on May 8th, 2022 at 1:00am UTC, that our farm will correctly report time in it's local time zone. Pretty straight forward. 

### Comparing Time

One gotcha when writing tests involving time in Ruby is that you can be unqueal due to precision. The example below shows that Time penty of precision. For our purpses, this is a 63-bit signed integer and is the nanoseconds since November 12th, 1823 (anyone know why was chosen? I don't). Even 2 calls on the same line of code will result in different times being captured. 
```ruby
Time.current
 => Wed, 05 Oct 2022 15:13:32.966086000 UTC +00:00
 3.1.2 :008 > a = []
 => []
3.1.2 :009 > a << Time.current << Time.current
 => [Wed, 05 Oct 2022 15:19:41.538090000 UTC +00:00, Wed, 05 Oct 2022 15:19:41.538223000 UTC +00:00]
 ```
 One way to test times robustly is just to allow this to be noisy using the [`be_within`](https://relishapp.com/rspec/rspec-expectations/v/3-11/docs/built-in-matchers/be-within-matcher) matcher:
  ```ruby
 expect(planned_start_time).to be_within(1.second).of DateTime.parse("2022-05-24T17:00:00-00:00")
 ```
 Since we were specifically looking for the time zone component above, we compared strings which have fixed precision. This also works, but is a bit more brittle if you don't care about the time zone the time is recorded in.  

## Timecop

We dig into the source code for Timecop, and we find the _magic_ in the [time_extension.rb](https://github.com/travisjeffery/timecop/blob/master/lib/timecop/time_extensions.rb)
file. In this file, Timecop monkey-patches `Time`, `Date`, and `DateTime` classes to alias `now`, `new`, `today`, `parse`, and `strptime`. 
When you are using Timecop, and you call these methods, you will find any stubbed time that you've previously set through Timecop.

If you look into `Time`, `Date`, and `DateTime` classes, they all fall back to `now`, `new`, or `today` to get the time (`now` even falls back to `new`). 
An example of this is [`Time#current`](https://github.com/rails/rails/blob/8015c2c2cf5c8718449677570f372ceb01318a32/activesupport/lib/active_support/core_ext/time/calculations.rb#L39)
which is implemented as:
```ruby
def current
  ::Time.zone ? ::Time.zone.now : ::Time.now
end
```
If you were to dig into the `Time.zone.now`, you would also hit a starting point of `Time.now`. So stubbing these methods is sufficient to change all calls to get the time in a Ruby on Rails app. 

So back to looking at Timecop, here is what calling `Time#now` when using `Timecop` will get you:
```ruby
class Time
  class << self
  
    alias_method :now_without_mock_time, :now
    def now_with_mock_time
      mock_time || now_without_mock_time
    end
    alias_method :now, :now_with_mock_time
  end
  
  # ...
end
```
If you're unfamiliar with aliasing in ruby, this says that `#now` will be called `#now_without_mock_time`.
And then it renames `#new_with_mock_time` to be called `#now`. So when you call `Time#now` you will reach `#now_with_mock_time`.
(As a fun little exercise, you can write a test which calls `Time.now_without_mock_time`. It's available.)
`mock_time` is a simple method that just pulls the time from `Timecop`, if you've set anything so far during your testing; 
otherwise it falls back to the default implementation.
```ruby
3.1.2 :013 > Time.now_without_mock_time
 => 2022-10-05 11:47:41.356476 -0400
 ```

## TimeHelpers

The implementation in Rails is a bit more convoluted to follow, but essentially the same once you grok what's going on. 
The _magic_ for TimeHelpers is all in the [`#travel_to`](https://github.com/rails/rails/blob/48661542a2607d55f436438fe21001d262e61fec/activesupport/lib/active_support/testing/time_helpers.rb#L113)
method. It works very similar to Timecop stubbing `now` and `today` on `Time`, `Date`, and `DateTime`.

The key bit of code is the stubbing:
```ruby
simple_stubs.stub_object(Time, :now) { at(now.to_i) }
simple_stubs.stub_object(Date, :today) { jd(now.to_date.jd) }
simple_stubs.stub_object(DateTime, :now) { jd(now.to_date.jd, now.hour, now.min, now.sec, Rational(now.utc_offset, 86400)) }
```
This indirection to the stubbing is the convoluted part, but this code does what you think it does. `simple_stubs` is an instance of a [`SimpleStubs`](https://github.com/rails/rails/blob/48661542a2607d55f436438fe21001d262e61fec/activesupport/lib/active_support/testing/time_helpers.rb#L10)
which is implemented in the same file. This aliases the method with a `__simple_stub__` prefix and supports then also supports unstubbing.
Lets unpack the key part of the `stub_object` method:
```ruby
object.singleton_class.send :alias_method, new_name, method_name
object.define_singleton_method(method_name, &block)

# rewriting the above with values for instance variables
object.singleton_class.send :alias_method, :__simple_stub__now, :now
object.define_singleton_method(:now, { at(now.to_i) })

# and rewriting again in the same style the Timecop uses
class Time 
  class << self
    alias_method :__simple_stub__now, :now
    def now
       at(now.to_i)
    end
  end
end
```
The `now` is a local variable set to your desired date or time for the test.

## Differences

So what's the real difference here? Not much. At the end of the day, both implementations stub the same few methods the same core time/date libraries. 
There are a couple though:

1. Timecop does not support unstubbing methods. It just falls back to the default implementation if nothing is specified. 
`Timecop#return` clears the stubbed times. 
`TimeHelpers#travel_back` actually removes the stubbed methods and restores the original.
2. Timecop stubs more methods. `Time#new`, `Date#strptime`, `Date#parse`, and `DateTime#parse` are overridden by Timecop, but not TimeHelpers. 
This lets Timecop sub in stubbed times in these additional methods that might have missing pieces. 
  - As an example:
```ruby 
3.1.2 :020 > Date.parse("Jan-10")
 => Mon, 10 Jan 2022
3.1.2 :021 > Timecop.travel(Date.parse("2020-10-05")) {Date.parse("Jan-10")}
 => Fri, 10 Jan 2020 # Note the year is remembered from the #travel call
```
3. Timecop supports nested travels. 
The same will not work with TimeHelpers, but at least they catch it and raise a runtime error rather than returning the wrong times. 
  - Example:
```ruby
3.1.2 :025 > Timecop.travel(Date.parse("Jan-10")) { Timecop.travel(Date.parse("Feb-20")) { puts "I'm nested! #{Date.today}" }; puts "I'm the outer block! #{Date.today}" }
I'm nested! 2022-02-19
I'm the outer block! 2022-01-09
3.1.2 :026 > travel_to(Date.parse("Jan-10")) { travel_to(Date.parse("Feb-20")) { }}
...gems/activesupport-7.0.4/lib/active_support/testing/time_helpers.rb:154:in `travel_to':  (RuntimeError)
Calling `travel_to` with a block, when we have previously already made a call to `travel_to`, can lead to confusing time stubbing.
```
4. Timecop has some extra options:
  - Using `Timecop#travel` doesn't automatically freeze the clock -- you have to use `Timecop.freeze` for that. `TimeHelpers` always freezes time.
  - `Timecop` supports scaling time. Calling `Timecop.scale` can set the value of a second. 

## Overall

TimeHelpers is good. It's not _quite_ as good or as flexible as Timecop, but it has the advantage of being built into Rails. I am biased towards minimizing dependencies, even in my test envs.

Use TimeHelpers. You probably don't need Timecop. 
Unless you use [Capybara](https://github.com/teamcapybara/capybara) or tests that will run in separate processes, then you want Timecop. 
That will have to wait for the next post. 




