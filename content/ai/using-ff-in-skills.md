+++
title = "Using feature flags in Claude skills"
date = "2026-05-26"
aliases = ["ai"]
tags = ["ai", "dev"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Intro
I have been playing around with skills for a while now. I've also been using feature flags in my code for some time as well. Feature flags are a really neat way of separating deployment and release in any codebase. I decided to explore how I could use feature flags in a Claude skill. This would essentially enable me to change and control the skill using nothing other than feature flags.

This particular exploration will use a skill that does nothing other than tell the time and date. I will use feature flags to control which parts of the time and date are shown.

This becomes important when you distribute your skill more widely, particularly in an enterprise context. Essentially, the moment you distribute it to more than one person, you need to manage the skill like you do any other enterprise code.

## What are feature flags?

## Why use feature flags in skills?
I wanted to explore a way to alter a skills behaviour without changing the skill itself. This came about when I was speaking to a customer and they said that they were writing a skill and distributing it within their organisation.

This made me think about distributing a skill and how testing new versions would work. It's easy enough to embed feature flags in applications, and a skill is an application, so why can't I use feature flags inside a skill?

This eases the deployment and testing of new features by separating release and deployment. New features can be coded into the skill and gated by feature flags.

## Code walkthrough
The entire skill comprises of two things. 

- A SKILL.md that defines the behaviour of the skill.
- A script to query my feature flags.

## The SKILL.md file
The skill is contained within a SKILL.md file. 
The core part of the skill, and what makes it generic is the following section.

```md
This table is the single source of truth for which flags this skill uses. To change behaviour, update this table only — no other section needs editing.

| Flag key | Gates |
|---|---|
| `flag_show_date` | Date display |
```

The other relevant section of the SKILL.md is the gating portion.

```md
#### Behaviour per flag



| Flag key | Gates | Enabled behaviour | Disabled behaviour |
|---|---|---|---|
| `flag_show_date` | Date display | Print `Date: <date>` line | Omit date line |
```

The best part about this, is that additional flags can be added to the table to scale the skill and alter even the skill behaviour. This can all be done in runtime without needing to redeploy or redistribute the skill to end users. 


## The flag script
The script in this case is a helper script that interfaces with my flag management system. In this case, I'm using LaunchDarkly. The script just uses a pull method to get the flags. While this isn't the most efficient means, it's a simple proof of concept. The objective here is to get all of the flags that are configured and defined. 

The script is here: [https://github.com/codecowboydotio/ld-skill/blob/main/.claude/skills/ld-skill/scripts/list_flags.py](https://github.com/codecowboydotio/ld-skill/blob/main/.claude/skills/ld-skill/scripts/list_flags.py)

## What does it look like?
The first image below shows the results with the flag enabled. 
Note that the date is shown alongside the time.

![With flag enabled](/images/skill-flag-enabled.jpg)


The image below shows the results with the flag disabled. 
Note that the date is not shown alongside the time.
![With flag disabled](/images/skill-flag-disabled.jpg)



## Extending beyond a simple use case
Extending beyond simple use cases into delivering more complex skills or more complex combinations of features based on environment variables becomes even more interesting.

Extending to complex use cases based on multi-step workflows. 

This solves the simple problem of needing to redistribute a skill constantly as it changes to multiple people / endpoints.
