---
layout: post
title: Improve an activerecord wrapper
date: 2021-10-14 21:20 +0700
categories: rails activerecord log
---

## Context

We have a gem wraps around activerecord, which provide some function like.

```ruby
class Attribute
  belongs_to :attribute_type

  def as_json(**options)
    super(options.merge(except: %i[id created_at updated_at])).tap do |h|
      h['id'] = fancize(id)
    end
  end
end

class AttributeType
  def type_info
    'info_ae_different_betweens_types'
  end
end

class Query
  def initialize(tenant_id)
    @tenant_id = tenant_id
  end

  def business1
    query = root_scope
    query = query.scope1.scope2
    # Add some more magic and complexities to make problem more serious

    query.as_json
  end

  private

  def root_scope
    Attribute.where(tenant_id: @tenant_id)
  end
end
```

Then use `Query.new(1).business1`. 

Today, I want to have `type_info` optionally in `Query.new(1).business1` result. It first may quite straightforward, add to the `business1` method. 

```ruby
class Query
  ...

  def business1(includes: :type_info)
    query = root_scope
    query = query.scope1.scope2
    # Add some more magic and complexities to make problem more serious

    query.as_json(include: :type_info)
  end
  ...
end
```

Well, `type_info` is a method, not attribute. So we continue patch the `as_json`

```ruby
class Attribute
  ...

  def as_json(**options)
    # Exclude `type_info` in `options`
    super(options.merge(except: %i[id created_at updated_at])).tap do |h|
      h['id'] = fancize(id)
      h['type_info'] = attribute_type.type_info if options[:include] == :type_info
    end
  end
end
```

It works, but with 2 drawbacks. 

1. `business2` has a ton of params, `options` will be pushed to the tail. 
2. N+1, as we load `attribute_type` everytime. 

So, we need one more `includes`. And why not adding a little more magic, render `type_info` if it's ready, no matter what `as_json` options. 

```ruby
class Query
  def initialize(tenant_id, includes = [])
    @tenant_id = tenant_id
    @includes = includes
  end

  def include(includes)
    new(@tenant_id, @includes | Array(includes))
  end

  def business1
    ...
    query = query.includes(:attribute_type) if @includes.include?(:type_info)
    ...
    query.as_json
  end
  ...
end

class Attribute
  ...

  def as_json(**options)
    super(options.merge(except: %i[id created_at updated_at])).tap do |h|
      h['id'] = fancize(id)
      # TIL
      if association(:attribute_type).loaded?
        h['type_info'] = attribute_type.type_info
      end
    end
  end
end
```

Using `association(:attribute_type).loaded?` to check if it's loaded by `includes`. We could use `attribute_type.loaded?` but it like tricking my mind, if we called `attribute_type`, mean we had it already. Now, if we don't use `type_info`, just call `Query.new(1).business1` like normal, if we need `type_info`, just `includes` like rails syntax `Query.new(1).includes(:type_info).business1`. 

### Bonus

Along with `association`, I found another hidden hem `alias_attribute :new_attribute, :exist_attribute`, which could helpful if we want to render `attribute_type` in `Attribute#as_json` with a different name.

```ruby
class Attribute
  belongs_to :attribute_type
  alias_attribute :type_info, :attribute_type
end

attribute.as_json(include: :type_info)
attribute.as_json(include: {type_info: {...}})
```

But this approach doesn't provide much customization, just `exclude`, `only` or nested some fields. Another disadvantage, we couldn't use it with `includes` to preload ;(

## Note

### To enable rails sql log on console

To make sure `includes` works

```ruby
ActiveRecord::Base.logger = Logger.new(STDOUT)
```

Rails 6

```ruby
ActiveRecord::Base.verbose_query_logs = true
```

Sometime, need lower log level

```ruby
Rails.logger.level = 0
```
