---
layout: post
title: Import to terraform all first AWS Access key to terraform
date: 2022-01-19 11:04 +0700
categories: til bash
---

# Context

Import IAM users AWS Access Key to terraform state to manage automatically. I'm have couple AWS IAM users manually created after 6 years using AWS. Now, on road to apply Infrastructure as code, managed these user accounts under code is a must. Import existing ones into terraform is most tedious task, which in turn, be the best candidate for automation.

# Script

```shell
aws iam list-users --page-size 100 --path-prefix '/user/' | \
  jq '.Users[].UserName' | \
  # head -n 1 | \
  xargs -I {} bash -c \
  'echo {};aws iam list-access-keys --user-name "{}" | jq -r .AccessKeyMetadata[0].AccessKeyId' |
  xargs -L2 golem-tf -e production -t developers import 'aws_iam_access_key.users["$0"]' $1
```

The script quite simple:

* List all users by pattern - user accounts used by developers in this case
* Parse to get the username
* `head -n 1` filter out to test on a small subset examples
* For each user, print name and their first Access Key to next pipeline
* Run terraform import command, `golem-tf -e production -t developers import ...` is a wrapper of `terraform import ...` with my internal convention.

I don't use bash frequently, so everytime, I have to search again. Hope this note could help me recalling them better.

# TIL

## `comm` command

In the beginning, I run import some resources manually. Then in the full run round, I must `grep -v ...` some names. But after, the list getting complex, I have to filter by a whitelist, where `comm` shines. 
`comm` does one simple thing, compares 2 files and prints items into 3 columns: file 1 only, file 2 only, both 2 files. Turn off column by flags `-123`. Using `comm`, we could easily pipeline intesect of list of username AWS returned and list we manage.

Ref [https://www.geeksforgeeks.org/comm-command-in-linux-with-examples/](https://www.geeksforgeeks.org/comm-command-in-linux-with-examples/)

## `xargs` command

Two common problems I mostly encounted:

* Inject arg into middle of a command: `-I {}` `-I ...` marks a string as placeholder for arguments, imply `-n 1`
* Inject multipe args: `-L2` Take N lines for a pipeline, `2` in this case. Then arguments will be named `$1`, `$2`, ... 

## bash `$'...'` string

In bash, to escape a single in a single quoted string. `\'` doesn't work. Instead, use `$'...'`, `\'` works inside those symbol

```shell
# This one waiting for string completed
echo 'Problems aren\'t stop signs, they are guidelines'

'
```

```shell
# Completed and printout
echo $'Problems aren\'t stop signs, they are guidelines'
```
