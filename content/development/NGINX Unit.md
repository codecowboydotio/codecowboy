+++
title = "NGINX Unit"
date = "2021-08-11"
aliases = ["nginx_unit"]
tags = ["nginx", "unit"]
categories = ["software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I recently discovered NGINX Unit - now there's a disclaimer here as well - I work for the company that produces this software. 
I do think that it's a very very cool piece of open source software, so it generally suits my ethos:

- Open Source
- Super cool software
- Extensible
- Makes my life as a developer easier

It pretty much ticks all the boxes.

## What is it?
This one is a little harder to answer. 

**It's a lot of things**

I'll let the [unit webpage](https://unit.nginx.org) do the talking here:

**NGINX Unit is a polyglot app server, a reverse proxy, and a static file server, available for Unix-like systems.**

Dare I say it's a lot more than that though.


## In a nutshell
In a nutshell, if I had to describe it I would say it's a multi language application server that has a declarative configuration and description interface.

That's wordy but that's what it is to me.

## Why is this important?
In a modern application ecosystem I have many services written in multiple languages.
Some of these will be written in spring, some will be written in golang and some will be written in nodejs as examples. 

This is great because every team writes services the way that best fits them and makes sense for the delivery of that particular service. It lets teams operate at the speed they're comfortable with and means that services get delivered in the best way.

- What if I want services in different languages to coexist together? 
- What if I want to describe my services in a common way?
- What I don't want my pipelines to be brittle and I want them to be templated?

This is why I think Unit is important.

![unit.JPG](/images/unit.JPG)

## Multiple languages

NGINX Unit allows you to use multiple langugaes. In a modern application architecture, this is a very common thing to want to do.
Multiple services, written by different teams in different languages - this is a completely common scenario.

NGINX Unit allows you to deploy services written in multiple different languages on the same application server. 
This makes deployment easier and makes the whole dev experience a lot easier also.

## Declarative configuration
NGINX Unit has a REST based declarative configuration.
This is one of the other very nice things about Unit -  everything in the configuration is declarative. With a declarative configuration, everything is real time, and I don't need to worry about restarting applications.

A simple configuration looks something like this:

```JSON
{
        "listeners": {
                "*:8080": {
                        "pass": "applications/python"
                },

                "*:80": {
                        "pass": "routes"
                }
        },

        "routes": [
                {
                        "action": {
                                "share": "/www/pacman-unit/"
                        }
                }
        ],

        "applications": {
                "python": {
                        "type": "python",
                        "path": "/www/git-pull-api/",
                        "module": "wsgi",
                        "callable": "app"
                }
        }
}
```

## Components 

Each component of the configuration above allows you to step through connecting to, and serving my application.
A picutre is worth a thousand words here, so here is one that I like as it explains the concepts of unit nicely.

![unit-config.JPG](/images/unit-config.JPG)

Listeners expose the application publicly, and can have characteristics like ports, certificates, names and so on.
Listeners can pass to either routes or directly to applications. In the case above, my first listener passes directly to my application named "python".

My second listener passes to a route. While I only have one route, it is possible to have multiple routes. Each route can have multiple conditions or matches and actions. In the case above, the route simply passes to a directory that serves out a static website (and yes it's pacman).

Routes in unit are handled by a separate router process. This software based router handles request routing for unit.

A more complex route block looks like this:
```JSON
        "routes": [
                {
                        "match": {
                                "host": "static.svkcode.org"
                        },

                        "action": {
                                "share": "/www/pacman-canvas"
                        }
                },
                {
                        "match": {
                                "host": "api.svkcode.org"
                        },

                        "action": {
                                "pass": "upstreams/rr-lb"
                        }
                },
                {
                        "match": {
                                "host": "jsp.svkcode.org"
                        },

                        "action": {
                                "pass": "applications/java"
                        }
                }
```

Each route in this case passes to a different application based on the incoming host header. In this way, the unit router can be used to handle incoming request routing at a very granular level.


My application block is the last piece of the puzzle here.

```JSON
        "applications": {
                "python": {
                        "type": "python",
                        "path": "/www/git-pull-api/",
                        "module": "wsgi",
                        "callable": "app"
                }
```

My application block tells unit to run my application.

**That's right unit is an application server.**

The example above is a python application, but unit can also run nodejs, java, golang, perl (yes really - though I haven't tried it), php, ruby, python and so on.

The intention here is that unit handles instantiating the application and everything is configured via the declarative configuration language of unit in real time.


## Low barrier to adoption

Unit offers a very low barrier to entry.
1. The concepts are simple and sensible.
2. The declarative configuration is JSON based with a "get it, put it" mentality.
3. Everything is real time - make a change, it's reflected immediately.

## Conclusion
NGINX Unit is a good application server, and has a lot of uses. 
Look out for additional posts on how to use it, how to configure it, and where it fits in a knative world.
