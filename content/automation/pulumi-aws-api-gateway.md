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

## TLDR
If you just want the code - it's here: 

![API Gateway git repository](https://github.com/codecowboydotio/pulumi/tree/main/aws-serverless)


## What am I building?
I am building out a few things today. 
- An API gateway
- Routes that proxy to a lambda function
- A lambda function that accepts an incoming request and sends it somewhere else.

Essentially this is a pattern that allows me to quickly build out new integrations with REST API's. 
I can send arbitrary data to my API gateway and lambda function, perform transformations, and then send the data on to another API.

![API Gateway logical view](/images/pulumi-api-gateway-logical.jpg)

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

![AWS API Gateway](/images/aws_api_gateway.jpg)


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

### Stage and Deployment
In order to get your API gateway up and running you need two things - a stage and a deployment. They are linked, but are not the same thing.

A deployment in AWS terms is a way of managing API lifecycle. A eployment allows you to make changes to your API and then apply them to a stage. Nothing happens until the changes are applied to a stage.

A stage is a **named referance to a deployment**. In this way, configuration and rollout can be separated and managed and lifecycled separately.

No matter what you think of this, and the internet has plenty of opinions, you need to create both a stage and a deployment to get your API gateway to work.

#### API deployment
In order to create a deployment, the following code is needed. This sets up a single deployment that can be used to manage the API gateway.

```Python
deployment = aws.apigateway.Deployment(
  name_prefix + "-" + project + "-deployment",
  opts=pulumi.ResourceOptions(depends_on=[rest_api_integration, rest_api_integration_root]),
  rest_api=rest_api.id,
  stage_name="",
)
```

#### API stage
To create a stage that references your deployment, the code block below will achieve this. Note that I am using a **depends_on** option on my resource in order to ensure that the deployment is created first.

```Python
stage = aws.apigateway.Stage(
  name_prefix + "-" + project + "-stage",
  opts=pulumi.ResourceOptions(depends_on=[deployment]),
  rest_api=rest_api.id,
  deployment = deployment.id,
  stage_name = "test",
)
```

### IAM
The very last piece of the puzzle here is to give the correct AWS permissions to the gateway and the lambda in order for anything to be executed.

The code below attaches a role to the lambda function that allows the API gateway to invoke the lambda function. In addition to this I am also attaching the execution ARN of my API gateway object **here seen as **rest_api** as the source object that is given the permissions to invoke my lambda function. 

In other words, it's not everything that can invoke my lambda function, just my API gateway.

```Python
rest_invoke_permission = aws.lambda_.Permission(
  name_prefix + "-" + project + "-" + var_lambda_name + "-permission",
  statement_id  = "AllowAPIGatewayInvoke",
  action="lambda:invokeFunction",
  function=lambda_func.name,
  principal="apigateway.amazonaws.com",
  source_arn=rest_api.execution_arn.apply(lambda arn: arn + "/*/*"),
)
```

#### Role
When I first create the lambda runction I pass a role to the resource that creates the function. The role and execution attachment are below. The code creates a role that allows assume role for my lambda function, while also allowing the basic **AWSLambdaBasicExecutionRole** for my function.

```Python
lambda_role = iam.Role(
    name_prefix + "-" + project + "-lambda-role",
    name=name_prefix + "-" + project + "-lambda-role",
    assume_role_policy="""{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Effect": "Allow",
                "Sid": ""
            }
        ]
    }""",
)
```

#### Execution Attachment

```Python
execution_attachment = iam.PolicyAttachment(
    name_prefix + "-" + project + "-execution-attachment",
    name=name_prefix + "-" + project + "-execution-attachment",
    roles=[lambda_role.name],
    policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
)
```

### Lambda code
The core of all of this work is my lambda code.
The code takes the entire data portion or body and simply sends it as a request to **httpbin.org/post**. The code then presents the response from httpbin back to the user. 

This is essentially a **scaffold** that can be extended to other use cases. I personally have done this into key value databases in cloudflare, into servicenow and a bunch of other downstream systems. 

```Python
import json
#from botocore.vendored import requests
import urllib3
import logging
import requests

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    raw_event = event['body']
    uri_path = event['path'].strip("/")
    logger.info(event)
    json_event = json.loads(raw_event)
    logger.info(json_event)


    # In the event that you want to POST the data to either an authenticated or unauthenticated API
    # You would invoke something like below

    # Let's do a POST to an external API endpoint with data
    # defining the api-endpoint
    API_ENDPOINT = "https://httpbin.org/post"

    # your API key here
    #API_KEY = "XXXXXXXXXXXXXXXXX"

    # sending post request and saving response as response object
    # inside lambda we need to encode the data object as json before sending
    data = json.dumps(json_event).encode('utf8')


    headers = {
      'Content-Type':'application/json',
      'Accept':'application/json'
    }
    response = requests.post(
      API_ENDPOINT,
      json = data,
      headers = headers
    )

    # extracting response text
    logger.info('data : %s', data)
    logger.info('Response from external API resp : %s', response.text)
    logger.info('Response from external API resp : %s', response.status_code)

    return {
        'statusCode' : '200',
        'body': response.text
    }
```

### Running the code
When I run the code, I get the following output, this creates all of my resources and makes an API gateway for me. The API gateway points to my lambda function, which in turn sends all requests to HTTPBIN.


```Shell
     Type                           Name                                        Status            Info
 +   pulumi:pulumi:Stack            aws-serverless-dev                          created (43s)
 +   ├─ aws:iam:Role                svk-test-webhook-lambda-role                created (4s)
 +   ├─ aws:apigateway:RestApi      svk-test-webhook-api-gw                     created (3s)
 +   ├─ aws:apigateway:Resource     svk-test-webhook-proxy-resource             created (1s)
 +   ├─ aws:apigateway:Method       svk-test-webhook-proxy-root                 created (1s)
 +   ├─ aws:iam:PolicyAttachment    svk-test-webhook-execution-attachment       created (1s)
 +   ├─ aws:lambda:Function         svk-test-webhook-webhook-lambda             created (12s)
 +   ├─ aws:apigateway:Method       svk-test-webhook-method                     created (2s)
 +   ├─ aws:apigateway:Integration  svk-test-webhook-integration                created (1s)
 +   ├─ aws:lambda:Permission       svk-test-webhook-webhook-lambda-permission  created (2s)
 +   ├─ aws:apigateway:Integration  svk-test-webhook-integration-root           created (2s)
 +   ├─ aws:apigateway:Deployment   svk-test-webhook-deployment                 created (1s)      1 warning
 +   └─ aws:apigateway:Stage        svk-test-webhook-stage                      created (1s)
```

I can test this in the following way:

```Shell
 curl -X POST https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/test -d '{"foo":"bar"}'
{
  "args": {},
  "data": "\"{\\\"foo\\\": \\\"bar\\\"}\"",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "application/json",
    "Accept-Encoding": "gzip, deflate",
    "Content-Length": "20",
    "Content-Type": "application/json",
    "Host": "httpbin.org",
    "User-Agent": "python-requests/2.32.3",
    "X-Amzn-Trace-Id": "Root=1-XXXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  },
  "json": "{\"foo\": \"bar\"}",
  "origin": "54.89.157.98",
  "url": "https://httpbin.org/post"
}
```

Things to note are that I need to append the stage to my web request. This allows me to hit my configured API endpoint. 

## Conclusion

All in all this is a neat way to deploy an API gateway to AWS, and makes my life easier the next time that I want to write an integration for another downstream provider. It's a nice and easy way to get up and running without too much effort and means that you can make your entire project about the integration itself and not the infrastructure.
