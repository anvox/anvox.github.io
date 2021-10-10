---
layout: post
title: Terraform variable types
date: 2021-10-10 17:27 +0700
categories: terraform
---

Today, I'm reminded about terraform object from an old course. I read once and forgot it. Just used map all the time. Complex type may be too complex for resorce declaration languague, but if we're using map, why don't we leverage script to use object a more strictly data format. Especially, when using it dynamically, under `golem-tf` wrapper, to provide resource planning for multiple stages with one unify code. 

## Primitive types

Number, bool and string are most used. 

## Collection types

* `map(type)` is widely used, in short hand `{}`
* `list(type)` 
* `set(type)` is similar to list, but unique.

## Structural types

The lost gems

### object

```
object({key=type,key=type})
```

While I'm separating stages differences using conditional expression `condition ? true_val : false_val`, which just enough for `"var.environment" == "production" ? "large-resource" : "small-resource"`. Using object may give us a more readability and flexible approach:

```
variable "resource_levels" {
  type = map(object({ web : string, size : number }))
  default = {
    production = {
      web  = "t2.medium",
      size = 2
    },
    staging = {
      web  = "t2.nano",
      size = 1
    }
  }
}
```

Then just simple in resource definition. 

```
resource aws_ec2_instance {
  count = var.resource_levels[var.environment].size

  type = var.resource_levels[var.environment].web
}
```

### tuple

I don't have any use case of `tuple` for now, so it's best to read from [documentation](https://www.terraform.io/docs/language/expressions/type-constraints.html). 
