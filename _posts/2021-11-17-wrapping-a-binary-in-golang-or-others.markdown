---
layout: post
title: Wrapping a binary in golang or others
date: 2021-11-17 01:58 +0700
categories: process-wrapping
---

## Context

The context is quite simple:

* The first case, I wrote a docker entrypoint, to filter the support commands and follow some internal conventions. It's quite simple with a clear requirement: entrypoint must pass term/kill signal to command process, to support graceful shutdown. 
* The second, I wrapped terraform command. It run from dev local, so I relax most bar, even the term/kill signal handling.

## The problem

Til I apply a large stack, and it stuck. Like normal routine, I trigger a `Command+C`, then had my prompt back, couple try and `Command+C`. Til I enable `TF_LOG=TRACE`, it shocked me that `Command+C` return the prompt but the log still print out. I found couple `terraform` process still run. 

I just reminded that after `Command+C`, terraform will inform about losing data on interupt. If I notice its absent early... I could avoid this mess. 

## Action

1. Manually term each terraform process. This case some orphan resources I must clean manually and some messes in state file.
2. Bring mechanism of forwarding signal to terraform wrapper [https://github.com/anvox/jinn-tf](https://github.com/anvox/jinn-tf) 
3. Make this note: always forward signals for wrappers.

