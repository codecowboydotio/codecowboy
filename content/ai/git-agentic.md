+++
title = "Github using claude routines"
date = "2026-09-19"
aliases = ["ai"]
tags = ["ai", "dev"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Using claude routines with github

## Intro
I've been using claude for some time now. I haven't touched on routines in claude yet, and thought that this would be a good opportunity to play around with them and explore them. 

The official claude docs say this:

{{< notice info >}}
Put Claude Code on autopilot. Define routines that run on a schedule, trigger on API calls, or react to GitHub events from cloud infrastructure.
{{</notice>}}

I thought to myself "could I get rid of actions entirely?" and set about to find out!

## What are routines?
Routines are saved configurations (prompts) inside claude code. They run on Anthropics infrastructure, and can be connected to one or more repositories. Routines run on Anthropics infrastructure and can respond to API requests or to Github events (like pull requests). The third option is more like cron, where routines are scheduled for particular times and just run, again this is on Anthropic's infrastructure and they will run when your personal instance of claude is not running.

This means that routines can run when I am no longer around. This is interesting.

## Getting Started
In order to get started you simply log into the web interface of claude code and hit the routines page.

[https://claude.ai/code/routines](https://claude.ai/code/routines)

![](/images/git-agentic-install-and-authorise.png)
![](/images/git-agentic-claude-authorised.png)
![](/images/git-agentic-github-app-not-installed.png)

![](/images/git-agentic-create-routine.png)
![](/images/git-agentic-after-create.png)
![](/images/git-agentic-installed-connectors.png)


## Summary

