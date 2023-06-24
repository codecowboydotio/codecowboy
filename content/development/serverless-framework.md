+++
title = "Serverless Framework"
date = "2023-06-24"
aliases = ["serverless framework"]
tags = ["serverless"]
categories = ["software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro

I have been exploring serverless platforms in various guises for some time now. This post is about the serverless framework. 

I recently came across it as part of my day job and it's good enough and simple enough that I thought I'd write about it. 

# What is it

[Serverless framework](https://www.serverless.com/)

The serverless framework is a way to deploy serverless functions on AWS. This means a couple of things:

- Serverless only

- AWS only

This is actually a neat choice if you're only looking to deploy onto AWS and use lambda. I know from my own experience, this is where I started with serverless platforms, and my anecdotal experience is that a lot of people also start with lambda (though this is changing).

## What Does it do?



# Install

In order to install the serverless framework you just need to use npm.

```Bash
npm install -g serverless
```

Assuming that you have your AWS credentials already set up within your environment (like I do) you can start to use the serverless framework right away.

# Up and going

```Bash
serverless
```

By default you will be asked which type of framework you want to use for your project

```Bash
[root@fedora serverless-framework]# serverless

Creating a new serverless project

? What do you want to make? (Use arrow keys)
❯ AWS - Node.js - Starter
  AWS - Node.js - HTTP API
  AWS - Node.js - Scheduled Task
  AWS - Node.js - SQS Worker
  AWS - Node.js - Express API
  AWS - Node.js - Express API with DynamoDB
  AWS - Python - Starter
  AWS - Python - HTTP API
  AWS - Python - Scheduled Task
  AWS - Python - SQS Worker
  AWS - Python - Flask API
  AWS - Python - Flask API with DynamoDB
  Other
```

Don't be fooled, there are a lot of templates that are available in a large number of languages, and across a number of services. 

By default all you get presented are nodejs and python - **there are many more available**.

In order to see more templates, you can use the command

```Bash
serverless create --help
```
## Select a framework

Once you select a framework, and give it a name, the corresponding template is downloaded to your filesystem. The name you choose is used as a directory where your template is installed.

I have chosen the default python HTTP api as an example.

```Bash
[root@fedora serverless-framework]# serverless

Creating a new serverless project

? What do you want to make? AWS - Python - HTTP API
? What do you want to call this project? aws-python-http-api-project

✔ Project successfully created in aws-python-http-api-project folder

? What org do you want to add this service to? codecowboydotio
? What application do you want to add this to? aws-python-http-api-project

✔ Your project is ready to be deployed to Serverless Dashboard (org: "codecowboydotio", app: "aws-python-http-api-project")

? Do you want to deploy now? Yes

Deploying aws-python-http-api-project to stage dev (us-east-1)

✔ Service deployed to stack aws-python-http-api-project-dev (132s)

dashboard: https://app.serverless.com/codecowboydotio/apps/aws-python-http-api-project/aws-python-http-api-project/dev/us-east-1
endpoint: GET - https://ydc8qh1gy6.execute-api.us-east-1.amazonaws.com/
functions:
  hello: aws-python-http-api-project-dev-hello (85 kB)

What next?
Run these commands in the project directory:

serverless deploy    Deploy changes
serverless info      View deployed endpoints and resources
serverless invoke    Invoke deployed functions
serverless --help    Discover more commands
```

With just a few commands I've deployed a default serverless function into AWS.

While it's not perfect, it's super easy to do.

## What's in the directory?

By default, everything is deployed to us-east-1.
This can be changed by editing the configuration within your directory.

Let's take a look at what was created inside the directory.

```Bash
[root@fedora serverless-framework]# cd aws-python-http-api-project/
[root@fedora aws-python-http-api-project]# ll
total 12
-rw-r--r--. 1 root root  246 Jun 24 11:14 handler.py
-rw-r--r--. 1 root root 3836 Jun 24 11:14 README.md
-rw-r--r--. 1 root root  274 Jun 24 11:15 serverless.yml
```

We have three files:

- handley.py 

- README.md

- serverless.yml

Each of these represent different configuration items for my serverless function.

The handler.yml is the python that represents my lambda function. I can modify this to say something meaningful and perform actions that I need it to.

The serverless.yml represents the configuration of my serverless framework options. This is where I can override things like the default deployment location.

Inside the serverless.yml we have the following:

```yaml
org: codecowboydotio
app: aws-python-http-api-project
service: aws-python-http-api-project
frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.9
  region: ap-southeast-2

functions:
  hello:
    handler: handler.hello
    events:
      - httpApi:
          path: /
          method: get
```

you will note that I have added a region parameter to choose the region that I want to deploy to. I can then perform deployment using the **serverless deploy** command.

```Bash
[root@fedora aws-python-http-api-project]# serverless deploy

Deploying aws-python-http-api-project to stage dev (ap-southeast-2)

✔ Service deployed to stack aws-python-http-api-project-dev (120s)

dashboard: https://app.serverless.com/codecowboydotio/apps/aws-python-http-api-project/aws-python-http-api-project/dev/ap-southeast-2
endpoint: GET - https://n8rjd60guf.execute-api.ap-southeast-2.amazonaws.com/
functions:
  hello: aws-python-http-api-project-dev-hello (85 kB)
```

Performing a deployment creates a changeset within my CFT and automatically deploys it for me.

## Let's look at what was deployed

If we interrogate AWS to see what was actually deployed, we can see the following snippet

```Bash
[root@fedora aws-python-http-api-project]# aws lambda list-functions
{
    "Functions": [
        {
            "FunctionName": "aws-python-http-api-project-dev-hello",
            "FunctionArn": "arn:aws:lambda:ap-southeast-2:XXX:function:aws-python-http-api-project-dev-hello",
            "Runtime": "python3.9",
            "Role": "arn:aws:iam::XXX:role/aws-python-http-api-project-dev-ap-southeast-2-lambdaRole",
            "Handler": "s_hello.handler",
            "CodeSize": 85230,
            "Description": "",
            "Timeout": 6,
            "MemorySize": 1024,
            "LastModified": "2023-06-24T01:24:04.056+0000",
            "CodeSha256": "NxFTI7HKbkZ01WwOZY54lOxKpa5DmjR2zuiMoiuWun0=",
            "Version": "$LATEST",
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "9209469d-fe43-46fe-89b4-82402ee03175",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ],
            "EphemeralStorage": {
                "Size": 512
            },
            "SnapStart": {
                "ApplyOn": "None",
                "OptimizationStatus": "Off"
            }
        },
```

If I look at the CFTs that have been deployed, we see the following:

```Bash
[root@fedora aws-python-http-api-project]# aws cloudformation list-stacks | jq '.[] | .[] | .StackName' | grep aws-python
"aws-python-http-api-project-dev"
```

We can see that I have a stack that has been deployed.

If I dig into this a little more, we can see the following.

```Bash
[root@fedora aws-python-http-api-project]# aws cloudformation list-stacks | jq '.[] | .[] | select (.StackStatus != "DELETE_COMPLETE")'
{
  "StackId": "arn:aws:cloudformation:ap-southeast-2:XXX:stack/aws-python-http-api-project-dev/97e72400-122d-11ee-b25b-06bfab59ad12",
  "StackName": "aws-python-http-api-project-dev",
  "TemplateDescription": "The AWS CloudFormation template for this Serverless application",
  "CreationTime": "2023-06-24T01:22:32.787000+00:00",
  "LastUpdatedTime": "2023-06-24T01:23:22.185000+00:00",
  "StackStatus": "UPDATE_COMPLETE",
  "DriftInformation": {
    "StackDriftStatus": "NOT_CHECKED"
  }
}
```

You can see that this stack has been deployed and has had an **UPDATE** to it. This means that we can safely know that our stack has been deployed and is in production.

## How do I check?

There are two ways to check this. The first way is to use the native serverless framework.

```Bash
[root@fedora aws-python-http-api-project]# serverless info
service: aws-python-http-api-project
stage: dev
region: ap-southeast-2
stack: aws-python-http-api-project-dev
dashboard: https://app.serverless.com/codecowboydotio/apps/aws-python-http-api-project/aws-python-http-api-project/dev/ap-southeast-2
endpoint: GET - https://n8rjd60guf.execute-api.ap-southeast-2.amazonaws.com/
functions:
  hello: aws-python-http-api-project-dev-hello
```

To validate this, I will also interrogate the serverless functions using the AWS cli.

```Bash
[root@fedora aws-python-http-api-project]# aws lambda list-functions | jq '.Functions | .[]'
{
  "FunctionName": "aws-python-http-api-project-dev-hello",
  "FunctionArn": "arn:aws:lambda:ap-southeast-2:XXX:function:aws-python-http-api-project-dev-hello",
  "Runtime": "python3.9",
  "Role": "arn:aws:iam::XXX:role/aws-python-http-api-project-dev-ap-southeast-2-lambdaRole",
  "Handler": "s_hello.handler",
  "CodeSize": 85230,
  "Description": "",
  "Timeout": 6,
  "MemorySize": 1024,
  "LastModified": "2023-06-24T01:24:04.056+0000",
  "CodeSha256": "NxFTI7HKbkZ01WwOZY54lOxKpa5DmjR2zuiMoiuWun0=",
  "Version": "$LATEST",
  "TracingConfig": {
    "Mode": "PassThrough"
  },
  "RevisionId": "9209469d-fe43-46fe-89b4-82402ee03175",
  "PackageType": "Zip",
  "Architectures": [
    "x86_64"
  ],
  "EphemeralStorage": {
    "Size": 512
  },
  "SnapStart": {
    "ApplyOn": "None",
    "OptimizationStatus": "Off"
  }
}
```

It's there and it is deployed with a real endpoint.


## Let's test it

If we take the endpoint url from the **serverless info** command, we can see the following:

```Bash
[root@fedora aws-python-http-api-project]# curl  https://n8rjd60guf.execute-api.ap-southeast-2.amazonaws.com/ | jq

{
  "message": "Go Serverless v3.0! Your function executed successfully!",
  "input": {
    "version": "2.0",
    "routeKey": "GET /",
    "rawPath": "/",
    "rawQueryString": "",
    "headers": {
      "accept": "*/*",
      "content-length": "0",
      "host": "n8rjd60guf.execute-api.ap-southeast-2.amazonaws.com",
      "user-agent": "curl/7.79.1",
      "x-amzn-trace-id": "Root=1-64964bc1-4213e6ce0c854cc547427a20",
      "x-forwarded-for": "x.x.x.x",
      "x-forwarded-port": "443",
      "x-forwarded-proto": "https"
    },
    "requestContext": {
      "accountId": "XXX",
      "apiId": "n8rjd60guf",
      "domainName": "n8rjd60guf.execute-api.ap-southeast-2.amazonaws.com",
      "domainPrefix": "n8rjd60guf",
      "http": {
        "method": "GET",
        "path": "/",
        "protocol": "HTTP/1.1",
        "sourceIp": "x.x.x.x",
        "userAgent": "curl/7.79.1"
      },
      "requestId": "HADGRiohywMEJqw=",
      "routeKey": "GET /",
      "stage": "$default",
      "time": "24/Jun/2023:01:49:53 +0000",
      "timeEpoch": 1687571393428
    },
    "isBase64Encoded": false
  }
}
```

## How do I undeploy?

Undeploying is very easy. You simply run the **serverless remove** command.

```Bash
[root@fedora aws-python-http-api-project]# serverless remove
Removing aws-python-http-api-project from stage dev (ap-southeast-2)

✔ Service aws-python-http-api-project has been successfully removed (32s)
```

If I run the aws list command again, you can see that it has been removed.

```Bash
[root@fedora aws-python-http-api-project]# aws lambda list-functions | jq '.Functions | .[]'
```

## How do I view the CFT?

There is another command you can run that will download the generated CFT locally so that you can inspect it.

```Bash
[root@fedora aws-python-http-api-project]# serverless package

Packaging aws-python-http-api-project for stage dev (ap-southeast-2)

✔ Service packaged (9s)
```

This creates a .serverless directory where the generated CFTs are stored.

```Bash
[root@fedora aws-python-http-api-project]# ls -la .serverless
total 120
drwxr-xr-x. 2 root root   172 Jun 24 12:00 .
drwxr-xr-x. 3 root root   100 Jun 24 12:00 ..
-rw-r--r--. 1 root root 85231 Jun 24 12:00 aws-python-http-api-project.zip
-rw-r--r--. 1 root root  2077 Jun 24 12:00 cloudformation-template-create-stack.json
-rw-r--r--. 1 root root 12156 Jun 24 12:00 cloudformation-template-update-stack.json
-rw-r--r--. 1 root root 20034 Jun 24 12:00 serverless-state.json
```

All of the generated artifacts, including the zip file of your code are stored here. 

## Observability?

The serverless framework offers very good observability, but I will focus on that another time :)

# Conclusion

If you are purely interested in deploying serverless functions onto AWS using ubiquitous languages and don't want to concern yourself with infrastructure components and building the infrastructure components, then the serverless framework is a good option.

In three commands, you can create, deploy and remove a serverless function in AWS without needing to know anything about the underlying infrastructure. 

It is flexible enough to allow for the creation of custom templates and has very good observability and github integration. 

It's a good choice if you want to focus on code and not infrastructure.

