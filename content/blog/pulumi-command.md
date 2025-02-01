+++
title = "Pulumi - the pulumi command"
date = "2024-07-30"
aliases = ["pulumi_command"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I've been fooling around with **pulumi** for a bit now, and thought I would write about it here. 
This post is about the pulumi command. It's useful to know how to use the command when you're spinning up varioup stacks.

## What is it?
Pulumi is an infrastructure as code framework with a twist. You can use any language you like in order to write your code (within reason). 

The pulumi tagline is **Your cloud. Your language. Your way.**

I've previously written about using pulumi and yaml and python. This particular post is about the pulumi command.


## Installation
Installation is simple, I am using AWS and will follow the docs at [https://www.pulumi.com/docs/get-started/aws/begin/](https://www.pulumi.com/docs/get-started/aws/begin/).

### Install pulumi
In order to install pulumi, just as the docs say, use the following command:

```Bash
curl -fsSL https://get.pulumi.com | sh
```

You will also need to install your language runtime - for me this is javascript.
If you're using an up to date version of ubuntu then you should have a relatively new version of javascript and everything should work fine.

Next configure your AWS account in the normal way that you would

```Bash
export AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID> 
export AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>
```

### Create a new project

In order to get up and going once pulumi is installed you will need to install a new project. 

To do this you use the commands:

```Bash
mkdir quickstart 
cd quickstart
pulumi new some-project
```

By default this will create a project and a javascript project template for you.


If you do not use any command line switches, then by default you will be asked for information about the stack that you are creating. 
```Bash
[root@fedora foo]# pulumi new
Please choose a template (30/221 shown):
 aws-javascript                     A minimal AWS JavaScript Pulumi program
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

project name: (foo)
project description: (A minimal AWS JavaScript Pulumi program)
Created project 'foo'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
stack name: (dev)
Created stack 'dev'

aws:region: The AWS region to deploy into: (us-east-1)
Saved config

Installing dependencies...

(⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂⠂) ⠧ idealTree:foo: sill idealTree buildDeps
```

This installs the required dependencies and also creates a boilerplate for you to use.



If you check in the pulumi web console, the stack is created automatically as well.

![Pulumi stack](/images/pulumi-python-blog.jpg)



### Cleanup


## Conclusion
Pulumi is a revolution in IaC.
I don't know how else to say it.

