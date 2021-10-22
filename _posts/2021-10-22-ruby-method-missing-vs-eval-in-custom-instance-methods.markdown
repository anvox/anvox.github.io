---
layout: post
title: Ruby method_missing vs eval in custom instance methods
date: 2021-10-22 16:27 +0700
categories: ruby method_missing eval meta-programming 
---

## Context - a user profile

While implement new feature, we extract a user profile into a separated gem. 
This profile object has interface like activerecord, we could access profile attribute by name as method or using hash syntax. We also have 2 types of attribute: system and custom. Every users have the same group of system attributes, but depends on their subscription or tenant configuration, custom attributes may vary. I wished we could call `o.foo` or `o.fee` for consistency, still have no idea at that time. 

To simplify, I call `"attribute:(#{@id})[#{key}]"` attribute value. In real life, we have to hit db with a bit complex logic for the value. 

```ruby
class DtoOld
  [:system_foo, :system_fee].each do |method_name|
    define_method(method_name) do |_args|
      self[method_name]
    end
  end

  def initialize(id)
    @id = id
  end

  def [](key)
    "attribute:(#{@id})[#{key}]"
  end
end

o = DtoOld.new(1)
o.system_foo
o.system_fee
o[:foo]
o[:fee]
```

It's good enough, until we have more system attribute. Because this code placed in separated gem, we must bump it everytime new attribute added. I moved it to a config to extract the list to config.

```ruby
class DtoOld
  def self.system_attributes=(value)
    @system_attributes = value
  end

  def self.system_attributes
    @system_attributes || [:system_foo, :system_fee]
  end
  # ...
end

# In an initializer
DtoOld.system_attributes = [:system_foo, :system_fee, :system_more]
```

And it doesn't work. `o.system_more` is not defined. `DtoOld`'s `defined_method` runs at load time, everything is done at initialize. We could set `require: false` in `Gemfile` and `require user_profile/dto_old` after initialize, but it's not the beautiful of this magic world. Just unacceptable.

## `method_missing` comes in

The first light, defer all of them til we call it., by `method_missing`. Except the rumor of `method_missing` performance and we must checking key in list everytime, everything looks good. Accidentially, I archived the common interface, now we could call attribute name as method `o.foo`.

```ruby
class Dto1
  def method_missing(method_name, *args, &block)
    if supported_attributes.include?(method_name)
      self[method_name]
    else
      super
    end
  end

  def initialize(id)
    @id = id
  end

  def [](key)
    "attribute:(#{@id})[#{key}]"
  end

  private

  def supported_attributes
    # [:foo, :fee] is fetch by @id
    [:system_foo, :system_fee] + [:foo, :fee]
  end
end
```

## Try some evil

We call `eval` evil :D and keep avoid it, but how about this case, maybe eval faster than `method_missing`, we could reduce ton of key checking and `method_missing`. 

```ruby
class Dto2
  class << self
    def load_system_attribute_types
      return if @system_attribute_type_loaded

      @system_attribute_type_loaded = true
      self.class_eval([:system_foo, :system_fee].map do |attr_type|
        "def #{attr_type};self[\"#{attr_type}\"];end"
      end.join(';'))
    end
  end

  def initialize(id)
    @id = id

    self.class.load_system_attribute_types
    # [:foo, :fee] is fetch by @id
    self.instance_eval([:foo, :fee].map do |attr_type|
      "def #{attr_type};self[\"#{attr_type}\"];end"
    end.join(';'))
  end

  def [](key)
    "attribute:(#{@id})[#{key}]"
  end
end
```

So good so far, let make a benchmark to proof. 

## Benchmark

```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.report("method missing") do
    100.times do |i|
      o = Dto1.new(i)
      o.foo
      o.fee
      o.system_foo
      o.system_fee
    end
  end
  x.report("eval") do
    100.times do |i|
      o = Dto2.new(i)
      o.foo
      o.fee
      o.system_foo
      o.system_fee
    end
  end
  x.compare!
end
```

😱😱😱😱😱😱 `eval` 8x slower than `method_missing`

```
# Benchmark 1
Warming up --------------------------------------
      method missing   272.000  i/100ms
                eval    34.000  i/100ms
Calculating -------------------------------------
      method missing      2.841k (± 2.3%) i/s -     14.416k in   5.077444s
                eval    352.216  (± 3.4%) i/s -      1.768k in   5.025971s

Comparison:
      method missing:     2840.7 i/s
                eval:      352.2 i/s - 8.07x  (± 0.00) slower
```

Ouch, it's clearly there are some problem with `eval` approach. But what if we attack to `missing_method` the weakness by call methods more than init object. 


```ruby
# Code to benchmark
o = Dto.new(1)
100.times do |i|
  o.foo
  o.fee
  o.system_foo
  o.system_fee
end
```

Not bad, `eval` is faster 1.5x in this case.

```
# Benchmark 2
Warming up --------------------------------------
      method missing   299.000  i/100ms
                eval   381.000  i/100ms
Calculating -------------------------------------
      method missing      2.852k (±11.9%) i/s -     14.053k in   5.013784s
                eval      4.542k (± 2.4%) i/s -     22.860k in   5.035900s

Comparison:
                eval:     4542.1 i/s
      method missing:     2851.9 i/s - 1.59x  (± 0.00) slower
```

In real life, mostly, all attributes are called once or twice. So the benchmark#1 is closest to our case. I also tried to init 100 objects, call each of them 100 times, which then turns out `eval` faster than `method_missing` 1.8x, but not our case. 

## Hybrid solution

At this time, I nearly stop have fund to submit PR with `missing_method` approach. While looking back the problem, I just thought as the system attributes belong to all profile, just define them in class, then if custom attributes are vary, let the `method_missing` handles - a hybrid approach. 

```ruby
class Dto3
  class << self
    def load_system_attribute_types
      return if @system_attribute_type_loaded

      @system_attribute_type_loaded = true
      self.class_eval([:system_foo, :system_fee].map do |attr_type|
        "def #{attr_type};self[\"#{attr_type}\"];end"
      end.join(';'))
    end
  end

  def initialize(id)
    self.class.load_system_attribute_types
    @id = id
  end

  def method_missing(method_name, *args, &block)
    # [:foo, :fee] is fetch by @id
    if [:foo, :fee].include?(method_name)
      self[method_name]
    else
      super
    end
  end

  def [](key)
    "attribute:(#{@id})[#{key}]"
  end
end
```

```ruby
# Benchmark 3.1
Benchmark.ips do |x|
  x.report("method missing") do
    100.times do |i|
      o = Dto1.new(i)
      o.foo
      o.fee
      o.system_foo
      o.system_fee
    end
  end
  x.report("eval") do
    100.times do |i|
      o = Dto2.new(i)
      o.foo
      o.fee
      o.system_foo
      o.system_fee
    end
  end
  x.report("hybrid") do
    100.times do |i|
      o = Dto3.new(i)
      o.foo
      o.fee
      o.system_foo
      o.system_fee
    end
  end
  x.compare!
end
```

Result as expected, hybrid approach faster than both others in this benchmark.

```
# Benchmark 3.1 100 objects, each object calls 4 methods
Warming up --------------------------------------
      method missing   297.000  i/100ms
                eval    36.000  i/100ms
              hybrid   400.000  i/100ms
Calculating -------------------------------------
      method missing      3.020k (± 2.3%) i/s -     15.147k in   5.018850s
                eval    373.974  (± 2.4%) i/s -      1.872k in   5.008605s
              hybrid      4.070k (± 2.5%) i/s -     20.400k in   5.015350s

Comparison:
              hybrid:     4070.1 i/s
      method missing:     3019.7 i/s - 1.35x  (± 0.00) slower
                eval:      374.0 i/s - 10.88x  (± 0.00) slower
```

And the rest 2 benchmarks, hybrid method has performance similar the fast approach - `eval`. 

```
# Benchmark 3.2 1 object, each object calls 100*4 methods
Warming up --------------------------------------
      method missing   299.000  i/100ms
                eval   373.000  i/100ms
              hybrid   441.000  i/100ms
Calculating -------------------------------------
      method missing      3.193k (± 2.1%) i/s -     16.146k in   5.058729s
                eval      4.680k (± 5.3%) i/s -     23.499k in   5.035794s
              hybrid      4.487k (± 3.2%) i/s -     22.491k in   5.018308s

Comparison:
                eval:     4680.3 i/s
              hybrid:     4486.9 i/s - same-ish: difference falls within error
      method missing:     3193.1 i/s - 1.47x  (± 0.00) slower

# Benchmark 3.3 100 objects, each object calls 100*4 methods
Warming up --------------------------------------
      method missing     3.000  i/100ms
                eval     4.000  i/100ms
              hybrid     4.000  i/100ms
Calculating -------------------------------------
      method missing     28.706  (±10.5%) i/s -    144.000  in   5.070629s
                eval     42.400  (±14.2%) i/s -    208.000  in   5.012264s
              hybrid     44.485  (± 2.2%) i/s -    224.000  in   5.036704s

Comparison:
              hybrid:       44.5 i/s
                eval:       42.4 i/s - same-ish: difference falls within error
      method missing:       28.7 i/s - 1.55x  (± 0.00) slower
```

## Summary

I initially thought `eval` is faster than `method_missing`, but benchmark proved the opposite, in my specific case. But then, a hybrid approach, `eval` for the common things, `method_missing` for the various things, win the game. 
