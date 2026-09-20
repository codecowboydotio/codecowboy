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

![](/images/git-agentic-new-routine.png)

From here I create a new routine. 

![](/images/git-agentic-create-routine.png)

As part of my new routine, I give it a name. In my case, I am going to ensure that all of my pull requests update the README.md in my repo. There are three actions that I need to take at this point:
- Add a prompt
- Select a repository
- Select a trigger

In my case, the prompt gives instructions on what to do with the README. I select one of my repositories, which in my case, is my space invaders game. Lastly, I select a trigger. In my case, I am selecting a github pull request. 

### Triggers
There are three types of triggers. Each one has different use cases.

**Schedule:** This trigger type is a more simple type, where you want to run an agentic workflow on a schedule. For example "Search my google calendar and send me an email summarising my day". You could schedule this for 7am each morning.

**Github event:** This is where you can choose a github event type and automatically have the workflow kick off. In my case I chose a pull request as the trigger. Other out of the box event types are PR merged, Release published and Issue opened.

**API:** This is probably the most flexible. It gives you a URL and a token that means the routine can be called via a POST request. This is incredibly flexible because it can be embedded in other code, or an external orchestrator. 

### Notifications
There is also an option to include notifications as part of the routine. The default options for notifications are: 

**Push Notifications:** Send notifications to the claude app. This works for both the mobile and desktop apps.

**Email:** Send an email notification to the email in your account.

**Slack:** Send a slcak message (requires slack to be connected)

## Workflow created
Once the workflow has been successfully created, you can see both the workflow details on the right hand side panel, but also any runs that have occurred.

![](/images/git-agentic-after-create.png)


The very first time that you run you may see a note that the github app is not installed. Click on the link to install the github app and grant the appropriate permissions.

![](/images/git-agentic-github-app-not-installed.png)


Once the app is authorised and you have selected a repository, then click install.

![](/images/git-agentic-install-and-authorise.png)


After everything is installed, you should be redirected back to the installed connectors page in your claude web browser. This should now show that the github integration is installed.

![](/images/git-agentic-installed-connectors.png)


## Creating a PR
Now that my routine is created inside claude, and will run inside Anthropic, not on my laptop, it's time to create a pull request on my repo. I'm going to use the web based interface from github just to prove that nothing is running on my laptop.

The first thing to note is that there is no README present in the repository at all.

![repo files](/images/git-agentic-github-repo-1.png)

I add a single comment to the shell script that I use to kick off the web server and serve the html space invaders game. This is kicked off using python's inbuilt **http.server**. This is not really very efficient or robust, but does work for local testing. I simply add a comment to this effect to the file.

![comment in file](/images/git-agentic-github-repo-2.png)

I commit this to a new branch and start a new pull request. This should kick off my workflow and my routine. Remember that the trigger i defined earlier was a new PR being created. 

![create PR](/images/git-agentic-github-repo-3.png)

I them create the pull request.

![create PR](/images/git-agentic-github-repo-4.png)

After a short amount of time, the pull request is updated with the information from my routine. I can see that there is a new section within the PR that says "Add comprehensive Readme". This is exactly what my routine has done. 

![routine in pull request](/images/git-agentic-github-repo-5.png)

When I look at the actual content of the pull request, I can see that a new file called README.md has been added and the content shows that a comprehensive readme describing the code has been added.
![readme content](/images/git-agentic-github-repo-6.png)

## Routine
What does the routine look like after it's been fired by the pull request? The image below shows that the routine has fired, a commit has been added to the PR and all of the normal claude reasoning and tool use is available.

![claude routine](/images/git-agentic-github-repo-7.png)

## Using an API trigger
What if I want something to run via an API trigger instead? Adding an API trigger is very easy. I can simply edit the existing routine. When I edit the routine, I have the option to add an additional trigger.

![claude routine](/images/git-agentic-github-repo-8.png)

Adding the new trigger gives me a URL that's specific to that trigger. I also get a token that I can use to call the trigger. Helpfully, I can also expand the option for an example curl request. This is really helpful when I want to test, and has all of the appropriate headers set for me.

![claude routine](/images/git-agentic-github-repo-9.png)

I call the routine and pipe the output through jq so that it is formatted nicely. This gives me a session id, a url and a type field, which in this case is "routine_fire". I can use this method to fire the routine from anywhere, be it an external orchestrator, or a a terminal session using curl. The idea here is obvious, I not have the ability to fire agentic routines from anywhere.

![claude routine](/images/git-agentic-github-repo-11.png)

Once I have called the trigger, I get output in the claude code routines panel that shows the routine has run an in this case, it has done nothing. As my routine is tied to an existing pull request, nothing will occur if there is no pull request outstanding.

![claude routine](/images/git-agentic-github-repo-10.png)

## Additional data
If you noticed, the example curl request presented had a rather interesting part to it. The request has a data section that says **"optional extra turn turn appended to the session"**. 

I thought to myself "I wonder if I could just add another turn to this", so I tried it.

{{<notice info>}}
-H "Content-Type: application/json"   -d '{"text": "if no pull request exists, then validate the code against the readme anyway. If no readme exists, create it. Make a new PR is needed."}'
{{</notice>}}

## Summary
Claude code routines are a neat way of interacting with your github repo. I am still not sure whether or not this kills off actions or not. I think ultimately, actions will become a lot more simple. The ability to fire a routine from a PR, or even a generic API request adds a level of flexibility that allows for integration into the modern developer lifecycle. 

I can fire a routine associated with my codebase on a schedule, from a github event or via an API call. I can do this from github, an external orchestrator or anything else. Imagine a use case where I perform a regular security scan across my codebase and send the results to slack or use the results to fire off a code review and further actions on my codebase. 

Routines let me get creative with the actions and activities that can occur on my codebase.

