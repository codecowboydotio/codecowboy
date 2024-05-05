+++
title = "Terraform Scaffold: AWS API Gateway"
date = "2024-05-04"
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

## TLDR - The github repo

You can clone the repo with all of the working code here:

[Scaffold github repository](https://github.com/codecowboydotio/scaffolds)

## Why is this important?
I find this to be important as a pattern because it allows me to rapidly test integrations with various platforms. I have used this scaffold and pattern to test integrations with third parties as diverse as cloudflare, and other custom written applications.

## The architecture
The architecture for this scaffold is an API gateway, that points all routes to a single lambda function. The lambda function can be anything, but in my default scaffold, I have a lambda function that takes JSON imput, and then passes it on to another endpoint. 

This is useful for integration testing.

![AWS API Gateway](/images/aws-api-gateway.jpg)

## The basic scaffold

The basic scaffold that I share here is agnostic and sets up the "base AWS infrastructure" that I tend to use in each project. There are some tactics and techniques I use here that are interesting and have stood the test of time. Many of these have been developed over time.

## A starting point.
My standard scaffold looks like this:

```
├── api.tf
├── aws.tf
├── bucket.tf
├── lambda.tf
├── tags.tf
├── vars.tf
├── lambda_code/webhook.py
```

Each of the files contains various basic AWS components. 

- api.tf - This is the terraform template that creates the API scaffolding - the API gateway and routes.

- aws.tf - As this is a project that uses other services this only contains a variable for the region that I want to deploy in.

- bucket.tf - The bucket is used to hold the lambda code that is being used as part of the scaffold. This uses a random suffix to create a unique name.

- lambda.tf - This creates the lambda function and connects all of the required roles for execution.

- tags.tf - This contains a map of tags that I have to be able to change tags on a per project basis.

- vars.tf - This file contains all of the variables that I use as part of spinning up a new project using my scaffold.

- lambda_code/webhook.py - This is the lambda code that does all of the work. This example is written in python.

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
 resource "aws_api_gateway_rest_api" "scaffold" {
  name = "${var.name-prefix}-${var.project}-api-gw"
```

This resource would be named "test-scaffold-api-gw".

This is a very flexible way of naming and means that if I change the variables, the names of all of my resources change. 

### API Gateway

I create an API gateway, specifically a REST api gateway. 
I create a REST based API gateway in AWS as this is the example I am going to show. While there are different types of API gateways available in AWS, this is probably the most ubiquitous.


```Shell
resource "aws_api_gateway_rest_api" "scaffold" {
  name = "${var.name-prefix}-${var.project}-api-gw"
  description = "scaffold webhook example"

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
My two resources are type proxy, and allow any method without authorization. 
We can add authorization later, but for testing, this is an easy configuration.

```Shell
resource "aws_api_gateway_resource" "proxy" {
   rest_api_id = aws_api_gateway_rest_api.scaffold.id
   parent_id   = aws_api_gateway_rest_api.scaffold.root_resource_id
   path_part   = "{proxy+}"
}
resource "aws_api_gateway_method" "proxy" {
   rest_api_id   = aws_api_gateway_rest_api.scaffold.id
   resource_id   = aws_api_gateway_resource.proxy.id
   http_method   = "ANY"
   authorization = "NONE"
}
```

I then create an integration between my API gateway proxy resources and the function that I create. This allows my API gateway to call my underlying resource (in this case my lambda function).

The configuration here is interesting in that I do not allow any method to the lambda function, I only allow POST. For lambda functions, this is the only method that is available to invoke a lambda function. 

```Shell
resource "aws_api_gateway_integration" "lambda" {
   rest_api_id = aws_api_gateway_rest_api.scaffold.id
   resource_id = aws_api_gateway_method.proxy.resource_id
   http_method = aws_api_gateway_method.proxy.http_method
   integration_http_method = "POST"
   type                    = "AWS_PROXY"
   uri                     = aws_lambda_function.scaffold_webhook.invoke_arn
}
```

Next I create an API deployment
This allows me to capture the configuration. 
I am using the stage_name directive rather than stage. 
This has a side effect of state changes causing an interruption to my API gateway on deployment. This is fine for me during testing, but in production this may not be desirable.

```Shell
resource "aws_api_gateway_deployment" "scaffold" {
   depends_on = [
     aws_api_gateway_integration.lambda,
     aws_api_gateway_integration.lambda_root,
   ]
   rest_api_id = aws_api_gateway_rest_api.scaffold.id
   stage_name  = "test"
}
```

Lastly I grant permissions to execute the lambda.

```Shell
resource "aws_lambda_permission" "apigw" {
   statement_id  = "AllowAPIGatewayInvoke"
   action        = "lambda:InvokeFunction"
   function_name = aws_lambda_function.scaffold_webhook.function_name
   principal     = "apigateway.amazonaws.com"
# The "/*/*" portion grants access from any method on any resource
   # within the API Gateway REST API.
   source_arn = "${aws_api_gateway_rest_api.scaffold.execution_arn}/*/*"
}
```

### The bucket

I create an S3 bucket to hold the lambda function when it is deployed. 

The first thing I do is use the **random_string** resource to create a rnadom string for my bucket name. As S3 is a global namespace, you cannot have two buckets that are the same name globally. This is problematic if you're trying to create re-usable code. I work around this by using the **random_string** resource and appending it to the bucket name.

```Bash
resource "random_string" "bucket_suffix" {
  special = false
  upper   = false
  length  = 5
}

resource "aws_s3_bucket" "lambda_bucket" {
  bucket = "${var.name-prefix}-${var.project}-${random_string.bucket_suffix.id}"
  force_destroy = true

  tags = {
    for k, v in merge({
      app_type = "production"
      Name = "${var.name-prefix}-${var.project}-bucket-${random_string.bucket_su
ffix.id}"
    },
    var.default_ec2_tags): k => v
  }
}
```

The next part of the bucket are just setting ownership and permission controls on the bucket. It is not possible to create public buckets by default any more, so this is required for all buckets. 

```Bash
resource "aws_s3_bucket_ownership_controls" "example" {
  bucket = aws_s3_bucket.lambda_bucket.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }

  depends_on = [ aws_s3_bucket.lambda_bucket ]
}

resource "aws_s3_bucket_acl" "lambda_bucket" {
  bucket = aws_s3_bucket.lambda_bucket.id
  acl    = "private"

  depends_on = [ aws_s3_bucket_ownership_controls.example ]
}
```

### The lambda function
I create an archive file of my python codebase in order to upload it to the bucket and deploy it as a lambda function.

The next step is to actually place the zip file that is generated into the bucket.

```Shell
data "archive_file" "python_lambda" {
  type = "zip"
  source_file = "lambda_code/webhook.py"
  output_path = "foo.zip"
}

resource "aws_s3_object" "python_lambda" {
  bucket = aws_s3_bucket.lambda_bucket.id

  key    = "program.zip"
  source = data.archive_file.python_lambda.output_path

  etag = filemd5(data.archive_file.python_lambda.output_path)
}
```

Once I have the zip file in the bucket, I can create the lambda function, referring to the zip file in the bucket.

It should be noted that this approach has a maximum file size for a lambda function. So if your code is very large, you may need to create your lambda function from a container image.

```Shell
resource "aws_lambda_function" "scaffold_webhook" {
  function_name = "${var.name-prefix}-${var.project}-function"

  s3_bucket = aws_s3_bucket.lambda_bucket.id
  s3_key    = aws_s3_object.python_lambda.key

  runtime = "python3.11"
  handler = "webhook.lambda_handler"

  source_code_hash = data.archive_file.python_lambda.output_base64sha256

  role = aws_iam_role.lambda_exec.arn
}
```

The next piece of the puzzle if a cloudwatch log group. This is very very useful in terms of logging the execution of the lambda function. This is most useful during development when you are looking to debug any problems or behaviour that may occur when you actually deploy your function.

```Bash
resource "aws_cloudwatch_log_group" "scaffold_webhook" {
  name = "/aws/lambda/${aws_lambda_function.scaffold_webhook.function_name}"

  retention_in_days = 30
}
```

Last of all, I create a role that allows my function to execute. 
I assign the lambda execution policy to the role.

```Bash
resource "aws_iam_role" "lambda_exec" {
  name = "${var.name-prefix}-${var.project}-lambda-exec-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Sid    = ""
      Principal = {
        Service = "lambda.amazonaws.com"
      }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_policy" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
"
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
Below I have a bucket resource, this gets created, and has the tags applied. 
Note that the tags are applied as a map AND there are individual tags being applied as well. 


```Shell
resource "aws_s3_bucket" "lambda_bucket" {
  bucket = "${var.name-prefix}-${var.project}-${random_string.bucket_suffix.id}"
  force_destroy = true

  tags = {
    for k, v in merge({
      app_type = "production"
      Name = "${var.name-prefix}-${var.project}-bucket-${random_string.bucket_suffix.id}"
    },
    var.default_ec2_tags): k => v
  }
}
```

The for loop in this case merges both the tags that I have explicitly set, as well as the tags from my map. The map is read in as a key value pair and applied to the resource. 

### The Python Code

The python code isn't that amazing, but it does show an example of how to receive a REST API request, taks the data portion, and send that on to another API. 

This is quite common when dealing with webhooks and performing integration between systems. 

I am performing a **POST** to httpbin - **https://httpbin.org/post**. This is a generic endpoint that will just echo back when I send to it. 

The last piece of the puzzle in my code is I take the response from httpbin and return it to my API gateway.

The code should work as is and I will write another blog post walking through it in more detail. This post is entirely focussed on building out the scaffold using terraform.

```Python
import json
#from botocore.vendored import requests
import urllib3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    raw_event = event['body']
    uri_path = event['path'].strip("/")
    logger.info(event)
    json_event = json.loads(raw_event)
    logger.info(json_event)


    # At this point I can take the path variable uri_path and write the alert to a bucket that is for a
    # specific customer or whatever the case may be.


    # In the event that you want to POST the data to either an authenticated or unauthenticated API
    # You would invoke something like below

    # Let's do a POST to an external API endpoint with data
    # defining the api-endpoint
    API_ENDPOINT = "https://httpbin.org/post"

    # your API key here
    API_KEY = "XXXXXXXXXXXXXXXXX"

    # sending post request and saving response as response object
    # inside lambda we need to encode the data object as json before sending
    http = urllib3.PoolManager()


    #data = json.dumps(json_event).encode('utf-8')
    data = json.dumps(json_event).encode('utf8')
    headers = {
      'Content-Type': 'application/json'
    }
    response = http.request(
      'POST',
      API_ENDPOINT,
      body = data,
      headers = headers
    )

    # extracting response text
    logger.info('Response from external API resp : %s', response)
    logger.info('Response from external API resp : %s', response.status)

    return {
        'statusCode' : '200',
        'body': response.data.decode("utf-8")
    }
```


## Why do things this way?
I am a big believer in standardising things. I am also a big believer in idempotency - **across projects**. 
I want to make sure that I can simply clone the scaffold, do nothing but change the variables for the name, and have a guarantee that I will get my standard set of infrastructure up and running.

## Conclusion
Using a standard template as a scaffold that you can extend is a lot of fun and makes my setups and experimentation easier.

I have used this particular scaffold multiple times across multiple projects and aside from occasionally needing to update various components due to deprections of versions - old versions of pyhton and so on - it has stood the test of time.

I find it vastly speeds up my development and testing if I have a scaffold ready to go - all I need to change is the underlying python code for the specific API that I am integrating to.
