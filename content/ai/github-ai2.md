+++
title = "Github dockerfile service using AI - Part 2"
date = "2025-11-25"
aliases = ["ai"]
tags = ["ai", "dev", "claude"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
I have been fooling around a lot with ai recently, and I thought I would write something about what I've been doing. There are a few things that I've been doing, and they're all fascinating. 

- Writing average looking code using minimum viable prompts.
- Getting this average looking code refactored to be a robust service.
- Generating API documentation - and having Claude fix this code.

This is part two of a small series that I have created to walk through the process I went through to get decent code.

## A crazy idea - update my dockerfiles
I had a crazy idea. I thought to myself, let's write something that will go through my git repos and automagially update my dockerfiles so that the dockerfile uses a fixed but more recent version of the base image.

In my first post, I looked at the codebase, which frankly was very average, but worked. I thought to myself "I wonder how much of a refactor it would take to get a real service working".

As it turns out.... not very much.

## Refactor my code ... please
I have been using Claude (not Claude code) for a little while now, and thought I might try to get it to refactor my **terrible** code to be well, better.

I use a simple prompt

```Shell
refactor the following code to make it more robust
```

Then I pasted in my awful code. 

![Code refactor 1](/images/code-refactor-1.jpg)

What came out was generally pretty good, however, Claude initially generated another script that I had to run that was relatively static.

![Code refactor 1](/images/refactor-1-improvements.jpg)

I realised I didn't want a static python script. I wanted a service.

![Code refactor 1](/images/refactor-1-how-to-use.jpg)

I used the prompt below to get Claude to refactor my code to make it more robust, and so that it had a REST based interface that I could send requests to.

```Shell
refactor the code to accept the repo as an input via a rest interface
```

![Code refactor 1](/images/refactor-2-outputs.jpg)

I realised that I didn't know how to use this, so I also asked for example usage using the prompt

```Shell
write a curl example of how to use the api
```

![Code refactor 1](/images/refactor-2-example-usage.jpg)
