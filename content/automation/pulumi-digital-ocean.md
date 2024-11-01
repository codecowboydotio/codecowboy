+++
title = "Pulumi - creating cloud infrastructure in digital ocean"
date = "2024-11-01"
aliases = ["pulumi_digital_ocean"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
As many of you know, I've been using Pulumi for a while and I am a big fan.
Today, I'm going to walk through creating resources in digital ocean.

## Why digital ocean?
A lot of people will wonder **"why digital ocean"??** the answer is simple.
It's cheap, fast, and has most of the services I need. 

As always, as this is my first digital ocean post, I will start with a simple virtual machine, or a droplet in digital ocean terminology.
I will eventually walk through other services with digital ocean, but I think a virtual machine is a simple and good enough place to start.

## Setting up the pulumi project
There are some digital ocean templates that are available as part of Pulumi out of the box.

```Shell
pulumi new --list-templates | grep digitalo
  digitalocean-go                    A minimal DigitalOcean Go Pulumi program
  digitalocean-javascript            A minimal DigitalOcean JavaScript Pulumi program
  digitalocean-python                A minimal DigitalOcean Python Pulumi program
  digitalocean-typescript            A minimal DigitalOcean TypeScript Pulumi program
  digitalocean-yaml                  A minimal DigitalOcean Pulumi YAML program
```

I am going to choose python for this this exercise. 

We create a new template for our project using the new command.

```Shell
pulumi new digitalocean-python --name blog_post --description "blog post tutorial"
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

Created project 'blog_post'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
Stack name (dev):
Created stack 'dev'

The toolchain to use for installing dependencies and running the program pip
Installing dependencies...

Creating virtual environment...
```

Once we have entered the stack name, the project gets created, and all of the dependencies are configured and set up.
This doesn't take very long.

## Configure the Digital Ocean side
In order to get working there are two things that you will need to do.
- Configure an API key within digital ocean.
- Configure an ssh key (optional) 

Strictly, the SSH key isn't required, however, you wont be able to access your droplet if you don't do this.

### Configure an API key
In order to get Pulumi to use digital ocean, you will need to have an API configured within digital ocean.
Log in to digital ocean and click on the API menu

![API menu](/images/do_API_menu.jpg)

Then click on generate key.
Give the key a name, an expiration and a scope.
The scopes can be used to limit access, or you can give full access. 
I have chosen full access, however I do recommend limiting the scope for practical purposes.

![Generate API key](/images/do-generate-api-key.jpg)

### Configure and SSH key
![Settings Menu](/images/do-settings-menu.jpg)

![SSH key](/images/do-add-ssh-key.jpg)
