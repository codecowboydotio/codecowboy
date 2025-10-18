+++
title = "Pulumi - creating a website bucket and cloudfront distribution"
date = "2025-03-07"
aliases = ["pulumi_javascript"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
I've been fooling around with **pulumi** for a bit now, and thought I would write about it here. 
One of the things that I've always wanted to do, but never have is understand cloudfront a little more. I decided to do this by building a simple website bucket, and then exposing it via a cloudfront distribution.

## TLDR
If you just want the code - it's here: 

[API Gateway git repository](https://github.com/codecowboydotio/pulumi/tree/main/aws-serverless)

## What
I will build a single bucket that is used to host a website, and upload a simple index.html file to the bucket. The bucket will have appropriate security controls applied to allow public access. 

The second part will involve placing the bucket behind a cloudfront distribution and allowing access, then disallowing access from a country to test the cloudfront distribution itself.

I will be building two architectures today, to demonstrate cloudfront with and without geo restrictions.

![Cloudfront no geo restrictions](all-cloudfront.jpg)

Finally, I will demonstrate cloudfront using geo restrictions.

![Cloudfront with geo restrictions](no-au-architecture.jpg)

## Configuration
The configuration of the template is simple and standard. Import the relevant modules, and set the config imports to get the variables from the stack config.

The **pulumi_random** module is imported to handle the unique namespace requirements for S3 buckets. This is explained further down.

```Python
import pulumi
from pulumi_aws import s3
import pulumi_aws as aws
import pulumi_random as random
import json

config = pulumi.Config()
project_name = config.get("project_nane", "svk")
bucket_name = config.get("bucket_name")
```

## Create an S3 bucket
S3 buckets need a unique name globally as S3 is a shared namespace. This means that each bucket needs to have a unique name. 

[https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html)

Using the **pulumi_random** module, I create a string of six characters, that is all lower case and has no special characters. This is then concatenated together with the project name and the bucket name variables to create a unique name for the bucket before creating it. 

```Python
# Random suffix for bucket
random = random.RandomString("random",
    length=6,
    upper=False,
    special=False
)

suffix = random.id.apply(lambda id: f"{project_name}-{bucket_name}-{id}")
```

The bucket is created using the standard bucket method from the Pulumi modules. The name of the bucket is set to the project name and the bucket name with the random string as a suffix on the end that was created in the previous step.

```Python
# Create an AWS resource (S3 Bucket)

website_bucket = aws.s3.Bucket("websiteBucket",
    bucket=suffix,
    force_destroy=True,
)
```

Once created, the bucket looks like this: 

![Bucket](bucket.jpg)


## Bucket config 
In order to use the bucket as a website, some configuration is required. The following code block configures the bucket to be used as a website. This allows the index and error documents to be defined as well as a routing rule. 

The routing rule means that accessing the URL **docs/** will redirect to **documents/**. Routing rules can be simple or complex, this is an example of a simple routing rule.

```Python
bucket_config = aws.s3.BucketWebsiteConfiguration("bucket_config",
    bucket=website_bucket.id,
    index_document={
        "suffix": "index.html",
    },
    error_document={
        "key": "error.html",
    },
    routing_rules=[{
        "condition": {
            "key_prefix_equals": "docs/",
        },
        "redirect": {
            "replace_key_prefix_with": "documents/",
        },
    }])
```

The next code block removes the public access block on the bucket. When creating an S3 bucket in AWS, the default is to put in place a public access block. As this is a website, we remove the public access block so that people can access the website.

```Python
website_bucket_public_access_block = aws.s3.BucketPublicAccessBlock("access_block",
    bucket=website_bucket.id,
    block_public_acls=False,
    block_public_policy=False,
    ignore_public_acls=False,
    restrict_public_buckets=False
)
```

## Bucket Policy
Even though the bucket is public, there is no way to access or upload content to the bucket. This includes being able to read files. In order to allow both reads and uploads to the bucket, we need to create and attach a policy.

The first line of code uses a python lambda function to get the current bucket ARN. This is required as part of the policy document so that the policy can be applied to the correct bucket.

The first statement in the policy, **Statement1**, allows all actions (including write and delete) to the account where the bucket is being created. This essentially allows upload access and file management.

The second statement in the policy, **Statement2**, allows get and list access to everyone in the world.

Lastly, this policy document is added to the bucket.

```Python
policy_arn = website_bucket.arn.apply(lambda arn: f"{arn}/*")

bucket_policy_document = {
    "Version": "2012-10-17",
    "Statement": [
        {
           "Sid": "Statement1",
           "Effect": "Allow",
           "Principal": {
             "AWS": "YOUR_ACCOUNT",
           },
           "Action": "*",
           "Resource": [
             website_bucket.arn,
             policy_arn,
           ],
        },
        {
            "Sid": "Statement2",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket",
            ],
            "Resource": [
                policy_arn,
                website_bucket.arn,
            ],
        }
    ]
}

# Attach the bucket policy to the S3 bucket
website_bucket_policy = aws.s3.BucketPolicy("website-bucket-policy",
    bucket=website_bucket.id, # Referencing the ID of the bucket created above
    policy=bucket_policy_document # The JSON policy document
)
```

## Upload a file to the bucket
The followinfg code snippet uploads a file named **index.html** that is on the local filesystem to the bucket. In this case, the local HTML file just says the word "foo".

```Python
# Upload a simple index.html file to the S3 bucket
index_html = aws.s3.BucketObject("indexHtml",
    bucket=website_bucket.id,
    key="index.html",
    content_type="text/html",
    source=pulumi.FileAsset("index.html"), # Assuming an index.html file exists in the same directory
)
```

## Cloudfront 
The following configuration is a minimum configuration for cloudfront. Cloudfront is an edge distribution service from AWS. It has a lot of configuration options, that allow for a lot of different use cases. The configuration below is for a basic configuration that uses a cloudfront edge distribution backed by an S3 bucket.

The distribution resource is an edge cloudfront node that allows my content to be distributed. 

Most of the code below is boilerplate default with the exception of both the origins and restrictions sections.

```Python
s3_origin_id=project_name + "-" + bucket_name
distribution_resource = aws.cloudfront.Distribution("distributionResource",
    default_root_object="index.html",
    default_cache_behavior={
        "allowed_methods": [
          "DELETE",
          "GET",
          "HEAD",
          "OPTIONS",
          "PATCH",
          "POST",
          "PUT",
        ],
        "viewer_protocol_policy": "allow-all",
        "cached_methods": [
          "GET",
          "HEAD",
        ],
        "target_origin_id": s3_origin_id,
        "forwarded_values": {
            "cookies": {
                "forward": "none",
            },
            "query_string": False,
        },
    },
    enabled=True,
    origins=[{
        "domain_name": website_bucket.bucket_regional_domain_name,
        "origin_id": s3_origin_id,
    }],
    restrictions={
        "geo_restriction": {
            "restriction_type": "whitelist",
            "locations": [
              "AU",
              "CA",
            ],
        },
    },
    viewer_certificate={
        "cloudfront_default_certificate": True,
    }
)
```

The origins portion of the code is where the bucket is selected as the source for the cloudfront distribution.

![cloudfront distribution](cloudfront-distribution.jpg)

![cloudfront origin](cloudfront-origin.jpg)

The restrictions portion of the code is where I set the geographies that are allowed to access the bucket.

The georgaphies list two - Canada and Australia as part of an allow list. This allows both Canada and Australia to access the cloudfront endpoint.

![cloudfront geographies](cloudfront-geographic-2.jpg)

```Python
    origins=[{
        "domain_name": website_bucket.bucket_regional_domain_name,
        "origin_id": s3_origin_id,
    }],
    restrictions={
        "geo_restriction": {
            "restriction_type": "whitelist",
            "locations": [
              "AU",
              "CA",
            ],
        },
    },
```
## Outputs
Lastly, there are two outputs that list both the **website URL** of the direct bucket, and the **cloudfront distribution** resource. 

This allows me to access both the website and bucket directly, as well as using the cloudfront distribution (and the cloudfront specific access restrictions).

```Python
# Export the name of the bucket
pulumi.export('website URL', bucket_config.website_endpoint)
pulumi.export('dist', distribution_resource.domain_name)
```

## Validation
Running the pulumi script, gives me the following output. All of the resources are successfully created, and I am able to successfully access the website bucket using the website URL from the output.


```Shell
Updating (dev)

     Type                                  Name                   Status
 +   pulumi:pulumi:Stack                   aws-cloudfront-dev     created (243s)
 +   ├─ random:index:RandomString          random                 created (0.30s)
 +   ├─ aws:s3:Bucket                      websiteBucket          created (2s)
 +   ├─ aws:s3:BucketObject                indexHtml              created (1s)
 +   ├─ aws:s3:BucketPublicAccessBlock     access_block           created (1s)
 +   ├─ aws:s3:BucketWebsiteConfiguration  bucket_config          created (1s)
 +   ├─ aws:cloudfront:Distribution        distributionResource   created (234s)
 +   └─ aws:s3:BucketPolicy                website-bucket-policy  created (2s)

Outputs:
    dist       : "duhjvh1wofuou.cloudfront.net"
    website URL: "svk-test-5vlxoo.s3-website-ap-southeast-2.amazonaws.com"

Resources:
    + 8 created

Duration: 4m5s
```

## Accessing the content
Using the direct method of access, I am able to access the **index.html** file in the bucket. 

![Bucket direct access](bucket-direct.jpg)

I can also access the cloudfront distribution via the couldfront distribution URL.

![cloudfront distribution](cloudfront-working.jpg)


If I change the distribution resource to remove the AU origin from the geo restriction allow list, I can no longer access the cloudfront distribution.

First I modify the code and re-apply it. 

```Shell
     Type                            Name                  Status            Info
     pulumi:pulumi:Stack             aws-cloudfront-dev
 ~   └─ aws:cloudfront:Distribution  distributionResource  updated (95s)     [diff: ~restrictions]

Outputs:
    dist       : "duhjvh1wofuou.cloudfront.net"
    website URL: "svk-test-5vlxoo.s3-website-ap-southeast-2.amazonaws.com"

Resources:
    ~ 1 updated
    7 unchanged

Duration: 1m39s
```

Checking the AWS console, I can see that the AU or Australian geography has been removed. 

![Cloudfront geography](cloudfront-geographic-1.jpg)

When I try to access the website, I get an error. The error is based on the geographic endpoint of AU not being part of the distribution any more.

![Cloudfron geo error](cloudfront-403.jpg)

## Summary
Using pulumi to create a website bucket, and expose that website via a cloudfront distribution is relatively simple. There are a lot of moving parts when you do this, and each one needs to be in place for this to work. The bucket needs to have a policy as well as being publicly exposed for example. If you have all of the moving pieces then creating and exposing a bucket via cloudfront is relatively easy.
