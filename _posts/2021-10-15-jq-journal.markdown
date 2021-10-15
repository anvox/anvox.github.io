---
layout: post
title: jq Journal
date: 2021-10-15 21:35 +0700
categories: jq json journal
---


## Extract commit message from github diff

While writing a release tool, I need to collect commit messages from release branch. It's quite easy with [github API compare](https://docs.github.com/en/rest/reference/repos#compare-two-commits). The result is quite big, but thanks to it, I learnt `jq`'s power. It's not just query, but able to transform json. The jq script below jsut extract commits/(commit/message, author/login) with commits. 

```shell
curl -u $GH_USERNAME:$GH_TOKEN https://api.github.com/repos/$ORGANIZATION/$REPO/compare/$BRANCH_TO...$BRANCH_FROM | jq '.commits | .[] | {commit: .commit | {message: .message},author: .author | {login: .login}}'
# .commits # => nest into commits element
#         .[] # => Extract array items. For each item, transform to an object
#            {commit: .commit | {message: .message},author: .author | {login: .login}}
#             commit: .commit # => use `commit` element through pipeline
#                            {message: .message} # => just get message
#             author: .author # => Similar, object has `author` element, piped from value
#                            {login: .login} # => Same to login elment
```

There are 2 notes need to improve and learn more about `jq`. 

1. Duplicated `commit: .commit`. I think `jq` has this in it. 
2. Result is not a JSON, `.[]` breaks the array. Still not try, but I believe `jq` covers it already. To piped out a JSON document properly.
3. 
