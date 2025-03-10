+++
title = "Pulumi - creating an API gateway and lambda function in AWS"
date = "2025-03-07"
aliases = ["pulumi_javascript"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
I've been fooling around with **pulumi** for a bit now, and thought I would write about it here. 
I have had an API gateway that is backed by a lambda function in terraform for quite some time. I use it quite frequently to spin up infrastructure when I need to perform testing on integrations. I eventually decided to re-write this in pulumi. 

## What am I building?
I am building out a few things today. 
- An API gateway
- Routes that proxy to a lambda function
- A lambda function that accepts an incoming request and sends it somewhere else.

Essentially this is a pattern that allows me to quickly build out new integrations with REST API's. 
I can send arbitrary data to my API gateway and lambda function, perform transformations, and then send the data on to another API.

![AWS API Gateway](/images/aws_api_gateway.jpg)

## Why?
The rationale for doing this is something like this:

As a user I want to send data from SaaS application X to SaaS application Y where there is no direct integration.

I work with a lot of different SaaS based tools, and it is common that these tools support a webhook to facilitate sending data to external systems. This API gateway and lambda function allows me to take any incoming data, format it appropriately and send it to any other system.

Over the years (yes years) that I've been using this, I have successfully integrated a large number of downstream SaaS platforms where no *direct* integration exsited. It's just a matter of changing the lambda function on a case by case basis.

## The Project 

### Setting up the project
There are some AWS templates that are available as part of Pulumi out of the box.

```Shell
pulumi new --list-templates | grep aws
  aws-csharp                         A minimal AWS C# Pulumi program
  aws-fsharp                         A minimal AWS F# Pulumi program
  aws-go                             A minimal AWS Go Pulumi program
  aws-java                           A minimal AWS Java Pulumi program
  aws-javascript                     A minimal AWS JavaScript Pulumi program
  aws-python                         A minimal AWS Python Pulumi program
  aws-scala                          A minimal AWS Scala Pulumi program
  aws-typescript                     A minimal AWS TypeScript Pulumi program
  aws-visualbasic                    A minimal AWS VB.NET Pulumi program
  aws-yaml                           A minimal AWS Pulumi YAML program

```

I am going to choose python for this this exercise.

We create a new template for our project using the new command.

```Shell
pulumi new aws-python --name api-gw --description "API gateway tutorial"
```

You should see some output that looks like this:

```Shell
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

Created project 'api-gw'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
Stack name (dev):

The toolchain to use for installing dependencies and running the program pip
Installing dependencies...

Creating virtual environment...
```

Once we have entered the stack name, the project gets created, and all of the dependencies are configured and s
et up.
This doesn't take very long.


### Variables
I like to use variables for everything, this way it's easy to rename a project and the resources that are created easily. This project is no different.

I am storing these within the pulumi stack config. Running the command **pulumi config** will show me all of the variables that exist within my current stack.

```Shell
pulumi config

KEY                VALUE
aws:region         us-east-1
aws-native:region  us-east-1
lambda-name        webhook-lambda
name-prefix        svk
project            test-webhook
```

### Lambda function
The lambda function creation is quite simple using pulumi.

```Python
lambda_func = aws.lambda_.Function(
  name_prefix + "-" + project + "-" + var_lambda_name,
  name=name_prefix + "-" + project + "-" + var_lambda_name,
  role=iam.lambda_role.arn,
  runtime="python3.11",
  handler="webhook.lambda_handler",
  code=pulumi.AssetArchive({".": pulumi.FileArchive("./app")}),
)
```

We define a lambda function with a name, a role, a runtime, a handler within my codebase and a location for the codebase.

- name: This is the name os the lambda function within AWS. I am using variables to set this, including the project name, the lamnda function name and project prefix. All of these are variables that are read in from the project configuration.
- role: This is the role that I have configured that allows the lambda function to run. I discuss this in a separate section below.
- runtime: This is the runtime version that my lambda function uses.
- handler: The is the handler within my code. AWS lambda functions require a default handler. For me, I am calling this is the filename and function name of my code.
- code: This is the directory where my codebase is stored. There are a lot of different ways to provide this. It can be zipped, uploaded to buckets and so on. The simplest way in pulumi that I have found is to simply create an archive of the directory where the code resides. This has the added benefit of also including any libraries I wish to include.


### API Gateway
To instantiate the API gateway, it's relatively simple, use the **aws.apigateway.RestApi** object and give it a name and a description.

By default this will create an API gateway but will not create any routes or methods - just the API gateway object.

```Python
rest_api = aws.apigateway.RestApi(
    name_prefix + "-" + project + "-api-gw",
    name=name_prefix + "-" + project + "-api-gw",
    description="svk-test-gw",
)
```

### API resources

Next we need to create some resources. These are three things:

- Resource: This represents the path component and host it is handled. In my case I use the PROXY resource. 
- Method: This represents the HTTP method that the API gateway allows. In my case, I al allowing ANY method, or all methods with no authentication.
- Integration: The integration ties these together with my lambda function. While the API gateway allows all methods, at the front, the integration to my lambda function allows a POST.

![API Gateway logical view](/images/pulumi-api-gateway-logical.jpg)


```Python
rest_api_resource = aws.apigateway.Resource(
    name_prefix + "-" + project + "-proxy-resource",
    rest_api=rest_api.id,
    parent_id=rest_api.root_resource_id,
    path_part="{proxy+}")

rest_api_method = aws.apigateway.Method(
    name_prefix + "-" + project + "-method",
    rest_api=rest_api.id,
    resource_id=rest_api_resource.id,
    http_method="ANY",
    authorization="NONE")

rest_api_integration = aws.apigateway.Integration(
    name_prefix + "-" + project + "-integration",
    rest_api=rest_api.id,
    resource_id=rest_api_resource.id,
    http_method=rest_api_method.http_method,
    integration_http_method="POST",
    type="AWS_PROXY",
    uri=lambda_func.invoke_arn
    )
```

This section is vital to understand how things work.

I am using the proy method so that anything under my API path (in this case /) will be sent through without being changed. For example, /foo, /bar and /foo/bar would all be sent through to my lambda function. 

While it is possible to configure routes and methods in the API gateway, I prefer to do this in my code. 

It's a simple way of saying "my API gateway accepts everything, and my codebase is where the real work is done".

I'm sure that depending on your perspective, this may be an anti pattern for you.

### API deployment

### API stage

### IAM

#### Role

#### Execution Attachment

#### Invoke permissions


### Cleanup


## Conclusion
