+++
title = "Terraform Scaffold: The lambda code"
date = "2024-06-15"
aliases = ["terraform_scaffold"]
tags = ["terraform", "iac"]
categories = ["automation", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I last wrote about an API gateway and lambda function in AWS. I wanted to expand on that and walk through the lambda function in more detail.

## TLDR - The github repo
Th lambda function can be anything, but in my case it takes a JSON input and passes certain fields to another API endpoint. This is most useful for being able to perform transformations on data.

You can clone the repo with all of the working code here:

[Scaffold github repository](https://github.com/codecowboydotio/scaffolds)

## Why is this important?
Having a generic lambda function that can take input, reformat that input, and then send more data on to another function is the starting point of all integrations. If I want to integrate between platforms, this is the building block that I can use.

## The basic scaffold

The basic scaffold that I share here is agnostic and can be used for any project, be it more complex or less complex.

## A starting point.
My standard scaffold looks like this:

I do some standard imports here. 

I import the json library as I'm accepting JSON input. 
I import the urllib3 library. This is interesting because in AWS at least, when you change the python version to a newer version, the requests library that I normally use is deprecated.
I import the logging library as I like to write out some logs so that I can see the requests in cloudtrail.

```Python
import json
import urllib3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)
```


```
import json
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

```
{
  "args": {},
  "data": "{\"foo\": \"bar\"}",
  "files": {},
  "form": {},
  "headers": {
    "Accept-Encoding": "identity",
    "Content-Length": "14",
    "Content-Type": "application/json",
    "Host": "httpbin.org",
    "User-Agent": "python-urllib3/1.26.18",
    "X-Amzn-Trace-Id": "Root=1-666cd94b-10665b2e1218e03e7f28a9f6"
  },
  "json": {
    "foo": "bar"
  },
  "origin": "13.239.47.129",
  "url": "https://httpbin.org/post"
}
```
