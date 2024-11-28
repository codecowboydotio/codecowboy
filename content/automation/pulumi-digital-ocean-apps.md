+++
title = "Pulumi - creating an app in digital ocean using the app platform"
date = "2024-11-30"
aliases = ["pulumi_k8s_digital_ocean"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev", "serverless"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
As many of you know, I've been using Pulumi for a while and I am a big fan.
Today, I'm going to walk through creating an application using the digital ocean app platform.

## Why digital ocean?
A lot of people will wonder **"why digital ocean"??** the answer is simple.
It's extremely well integrated, and this makes it very compelling to use. 

## The digital ocean app platform
Today I'm using the digital ocean app platform. It's a PaaS service that allows me to publish my application to it. 

The part that I like about it is that I can pull my app directly from github.

# Initial Setup

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
pulumi new digitalocean-python --name do-app --description "Digital Ocean app platform example"
```

You should see some output that looks like this:

```Shell
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

Created project 'do-app'

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

### Configure an API key
In order to get Pulumi to use digital ocean, you will need to have an API configured within digital ocean.
Log in to digital ocean and click on the API menu

![API menu](/images/do_API_menu.jpg)

Then click on generate key.
Give the key a name, an expiration and a scope.
The scopes can be used to limit access, or you can give full access. 
I have chosen full access, however I do recommend limiting the scope for practical purposes.

![Generate API key](/images/do-generate-api-key.jpg)

At this point I highly recommend copying the token / API key and placing it in a variable within your shell environment.

```Shell
export DIGITALOCEAN_TOKEN=dop_<your key>
```

{{< notice info >}}
This is the easy way without using Pulumi secrets.
{{< /notice >}}

# Pulumi Program
I have chosen python as my starting point for today, so let's take a look at what the command we ran earlier actually created for me.
If you remember, we ran a **pulumi new** command to create a scaffold for us.

Within this scaffold, there will be a few files that have been created.

```Shell
 .gitignore
 __main__.py
 Pulumi.dev.yaml
 Pulumi.yaml
 __pycache__
 requirements.txt
 venv
```
The files that we are going to focus on are:
- Pulumi.dev.yaml - this contains some configuration options that we will set.
- __main__.py - this is the main pulumi program that does the work of creating our droplet.

## Configuration options
When we look at configuration options, I am using Pulumi's stack level configuration to store options.
I have a habit of storing configuration options within the stack, these are things that I can change and don't want to necessarily hard code in my codebase.

I can see these using the following command:

```Shell
pulumi config
``` 

I can see that I have defined a number of variables underneath the **cfg** namespace, which shows as a nested value.
I use nested values as a default, so that I can store multiple configuration values with different options.

```Shell
KEY               VALUE
cfg:app-name      python-app
cfg:app-size      apps-s-1vcpu-1gb
cfg:branch        main
cfg:service-name  py-service
pulumi:tags       {"pulumi:template":"digitalocean-python"}
```

### Setting configuration values
In order to set my configuration values I do the following:

```Shell
pulumi config set cfg:app-name python-app
```

This sets the configuration value of **python-app** to **app-name** within the **cfg** namespace.
I can then reference this in my codebase.

## Pulumi program
In order to get going, I need to import some libraries and pull in my stack level configuration.

```Shell
"""A DigitalOcean Python Pulumi program"""

import pulumi
import pulumi_digitalocean as do


## Import configuration variables
import pulumi
import pulumi_digitalocean as digitalocean

stack_config = pulumi.Config("cfg")
var_app_name = stack_config.require("app-name")
var_app_size = stack_config.require("app-size")
var_service_name = stack_config.require("service-name")
var_branch = stack_config.require("branch")
```

In the code above, I import the pulumi modules, but also import my stack configuration. These are the configuration values that I set using the **pulumi config set** command earlier. 

These are variables that I may want to change over time, and are maintained outside my codebase. 
These variables are imported and referenced within my pulumi codebase. 

## Deploy an application
In order to deploy an application you will need to have an application hosted within a repository, or within a container registry. I am going to use a sample application that is hosted within github.

## The application
The application that I am deploying is based on the pulumi example for python. It's a relatively simple application that takes the HTTP path component and prints a message that says **"Hello! you requested /"**, or whatever URL was requested.

The application is simple, but serves the purpose of having an application that is deployable, and works.

## The application code
The following is the code of the application.

```Python
import os
import http.server
import socketserver

from http import HTTPStatus


class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(HTTPStatus.OK)
        self.end_headers()
        msg = 'Hello! you requested %s' % (self.path)
        self.wfile.write(msg.encode())


port = int(os.getenv('PORT', 80))
print('Listening on port %s' % (port))
httpd = socketserver.TCPServer(('', port), Handler)
httpd.serve_forever()
```

The code sets up an HTTP server that listens for incoming requests and then returns the **PATH** of the **GET** request that was sent to it.

## App platform specifics
There are a few things that you will need to do in oder to let the digital ocean appl platform know about your application. For example, you will need to let the digital ocean platform know what type of application it is - is it a python application, or a golang application?

### Application type
For a python application, the digital ocean platform will look for the following files:

- requirements.txt
- PipFile
- setup.py

These are the triggers to let the app platform know that you're running a python program.

This is referenced here: 
[https://docs.digitalocean.com/products/app-platform/reference/buildpacks/python/](https://docs.digitalocean.com/products/app-platform/reference/buildpacks/python/)

Other languages can be found here: 
[https://docs.digitalocean.com/products/app-platform/reference/buildpacks/](https://docs.digitalocean.com/products/app-platform/reference/buildpacks/)

In my case, I have used an empty **requirements.txt** file to indicate that I am running a python application.

### Application runtime
Similarly, the application runtime can be chosen by including a file named **runtime.txt**.
In my case this has a single line like the following:

```Shell
$ more runtime.txt
python-3.12.2
```

The versions that can be used are listed in the python buildpack reference document above.

### Alternatives
It is also possible to configure these parameters by using a yaml file. Digital ocean call this an **"App Specification"**. This is essentially a yaml file that is placed within the repo under a directory named "**.do/app.yaml**". This will be recognized by the app platform and used as your applications specification.

This is a more flexible way of configuring your application, however, for the sake of simplicity, I will use the defaults.

## The full code

# Conclusion
