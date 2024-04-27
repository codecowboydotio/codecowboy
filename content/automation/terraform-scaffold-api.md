+++
title = "Terraform Scaffold: AWS"
date = "2024-05-18"
aliases = ["terraform_scaffold"]
tags = ["terraform", "iac"]
categories = ["automation", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I do a lot of building various platforms for various reasons. I do this as part of my day job, I also do it as part of some open source projects that I work on. 
I work through building various stacks and infrastructure for things. 

Today I am going to share my scaffold for AWS API gateways and lambda functions using terraform. This has served me well, and can be adapted to different styles of deployment. 

## Why is this important?
I find this to be important as a pattern because it allows me to rapidly test integrations with various platforms. I have used this scaffold and pattern to test integrations with third parties as diverse as cloudflare, and other custom written applications.

## The architecture
The architecture for this scaffold is an API gateway, that points all routes to a single lambda function. The lambda function can be anything, but in my default scaffold, I have a lambda function that takes JSON imput, and then passes it on to another endpoint. 

This is useful for integration testing.

####################3
#DIAGRAM
#####################

## The basic scaffold

The basic scaffold that I share here is agnostic and sets up the "base AWS infrastructure" that I tend to use in each project. There are some tactics and techniques I use here that are interesting and have stood the test of time. Many of these have been developed over time.

## A starting point.
My standard scaffold looks like this:


├── aws.tf
├── bucket.tf
├── lambda.tf
├── tags.tf
├── vars.tf
├── lambda_code/webhook.py


Each of the files contains various basic AWS components. 

- aws.tf - As this is a project that uses other services this only contains a variable for the region that I want to deploy in.

- bucket.tf - The bucket is used to hold the lambda code that is being used as part of the scaffold. 

- lambda.tf - This creates the lambda function and connects all of the required roles for execution.

- tags.tf - This contains a map of tags that I have to be able to change tags on a per project basis.

- vars.tf - This file contains all of the variables that I use as part of spinning up a new project using my scaffold.

- lambda_code/webhook.py - This is the lambda code that does all of the work.

## Variables
Variables are worth spending a little time talking about.
I use variables for different purposes. 

Some of them are just standard variables, others relate to the project.
There are standard ones like the region, and so on, but I use two that are worth mentioning. 

### Naming Variables

I use the concept of a "name prefix" and a "project".

Examples of these are below.

```Shell
variable "name-prefix" {
  default = "test"
}
variable "project" {
  default = "scaffold"
}
```
In essence, these become the basis of my project. 
This allows me to name a project and also use a prefix like "test" or "prod" or "pre-prod".
When I tag resources for example, I use the format below. 

```Shell
 Name = "${var.name-prefix}-${var.project}-vpc-a"
```

This resource would be named "test-scaffold-bucket-a".

This is a very flexible way of naming and means that if I change the variables, the names of all of my resources change. 

#######################################################################################################
### API Gateway

I create an API gateway, specifically a REST api gateway. 
I tell the API gateway to 


```Shell
resource "aws_api_gateway_rest_api" "gw" {
  name = "Orca-Webhook"
  description = "api_scaffold webhook example"

  tags = {
    for k, v in merge({
      app_type = "production"
      Name = "${var.name-prefix}-${var.project}-api-gw"
    },
    var.default_ec2_tags): k => v
  }
}
```

I then create two proxy resources. 
These essentially allow access to the gateway from the outside world.

```Shell

resource "aws_api_gateway_resource" "proxy" {
   rest_api_id = aws_api_gateway_rest_api.orca.id
   parent_id   = aws_api_gateway_rest_api.orca.root_resource_id
   path_part   = "{proxy+}"
}
resource "aws_api_gateway_method" "proxy" {
   rest_api_id   = aws_api_gateway_rest_api.orca.id
   resource_id   = aws_api_gateway_resource.proxy.id
   http_method   = "ANY"
   authorization = "NONE"
}
```


I then create an integration between my API gateway proxy resources and the function that I create. 
```Shell
resource "aws_api_gateway_integration" "lambda" {
   rest_api_id = aws_api_gateway_rest_api.orca.id
   resource_id = aws_api_gateway_method.proxy.resource_id
   http_method = aws_api_gateway_method.proxy.http_method
   integration_http_method = "POST"
   type                    = "AWS_PROXY"
   uri                     = aws_lambda_function.orca_webhook.invoke_arn
}
```

### Tags
Tags are a very interesting topic and I've written about them before 

HERE

I do two things with tags.

- I use a map to hold standard or default tags
- I apply other tags as needed

```Shell
variable "default_ec2_tags" {
  description = "Default set of tags to apply to EC2 resources"
  type        = map
  default = {
    Owner       = "me"
    Environment = "Demo"
    SupportTeam = "Terraform-test-team"
    Contact     = "me@example.com"
  }
}
```

The very first thing I do is create a map variable. This is actually contained in a separate file, mainly because I feel that tags should be. 

The map contains a number of default tags that will be set on all resources.
Below I have a VPC resource, this gets created with every single project, and has the tags applied. 
Note that the tags are applied as a map AND there are individual tags being applied as well. 


```Shell
resource "aws_vpc" "vpc-a" {
  cidr_block       = var.vpc-a_cidr_block
  instance_tenancy = "default"
  enable_dns_support = "true"
  enable_dns_hostnames = "true"

  tags = {
    for k, v in merge({
      app_type = "web-test"
      Name = "${var.name-prefix}-${var.project}-vpc-a"
    },
    var.default_ec2_tags): k => v
  }
}
```

The for loop in this case merges both the tags that I have explicitly set, as well as the tags from my map. The map is read in as a key value pair and applied to the resource. 

## Virtual Machine
My template includes a virtual machine. If you don't need one, just remove it. That's fine.

I have an unusual way of interacting with my virtual machine. 
I use user_data to pass a shell script that runs on the machine and installs everything.
I know that this is not the packer / terraform way. 
I am conflicted about it. 

I find this useful because I am running up things to test every day, and they are necessarily different things. As a result, I find it easier to modify a template user_data file with some simple shell commands rather than spending time customizing resources for each specific use case. 

```Shell
resource "aws_instance" "web-server-az2a" {
  ami           = data.aws_ami.distro.id
  instance_type = var.instance_type_linux_server
  subnet_id = aws_subnet.vpc-a_subnet_1.id
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.vpc-a_allow_all.id]
  user_data = templatefile("web-server-az2a.sh.tpl", {
     linux_server_pkgs = var.project
  })

  tags = {
    for k, v in merge({
      app_type = "web"
      Name = "${var.name-prefix}-${var.project}-az2a-web-server"
    },
    var.default_ec2_tags): k => v
  }
}
```

You will note that the user_data file is just a shell script. 
I have made it a template so that I can also use variables inside it if I want to.

This actually speeds up my life considerably in many ways. 

Want to install gitlab - half a dozen shell commands
Want to install apache - one shell command
Want to clone a repo and play with some code - three or four shell commands

{{< notice info >}}
I do not need to change the terraform code at all. I just change the shell script template for different use cases.
{{< /notice >}}

I quite like the pragmatic flexibility of operating like this. This is particularly the case given I typically spin up and tear down things regularly for development purposes.

## Extending the scaffold
I have had occasion to extend the scaffold such that it includes buckets, has azure resources and so on. 
One of the big things that I learned with buckets is that the name needs to be completely unique. 
This presents a problem when you have multiple people using the same scaffold.

In this case you can use the random string resource. 
This will generate a random resource that allows you to re-use the same template, again, without needing to modify the template.


## Why do things this way?
I am a big believer in standardising things. I am also a big believer in idempotency - **across projects**. 
I want to make sure that I can simply clone the scaffold, do nothing but change the variables for the name, and have a guarantee that I will get my standard set of infrastructure up and running.

## The github repo

You can clone this at the github repo here:

This repo will grow over time to include other cloud providers and use cases in terraform.


## Conclusion
Using a standard template as a scaffold that you can extend is a lot of fun and makes my setups and experimentation easier.


