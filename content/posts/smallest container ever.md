+++
title = "Smallest Container Ever 17.7kB"
date = "2020-07-09"
aliases = ["smallest_container"]
tags = ["containers", "minimal"]
categories = ["containers", "testing"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I have a side hobby where I try to create minimal container images.
There are a number of reasons for this, but the primary one is because I'm a complete geek.

Other reasons include:

- It's a neatness thing
- There's an engineering imperative in me to be minimalist
- I don't like waste
- The process of creation helps me understand the technology a little better
- Smaller images are great for testing at scale

## Why
The why I do this is mainly because it's a hobby, but in this case, I started the ultra minimal container image build
because someone I work with wanted to build a container for testing Istio latency at scale. He needed a container image that was ultimately small, so that he could spin up in his words 'about 10,000' at once. The other goal was to minimise any application response time so that he could test Istio rather than the application behind Istio.

## Building a minimal image
Building a minimal image is easy, and I've spoken about it before at Container Camp, and you can see the [video](https://www.youtube.com/watch?v=SWwd4uTVeF0) if you like. 

What I have realised is that there are two parts to building a minimal image:

- Using SCRATCH and only including your application in your image.
- Statically linking your binary - but doing so in a super strict and optimised way.

When I spoke at Container Camp a few years ago, I only used default options during my compile. I have since realised that I can go even smaller by optimising my compiler options!

## The Code
This is the easy part. 

I needed an "application" that didn't introduce any latency at all so that my colleague could perform Istio testing. I decided to go as minimal as possible and create an "application" that actually doesn't do anything.

In order for the application to work within a containerised environment, it needs to continue to run, and never exit. If the application ever exits, the container will stop running.

C is my chosen language of choice today, and I created the following application.

```
#include <unistd.h>

int main(void) { 
    return pause();
}
```

This tiny little code block does a single thing - it pauses the thread until a signal is received. 

In container terms, this means the container will run, and will wait for a valid signal, and then honour that signal. This is the perfect behaviour for container testing. The "application" in the container doesn't do anything, so doesn't add any latency to the transaction, and is perfect for testing.


