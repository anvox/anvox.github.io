---
layout: post
title: API Acceptance testing with Robot Framework
date: 2021-12-22 14:02 +0700
categories: testing api robot-framework
---

# Problem

We have a public API system, using JSON Spec. It was manually well tested. But now, after couple years, when we migrating from Heroku to AWS ECS, with some libraries upgraded, I'm not sure if it still works the same. I could call some testers, but for this tedious task, I prefer to look for some kinds of automation testing, especially, when comparing old and new identical API service.<br />

# Tools recommended from internet

First step, let find the "best" tools recommended by internet community. 

The first jumped out of my head is [Postman](https://www.postman.com), but I quickly drop it as the code-like syntax and the UI. I prefer an CLI tools, so I could make automation quickly. `Postman` has CLI tool, but it's not first class, all its contents revolve around the GUI. Besides, the code-like syntax is a disavantage. The test suites target to testers than developers, a human language like Cucumber is mandatory. 
I dropped [SoapUI](https://www.soapui.org/) and [Hoppscotch](https://hoppscotch.io/) with the same reason. It doens't mean they're not good, but too overkill to my small test.

At this time, I just expected some kind of spec like Cucumber, and a binary, run on those spec to verify. Without the binary, at least our testers could read and manually test using those spec. 

I finally find [karate](https://github.com/karatelabs/karate), sound promise, until I found it uses java. Putting this heavy stuff to project just to run some API test makes me feel overkill. Especially, when I need to extend, well, java is not my strong. 

I once thought to write a home-maded script to do it. Why not? It's quite simple. Just loop through some API, parse to get next compare results from old and new endpoint, verify important data. It's totally tedious and simple, even a monkey or a robot could do it. Wait. A robot likes [robot framework](https://robotframework.org)? I used it before for acceptance test on browser. Honestly, it's just an automation framework for anything we plug into. Eureka!

Let's see how does it fit my imagined requirements:

* Automated for CI - just python on console, we could run anywhere.
* BDD/Cucumber for human - It's a keywords system, a superset of Cucumber. 
* Programmable - deep dived, it's python under. I'm not a fan of python, but it's 2nd favorite language in our company. 

# Robot framework - hands on

It's super easy to run a robot script. Create `Dockerfile` and `docker-compose.yml` to prepare environment. 

```Dockerfile
FROM python:3.8

RUN mkdir -p /test
RUN pip install --upgrade pip && \
  pip install robotframework \
  robotframework-requests \
  robotframework-jsonlibrary \
  pyyaml

WORKDIR /test
```

```yaml
version: "3"
services:
  robot:
    build: .
    volumes:
      - .:/test
    command: sleep infinity
```

Then run

```shell
docker-compose up -d && docker-compose exec robot bash
```

Voilà! You could run any robot script now.

## First API test

Just verify if we could call API.

```robot
*** Settings ***
Documentation     Just ping to API endpoint
Library           RequestsLibrary

*** Test Cases ***
Get root
    ${resp}=  GET  ${ENDPOINT_URL}
    Status Should Be    OK    ${resp}
```

Variables file `global.yml`:

```yaml
endpoint_url: https://api.my-endpoint.com
```

```shell
# From container
robot --variablefile global.yml --outputdir ./report/ *.robot
```

That's all to check robot framework for API.

## Takeaways

Some takeaways when implement migration test to verify new/old endpoint using robot framework. Most on writing test script.

* [JSONLibrary](https://github.com/robotframework-thailand/robotframework-jsonlibrary) is needed to parse and manipulate JSON data. 
* `${None}` is python's `None`
* `FOR    ${i}    IN RANGE    1000` to loop through the range
* `FOR    ${item}   IN    @{json_resp['items']}` to loop through a list. Notice the `@{...}` is a list.
* Don't confuse with [RPA Framework](https://rpaframework.org/)
* Most stuffs you need are in [Robot framework documentation](https://robotframework.org/robotframework/latest/libraries/BuiltIn.html)
