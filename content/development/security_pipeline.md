+++
title = "Security in my pipeline - why and how"
date = "2022-12-16"
aliases = ["git"]
tags = ["git"]
categories = ["software", "dev"]
author = "codecowboy.io"
+++

# Intro

I've been doing a lot recently with what is known in the industry as "shifting left" with security. Essentially this is the concept of moving security related checks closer to the beginning (or the **left**) of the software development lifecycle.

As I've been doing some of this with work, it has raised a number of questions for me personally about moving security checks earlier in the software development lifecycle. 

Questions like: 

- Why do it?
- What does this mean to the developer?
- Does it impact developer velocity?
- Is there a balance between speed and security?
- What does this balance look like?

In this post I'm going to explore some concepts about embedding security into a pipeline.


# Why put security in your pipeline?

The big question is **why do this at all?** Why put security into your pipeline at all? There are some really good reasons to do this. 

## Typical development flow

Typically when I've been developing software, I've been using a process something like this. Sometimes less complex, sometimes more, but the basics of the process I use are in the diagram below.

{{<mermaid align="left">}}
graph LR;
    A[development] --> B[build]
    B[build] --> C[test]
    C[test] --> D[artifact]
    D[artifact] --> E[deploy]
    E[deploy] --> F[run]
    F[run] --> G[observe]
    G[observe] --> | refactor | A
{{< /mermaid >}}

The thing to notice here is that there are no real security checks built in as part of the development lifecycle. This also means that security checks aren't built into my pipelines. Security checks typically become a problem for post deployment.

It looks something like this:

{{<mermaid align="left">}}
graph LR;
    A[development] --> B[build]
    B[build] --> C[test]
    C[test] --> D[artifact]
    D[artifact] --> E[deploy]
    E[deploy] --> F[run]
    H[security] --> F
    F[run] --> G[observe]
    G[observe] --> | refactor | A
{{< /mermaid >}}

You can see that security concerns are typically only thought about post deploymen. The thing that I hear in the back of my head is "can we do and of this **before** deployment?"

- Disconnected
- Late in the process after deployment

## Make security a part of the process


## Catch problems earlier 
It means that you can catch protential problems earlier in the development cycle. Generally speaking this is a good thing. I say **generally** speaking because there are a few facets here to be considered. 

The first facet is that nothing is perfect. To wait until you have a **perfect** piece of code, or the security is **perfect** around a deployment is not really feasible. You need to have a balance, and achieve a mantra of **"I understand and accept those risks"**.

This approach is important becasue without this approach you will take an unyielding attitude to code quality. Yes that's right **code quality**.

## Security and code quality
Catching security problems earlier can increase code quality. An example I will cite here is very simple. Input validation on fields on a web form. Performing input validation means that the code is less susceptible to security problems like SQL injection. 

This is a form of code quality **AND** security.

I am sure most people have seen the xkcd "cartoon of little johnny tables", where a parent has named their child "drop table;". 

This is a code quality problem but it is also a security problem. Embedding greater test coverage and checking for problems earlier is better.

# What does it mean to the developer?
In terms of the developer this can mean a few different things. 

It can mean additional unit tests.
It can mean additional steps in a pipeline before deployment.
It can mean additional hooks that are **pre commit**.

As a developer I want to ensure that security is operating earlier in my development lifecycle. 

## Pre commit vs Post Commit

There's a lot of talk about shifting left in security circles and a lot of this focuses on doing things in pipelines. Most pipelines are built to run **POST COMMIT**. That is to say, all of my actions in my pipeline happen after my code has hit my repository. For many things, this is a good idea:
- Linting
- Tests
- Security checks on built (but not deployed) artifacts

It is a bad idea for secrets. 

Pre commit hooks are really where you want to check for secrets. That is to say, before they have been checked into a source code repository. I don't want my AWS keys for example, to be checked via a pipeline **AFTER** they have been checked into my repo.

## Post Commit
Most tooling and other talks that I have seen have a flow that looks something like this.

{{<mermaid align="left">}}
sequenceDiagram
    Local Dev->>+Local Dev: Commit code
    Local Dev->>+Remote Repo: Check in code via push to remote
    Remote Repo->>+Remote Repo: Perform secret scan
    Remote Repo->>+Local Dev: Notify you're an idiot and checked secrets in 
{{< /mermaid >}}

All of the automated checks are running after the code has been commited to a remote repository. This is not desirable from a security perspective if I want to check for secrets. Using the flow above, the secrets would already be in my repository, along with all of the pain that this is going to cause me in terms of getting them out of my commit history and so on.
**This is not desirable from a security perspective**.

## Pre Commit
From a pre commit perspective, performing security checks before my code is checked into a source code repository looks something like this. 

{{<mermaid align="left">}}
sequenceDiagram
    Local Dev->>+Local Dev: Commit code
    Local Dev->>+Local Dev: Pre commit hook performs secret scan
    Local Dev->>+Remote Repo: Check in code via push to remote
{{< /mermaid >}}

Using a pre commit hook, no secret is every pushed to my repository. The check runs before commit, and I can rest assured that only "clean" code gets pushed to my remote repo.

This is much more desirable and means I do not check secrets into my remote repo.

## What about velocity?

This is great Scott, but what about developer velocity? 

I hear this a lot.

Like anything it's a balance. Don't put everything into your pipeline - there's no need for emoji's and other things (even though they're cool). 

- Have only the steps you need in your pipeline.
- Be conscious of the impact on pipeline run times.
- Don't put things into your pipeline for the sake of it.
- Choose wisely what you want to run pre commit.

The last one is really important, because let's face it, developer laptops are usually powerful beasts and being able to choose to run something locally before commit on a powerful beast of a laptop is going to make your life easier.

There is also a voice in my head that says speanding an extra minute checking for secrets saves me two hours or explanations and fixing git commits later.

**That's the important part**

Don't let expedience get in the way of saving yourself time later on.

# What should go in your pipeline?
This is a very subjective question. I think that you should put things into your pipelines that allow you to have a higher level of code quality. This include security checks / sweeps. 

Let's face it, I don't keep up to date with every single development in every single library or every single operating system or base image that I use as part of my development process. 

This is where external tooling comes in. External tooling and checks can do a lot of this heavy lifting for me.

I think that the things you should put into your pipline are things like:

- Pre checks
- Linting
- Build checks
- Code quality checks
- Tests <-- seriously get on board!
- Security checks
- Anything that makes your life as a dev faster

# What shouldn't go in your pipeline?
There are a number of things that you probably should not have in your pipeline.

- Things that slow you down
- Things that can be safely captured post deployment

I've seen all manner of weird things in people's pipelines. I've even seen observability on unit tests - **don't do it**. That's a testing function, let your test library handle it!!!!

Anything that can be captured post deployment definitely should not be in your pipeline.

Don't put things into your pipeline that are unnecessary. 

**BE RUTHLESS ABOUT WHAT GOES INTO YOUR PIPELINE**

# Conclusion
Moving security checks earlier in your pipeline is a good thing to do. Make sure that you consider pre commit hooks as well as post commit hooks. As I mention above, it's no good checking for secrets **AFTER** you've checked code into a remote repo.

Look out for my next post where I talk through setting up pre commit hooks and using a tool to perform secrets scans.

