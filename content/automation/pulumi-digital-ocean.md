+++
title = "Pulumi - creating cloud infrastructure in digital ocean"
date = "2024-11-02"
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
pulumi new digitalocean-python --name blog_post --description "blog post tutorial"
```

You should see some output that looks like this:

```Shell
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

At this point I highly recommend copying the token / API key and placing it in a variable within your shell environment.

```Shell
export DIGITALOCEAN_TOKEN=dop_<your key>
```

{{< notice info >}}
This is the easy way without using Pulumi secrets.
{{< /notice >}}

### Configure and SSH key
Next, we will configure an SSH key in digital ocean, so that we are able to log into our newly created droplet. 
We strictly don't need to do this, but it's useful for completeness.

Within the digital ocean portal, go to the settings menu.
![Settings Menu](/images/do-settings-menu.jpg)

Then click on the security tab.
Once you're there, click on the create ssh key button.

There are detailed instructions within the dialogue that pops up telling you how to create / import an ssh key.
![SSH key](/images/do-add-ssh-key.jpg)

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
Essentially the configuration is namespaced.

```Shell
KEY               VALUE
cfg:image-name    ubuntu-24-04-x64
cfg:ssh-key-name  digital-ocean-key
cfg:vm-name       do-vm
pulumi:tags       {"pulumi:template":"digitalocean-python"}

```

### Setting configuration values
In order to set my configuration values I do the following:

```Shell
pulumi config set cfg:vm-name do-droplet
```

This sets the configuration value of **vm-name** to **do-droplet** within the **cfg** namespace.
I can then reference this in my codebase.

## Pulumi program
In order to get going, I need to import some libraries and pull in my stack level configuration.

```Shell
"""A DigitalOcean Python Pulumi program"""

import pulumi
import pulumi_digitalocean as do


## Import configuration variables
stack_config= pulumi.Config("cfg")
var_ssh_key_name= stack_config.require("ssh-key-name")
var_image=stack_config.require("image-name")
var_vm_name=stack_config.require("vm-name")
```

In the code above, I import the pulumi modules, but also import my stack configuration. These are the configuration values that I set using the **pulumi config set** command earlier. 

These are variables that I may want to change over time, and are maintained outside my codebase. 
These variables are imported and referenced within my pulumi codebase. 

...yes I have some interesting habits, but some of these are also for readability within a blog.

## Create the droplet
In order to create the droplet, I need to supply a number of configuration parameters to the **do.Droplet** interface. 
These are: 
- name: The name of the droplet in the web interface.
- image: The image to use for the droplet - see here: [https://docs.digitalocean.com/products/droplets/details/images/](https://docs.digitalocean.com/products/droplets/details/images/) 
- region: The region where you would like your droplet to live - see here: [https://docs.digitalocean.com/platform/regional-availability/](https://docs.digitalocean.com/platform/regional-availability/)
- size: The size of the droplet - see here: [https://slugs.do-api.dev/](https://slugs.do-api.dev/) 
- ssh_keys: The id of the ssh key (in order to get this, you perform a lookup by name, and then use the **id** method)

```Shell
# Get my ssh key identifier
ssh_key= do.get_ssh_key(name=var_ssh_key_name)

# Create a web server
web = do.Droplet("web",
    name=var_vm_name,
    image=var_image,
    region=do.Region.SYD1,
    size="s-1vcpu-512mb-10gb",
    ssh_keys=[ssh_key.id],
)

pulumi.export('public_ip', web.ipv4_address)
```
Lastly, I print out the public IP address of my droplet so that I can connect to it.

## The full program
The full program is: 

```Shell
"""A DigitalOcean Python Pulumi program"""

import pulumi
import pulumi_digitalocean as do

## Import configuration variables
stack_config = pulumi.Config("cfg")
var_ssh_key_name = stack_config.require("ssh-key-name")
var_image=stack_config.require("image-name")
var_vm_name=stack_config.require("vm-name")

# Get my ssh key identifier
ssh_key = do.get_ssh_key(name=var_ssh_key_name)

# Create a web server
web = do.Droplet("web",
    name=var_vm_name,
    image=var_image,
    region=do.Region.SYD1,
    size="s-1vcpu-512mb-10gb",
    ssh_keys=[ssh_key.id],
)

pulumi.export('public_ip', web.ipv4_address)
```

## Run it
Now let's run it. 

In order to run this and see if a droplet is created, I simply run the command

```Shell
pulumi up -y
```

This should create my droplet and print out its IP address for me.

### The output from the cli
After I have run my command, I can see the output from the CLI

```Shell
pulumi up -y
Previewing update (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/XXXX

     Type                           Name            Plan
 +   pulumi:pulumi:Stack            do-droplet-dev  create
 +   └─ digitalocean:index:Droplet  web             create

Outputs:
    public_ip: output<string>

Resources:
    + 2 to create

Updating (dev)

View in Browser (Ctrl+O): https://app.pulumi.com/XXXX

     Type                           Name            Status
 +   pulumi:pulumi:Stack            do-droplet-dev  created (27s)
 +   └─ digitalocean:index:Droplet  web             created (23s)

Outputs:
    public_ip: "XX.XX.XX.210"

Resources:
    + 2 created

Duration: 35s
```

I can also validate this within the web browser in the digital ocean portal.
The droplet has the name that was part of my configuration, as well as the size and is located in the region that I specified. 

![Droplet Info](/images/do-droplet-info.jpg)

# Conclusion
Pulumi works well with digital ocean, and has a great level of configurability while maintaining simplicity of configuration.
I highly recommend having test of pulumi with digital ocean!

Look out for the next post where I will test out pulumi with digital ocean's kubernetes infrastructure.