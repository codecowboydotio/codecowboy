+++
title = "CDK - creating an ECS cluster"
date = "2025-04-27"
aliases = ["cdk"]
tags = ["cdk", "iac"]
categories = ["automation", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
There are multiple container technologies available in AWS, this post focuses on ECS, the Elastic Container Service. This service is interesting because it's not kubernetes based. This means that the service is typically simpler to get up and running, and is easier to work with. It also fits development workflows a lot better than many kubernetes distributions out there.

This post is about how to set up ECS using AWS' own CDK toolkit.

## TLDR
If you just want the code - it's here: 

[CDK ECS repository](https://github.com/codecowboydotio/cdk)


## What am I building?
I am building an ECS cluster with a single task that runs an NGINX container.
The NGINX container is pulled from a public repository - dockerhub.

## Why?
I am noticing that CDK is becoming more and more widely used as part of deploying to AWS. Given it's AWS specific, and so is ECS, I figured that the pairing made a lot of sense.

## The Project 
To get started we need to set up a new project.
There are two ways of doing this, from nothing, or cloning the rpository.

### Cloning the repo
In order to clone the repo you just need to use git and clone away.

```Shell
git clone https://github.com/codecowboydotio/cdk
cd cdk/minimal-ecs
```

This clones the ready to go git repository for you. Everything is there ready to run.

### Creating from scratch
In order to create a CDK project from scratch you need to do the following.

First, install CDK on your environment.

```Shell
npm install -g aws-cdk
```

Next you initialise the project.

```Shell
mkdir minimal-ecs
cd minimal-ecs

cdk init app --language=typescript
```

By default, CDK will set up a git repository so that you can check changes in.
You should see the following output

```Shell
cdk init app --language=typescript
Applying project template app for typescript
# Welcome to your CDK TypeScript project

This is a blank project for CDK development with TypeScript.

The `cdk.json` file tells the CDK Toolkit how to execute your app.

## Useful commands

* `npm run build`   compile typescript to js
* `npm run watch`   watch for changes and compile
* `npm run test`    perform the jest unit tests
* `npx cdk deploy`  deploy this stack to your default AWS account/region
* `npx cdk diff`    compare deployed stack with current state
* `npx cdk synth`   emits the synthesized CloudFormation template

Initializing a new git repository...
hint: Using 'master' as the name for the initial branch. This default branch nam                               e
hint: is subject to change. To configure the initial branch name to use in all
hint: of your new repositories, which will suppress this warning, call:
hint:
hint:   git config --global init.defaultBranch <name>
hint:
hint: Names commonly chosen instead of 'master' are 'main', 'trunk' and
hint: 'development'. The just-created branch can be renamed via this command:
hint:
hint:   git branch -m <name>
Executing npm install...
✅ All done!
```

You may see some other warnings depending on your node setup.

At this point you should have a minimal "hello world" CDK application ready to go.

## The code
The code has two major components, both are typescript.

There are two directories of note here
bin/
lib/

For out purposes today, we are going to focus on both of these directories, and replace the code completely in each one.

First, open the file **bin/minimal-ecs.ts**. 
Replace the contents with the following

```Typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { MinimalEcsStack } from '../lib/minimal-ecs-stack';

const app = new cdk.App();
new MinimalEcsStack(app, 'MinimalEcsStack', {
  /* If you don't specify 'env', this stack will be environment-agnostic.
   * Account/Region-dependent features and context lookups will not work,
   * but a single synthesized template can be deployed anywhere. */

  /* Uncomment the next line to specialize this stack for the AWS Account
   * and Region that are implied by the current CLI configuration. */
  // env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION },

  /* Uncomment the next line if you know exactly what Account and Region you
   * want to deploy the stack to. */
  env: { account: '1234567890', region: 'ap-southeast-2' },

  /* For more information, see https://docs.aws.amazon.com/cdk/latest/guide/environments.html */
});

cdk.Tags.of(app).add('customer-app', 'prod-app', {
  //applyToLaunchedInstances: false,
});
```

### Imports
The imports do two things, they import the entire CDK library, and also import my own library that I've written called MinimalEcsStack.

This is contained in my lib directory.

```Typescript
import * as cdk from 'aws-cdk-lib';
import { MinimalEcsStack } from '../lib/minimal-ecs-stack';
```

The latter exposes interfaces that I can use as part of my cdk application.

The next piece of code sets up my CDK application object, and environment.
You will note that the environment has a default region and account number. You should **change these** with your own values.

```Typescript
const app = new cdk.App();
new MinimalEcsStack(app, 'MinimalEcsStack', {
  env: { account: '1234567890', region: 'ap-southeast-2' },
});

```

### Tags

```Typescript
cdk.Tags.of(app).add('customer-app', 'prod-app', {
  //applyToLaunchedInstances: false,
});
```

## Conclusion
