+++
title = "NGINX Unit"
date = "2021-08-03"
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

.....more to come
