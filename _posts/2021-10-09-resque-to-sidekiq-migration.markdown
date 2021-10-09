---
layout: post
title:  "Resque to Sidekiq migration"
date:   2021-10-09 13:00:00 +0700
categories: resque sidekiq
---

## tl;dr

Some footnotes after migration from `resque 1.x` to `sidekiq` latest. 

## Why

Our current system rely on `resque` heavily, but it's outdated. We want to upgrade to latest versions, but ~300 jobs need to test and verify at once, this is a huge I don't want to face. Migrate to `sidekiq` helps us have time to test and verify each job. Besides, `sidekiq` is better in performance, resources consuming, with an active commnuity and we have Pro license from other project. 

## Hands on

Sidekiq uses a different approach, compare to Resque, there are some notices when hands on: 

1. Enqueue inside transaction. 

2. Complex payload - Sidekiq uses different Marshal/unmarshal mechanism, complex object is forbidden. 

3. Batch enqueue. This is just an improvement but significant, batch enqueue Sidekiq improve a lot. 

4. Migrate `enqueue_at`/`enqueue_in`/`deliver_at`/`deliver_in`. While changing to `perform_at`/`perform_in`, we must migrate the scheduled/delayed job. 

5. Keep retry options. If old Resque jobs don’t mention about retry, they’re not retriable. 

6. Use connection pool for external connection instead of class instance. We used globel `$redis` instance before, because Resque forks process, which in turn, creates different `$redis` for different job. Sidekiq use threads, which share same `$redis` instance, it’ll be locked or return wrong result. 

7. Named params doesn't work with Sidekiq. #1 covered this already, but while named params look native to ruby, it’s not with Sidekiq. 

8. Gem versions aren't compatible with Sidekiq but not lock in gemspec: `newrelic < 6.0`. On upgrade document, newrelic agent specify they support Sidekiq 6.0+ from their version 6.0. 

9. [WIP] Frequent “Socket error” on Sentry without any clear stack trace or context. Not sure why, but it doesn't happen on Sidekiq run on ECS. 

10. [WIP] Upgrade `rufus-scheduler` from `3.4` to `3.8`. `rufus-scheduler 3.5` changed cron string parser, which does not accept cron string has both `day of week` and `day of month`, we must lock it before. Upgrade to `3.8` it now accepts this kind cron string but changed in behavior, instead match both `DoW` and `DoM`, it matches both, which then enqueue job unexpected. Must change cron string before upgrade. 

## Look back

More than 100 PRs labeled (I want to count Loc/ changed but seems we needn’t)
8 months GEM of 6 developers. First PR dated from Jan 2021, (current) last PR dated on Sep 2021.
In which 1 full Technical debt (2-weeks) sprint of 3 developers

3 big outages: 

1. Name params doesn't work after deserialize. Jobs failed, a lot. 
2. Upgrade Sidekiq with whole eco in a bundle. Unable to start worker, unable to enqueue jobs, whole system down. 
3. Concurrency too much, throttled by 3rd party service. Job failed, a lot.

1 big incident made duplicated email to all active clients. 

Some small significant bugs:

1. Unable to start new app because of broken resque task
2. typo :( missing include worker module 
3. Broken logic because of retry. Some jobs are not idempotent. 
