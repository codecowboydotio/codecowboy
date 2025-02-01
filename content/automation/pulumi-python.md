+++
title = "Pulumi - creating cloud infrastructure using python"
date = "2022-12-19"
aliases = ["pulumi_python"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
I've been fooling around with **pulumi** for a bit now, and thought I would write about it here. 

## What is it?
Pulumi is an infrastructure as code framework with a twist. You can use any language you like in order to write your code (within reason). 

The pulumi tagline is **Your cloud. Your language. Your way.**

I'm here to say it's true!

I've previously written about using pulumi and yaml. This particulat post is about pulumi and python.

I'm going to create the exact same cloud infrastructure as the last post, but this time, I'm using python instead of yaml.

## Installation
Installation is simple, I am using AWS and will follow the docs at [https://www.pulumi.com/docs/get-started/aws/begin/](https://www.pulumi.com/docs/get-started/aws/begin/).

### Install pulumi
In order to install pulumi, just as the docs say, use the following command:

```Bash
curl -fsSL https://get.pulumi.com | sh
```

You will also need to install your language runtime - for me this is python.
If you're using an up to date version of ubuntu then you should have a relatively new version of python (either 3.7 or 3.8) and everything should work fine.

Next configure your AWS account in the normal way that you would

```Bash
export AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID> 
export AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>
```

### Create a new project

In order to get up and going once pulumi is installed you will need to install a new project. 

To do this you use the commands:

```Bash
mkdir quickstart 
cd quickstart
pulumi new aws-python
```

By default this will create a project and a python virtual environment in which pulumi will run in.


If you do not use any command line switches, then by default you will be asked for information about the stack that you are creating. 
```Bash
[root@fedora aaa]# pulumi new aws-python
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

project name: (aaa) blog
project description: (A minimal AWS Python Pulumi program) blog
Created project 'blog'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
stack name: (dev)
Created stack 'dev'

aws:region: The AWS region to deploy into: (us-east-1) ap-southeast-2
Saved config
```

You will also note that the installation process installs a full python virtual environment.

```Bash
Installing dependencies...

Creating virtual environment...
Finished creating virtual environment
Updating pip, setuptools, and wheel in virtual environment...
Requirement already satisfied: pip in ./venv/lib/python3.10/site-packages (21.2.3)
Collecting pip
  Downloading pip-22.3-py3-none-any.whl (2.1 MB)
     |████████████████████████████████| 2.1 MB 3.3 MB/s
Requirement already satisfied: setuptools in ./venv/lib/python3.10/site-packages (57.4.0)
Collecting setuptools
  Downloading setuptools-65.5.0-py3-none-any.whl (1.2 MB)
     |████████████████████████████████| 1.2 MB 8.9 MB/s
Collecting wheel
  Using cached wheel-0.37.1-py2.py3-none-any.whl (35 kB)
Installing collected packages: wheel, setuptools, pip
  Attempting uninstall: setuptools
    Found existing installation: setuptools 57.4.0
    Uninstalling setuptools-57.4.0:
      Successfully uninstalled setuptools-57.4.0

<OUTPUT TRUNCATED>

Finished installing dependencies

Your new project is ready to go!

To perform an initial deployment, run `pulumi up`
```

If you check in the pulumi web console, the stack is created automatically as well.

![Pulumi stack](/images/pulumi-python-blog.jpg)


### Files

Within the directory, you will see a number of files. There are additional files as compared to the YAML example in the previous post. The yaml files are part of every pulumi project. The other files are specific to the language being used - in this case, python.


- Pulumi.yaml defines the project itself.

```Bash
name: blog
description: blog
runtime:
    name: python
    options:
        virtualenv: venv
```

- Pulumi.dev.yaml defines the environment and any environment specifics.

```Bash
config:
  aws:region: ap-southeast-2
```

- requirements.txt is a standard python requirements file.

```Bash
pulumi>=3.0.0,<4.0.0
pulumi-aws>=5.0.0,<6.0.0
```

- __main__.py is the python file where all of your code goes. By default the AWS template for pulumi creates a boilerplate example that has an example of creating an S3 bucket.

```Python
"""An AWS Python Pulumi program"""

import pulumi
from pulumi_aws import s3

# Create an AWS resource (S3 Bucket)
bucket = s3.Bucket('my-bucket')

# Export the name of the bucket
pulumi.export('bucket_name', bucket.id)
```

# The code

Now let's get coding. The following sections represent what we are going to build. 
We are going to build the exact same infrastructure that we build the last time.
The diagram below represents the infrastructure that we will build.

![Infra stack to create using YAML](/images/pulumi.jpg)


### Variables
I set some variables to begin with. These are usually names for various objects as part of the codebase. Each name that I set. All of the variables are really just to name things nicely and consistently. It also means that in order to change my project, I just need to change the variables.

```Python
var_size = 't2.micro'
var_project_name = "pulumi-python"
var_vpc_name = "vpc"
var_key_name = 'my_keypair'
var_vpc_cidr_block = '10.100.0.0/16'
var_subnet_cidr_block = '10.100.1.0/24'
``` 

Each of the variables relates to objects we are going to build next. 


### AWS VPC
To create an AWS VPC we have the following code:

```Python
vpc = aws.ec2.Vpc(
        var_project_name + "-" + var_vpc_name,
        cidr_block=var_vpc_cidr_block,
        tags={
          "Name": var_project_name + "-" + var_vpc_name,
        }
)
```

This piece of code creates a VPC. It also uses the variables that we defined above. I have a habit of using the project name in all of the resources and then tagging the resources as well. 

These resources will be tagged as **pulumi-python-vpc**.


### AWS Subnet

The next step is to create an AWS subnet. This allows me to further carve up the VPC supernet.

The code for the next portion is as follows:

```Python
main_subnet = aws.ec2.Subnet(var_vpc_name + "-subnet",
    vpc_id=vpc.id,
    cidr_block=var_subnet_cidr_block,
    map_public_ip_on_launch=True,
    tags={
        "Name": var_project_name + "-subnet"
    }
)
```


I am using the CIDR block that I defined at the begining of my code. 
The **vpcId** is important as well. I am using the vpc object name from my previous piece of code, and the calulated id. This allows pulumi to calculate the **vpc ID** that was created in the previous step.

As we go through the code, each resource will use identifiers from the resources that were created before it.



### Internet Gateway

The next step is to create an internet gateway for my new VPC. This will allow internet connectivity in and out of my VPC.

```Python
ain_igw = aws.ec2.InternetGateway(var_project_name + "-" + var_vpc_name + "-igw",
    vpc_id=vpc.id,
    tags={
        "Name": var_project_name + "-igw"
    }
)
```

Again I use the **vpc ID** and tag the asset with a tag that includes my project name that I defined above.

In a lot of ways this is simpler than the constructs used when we use YAML as a DSL. I think this is because we are using API's and interfaces that are somewhat native to python.


### Route table and Association

The next steps are to create both a route table and associate that route table with the internet gateway created above, as well as a subnet. 

```Python
main_route_table = aws.ec2.RouteTable(var_project_name + "-" + var_vpc_name + "-rt",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=main_igw.id,
        ),
    ],
    tags={
        "Name": var_project_name + "-rt"
    }
)

main_rt_assoc = aws.ec2.RouteTableAssociation(var_project_name + "-" + var_vpc_name + "-rt",
    subnet_id = main_subnet.id,
    route_table_id = main_route_table.id
)
```

Again, the object identifier of **vpc ID** in object notation is used. I also use the internet gateway ID as well. For the route table association, I use the subnet identifier created above and the route table id.

When I run this, the output is as follows:

```Shell
     Type                              Name               Status
     pulumi:pulumi:Stack               aws-python-dev
 +   ├─ aws:ec2:RouteTable             route_table        created
 +   └─ aws:ec2:RouteTableAssociation  route_table_assoc  created
```

### Security Group
The next step is to create a security group and associate that security group with a VPC.

```Python
group = aws.ec2.SecurityGroup(var_project_name + "-sg",
    description = var_project_name,
    vpc_id = vpc.id,
        ingress = [aws.ec2.SecurityGroupIngressArgs(
            protocol='-1',
            from_port=0,
            to_port=0,
            cidr_blocks=['0.0.0.0/0'],
        )],
        egress = [aws.ec2.SecurityGroupEgressArgs(
            protocol=-1,
            from_port=0,
            to_port=0,
            cidr_blocks=['0.0.0.0/0'],
        )],
        tags={
        "Name": var_project_name + "-sg"
        }
)
```

The output from this step when I run it is:

```Shell
     Type                      Name            Status
     pulumi:pulumi:Stack       aws-python-dev
 +   └─ aws:ec2:SecurityGroup  security_group  created
```

### Instance 

The final step here is to create an instance within the newly created VPC and security groups.
In order to do that, I need to select an instance type to spin up in AWS. I could just pass the ami ID, but that's not particularly portable. Each region has different ami ID's for the same instance. A much more portable way of doing this is to use a data source. I use the **aws.ec2.get_ami** data source. This allows me to search for an ami, and filter it based on its attributes. This data source returns the ami ID that I can use. Using this method the ami ID is potable and the code can be used across regions.

```Python
ami = aws.ec2.get_ami(most_recent=True,
                      owners=["099720109477"],
                      filters=[
                        aws.GetAmiFilterArgs(
                          name="name",
                          values=["ubuntu/images/hvm-ssd/ubuntu-focal-20.04*"]
                        ),
                        aws.ec2.GetAmiFilterArgs(
                          name="virtualization-type",
                          values=["hvm"]
                        ),
                        aws.ec2.GetAmiFilterArgs(
                          name="architecture",
                          values=["x86_64"]
                        ),
                      ],
)
```

Next I create an instance and pass user data to it.

{{< notice tip >}}
When passing inline userdata like this in pulumi, I found that the shebang and shell needed to be on the same line as the three quotes enclosing the user data.
{{< /notice >}}


```Python
user_data="""#!/bin/bash
echo "foo" > /tmp/foo
"""
```

Then I instantiate an instance. This is fairly standard in terms of the pulumi lifecycle and how to make an instance work. I pass in all of the variables that I have defined up above, and I also pass in my user data. 


```Python
server = aws.ec2.Instance('test-server',
    instance_type=var_size,
    vpc_security_group_ids=[group.id],
    user_data=user_data,
    key_name=var_key_name,
    subnet_id=main_subnet.id,
    ami=ami.id,
    tags={
        "Name": var_project_name + "-instance"
    },
)
```


When I run this, I get the following output:

```Shell
     Type                 Name          Status
     pulumi:pulumi:Stack  aws-python-dev
 +   └─ aws:ec2:Instance  web           created
```


## End to end
End to end the entire python file looks like this:

```Python
"""An AWS Python Pulumi program"""

import pulumi
import pulumi_aws as aws

var_size = 't2.micro'
var_project_name = "pulumi-python"
var_vpc_name = "vpc"
var_key_name = 'svk_keypair'
var_vpc_cidr_block = '10.100.0.0/16'
var_subnet_cidr_block = '10.100.1.0/24'

vpc = aws.ec2.Vpc(
        var_project_name + "-" + var_vpc_name,
        cidr_block=var_vpc_cidr_block,
        tags={
          "Name": var_project_name + "-" + var_vpc_name,
        }
)

main_subnet = aws.ec2.Subnet(var_vpc_name + "-subnet",
    vpc_id=vpc.id,
    cidr_block=var_subnet_cidr_block,
    map_public_ip_on_launch=True,
    tags={
        "Name": var_project_name + "-subnet"
    }
)

main_igw = aws.ec2.InternetGateway(var_project_name + "-" + var_vpc_name + "-igw",
    vpc_id=vpc.id,
    tags={
        "Name": var_project_name + "-igw"
    }
)

main_route_table = aws.ec2.RouteTable(var_project_name + "-" + var_vpc_name + "-rt",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=main_igw.id,
        ),
    ],
    tags={
        "Name": var_project_name + "-rt"
    }
)

main_rt_assoc = aws.ec2.RouteTableAssociation(var_project_name + "-" + var_vpc_name + "-rt",
    subnet_id = main_subnet.id,
    route_table_id = main_route_table.id
)


ami = aws.ec2.get_ami(most_recent=True,
                      owners=["099720109477"],
                      filters=[
                        aws.GetAmiFilterArgs(
                          name="name",
                          values=["ubuntu/images/hvm-ssd/ubuntu-focal-20.04*"]
                        ),
                        aws.ec2.GetAmiFilterArgs(
                          name="virtualization-type",
                          values=["hvm"]
                        ),
                        aws.ec2.GetAmiFilterArgs(
                          name="architecture",
                          values=["x86_64"]
                        ),
                      ],
)

group = aws.ec2.SecurityGroup(var_project_name + "-sg",
    description = var_project_name,
    vpc_id = vpc.id,
        ingress = [aws.ec2.SecurityGroupIngressArgs(
            protocol='-1',
            from_port=0,
            to_port=0,
            cidr_blocks=['0.0.0.0/0'],
        )],
        egress = [aws.ec2.SecurityGroupEgressArgs(
            protocol=-1,
            from_port=0,
            to_port=0,
            cidr_blocks=['0.0.0.0/0'],
        )],
        tags={
        "Name": var_project_name + "-sg"
        }
)

user_data="""#!/bin/bash
echo "foo" > /tmp/foo
"""

server = aws.ec2.Instance('test-server',
    instance_type=var_size,
    vpc_security_group_ids=[group.id],
    user_data=user_data,
    key_name=var_key_name,
    subnet_id=main_subnet.id,
    ami=ami.id,
    tags={
        "Name": var_project_name + "-instance"
    },
)

pulumi.export('public_ip', server.public_ip)
pulumi.export('public_dns', server.public_dns)
```

When I run it from the beginning, I get the following output:

```Yaml
     Type                              Name               Status
 +   pulumi:pulumi:Stack               aws-python-dev     created
 +   ├─ aws:ec2:Vpc                    main_vpc           created
 +   ├─ aws:ec2:Instance               web                created
 +   ├─ aws:ec2:SecurityGroup          security_group     created
 +   ├─ aws:ec2:Subnet                 main_subnet        created
 +   ├─ aws:ec2:InternetGateway        internet_gateway   created
 +   ├─ aws:ec2:RouteTable             route_table        created
 +   └─ aws:ec2:RouteTableAssociation  route_table_assoc  created

Resources:
    + 8 created

Duration: 39s
```


### Cleanup

Cleanup is just as easy. 

```Shell
pulumi delete -y
```

This deletes all of the resources that have been created.

```Shell
     Type                              Name               Status
 -   pulumi:pulumi:Stack               aws-python-dev     deleted
 -   ├─ aws:ec2:RouteTableAssociation  route_table_assoc  deleted
 -   ├─ aws:ec2:RouteTable             route_table        deleted
 -   ├─ aws:ec2:Subnet                 main_subnet        deleted
 -   ├─ aws:ec2:SecurityGroup          security_group     deleted
 -   ├─ aws:ec2:InternetGateway        internet_gateway   deleted
 -   ├─ aws:ec2:Instance               web                deleted
 -   └─ aws:ec2:Vpc                    main_vpc           deleted

Resources:
    - 8 deleted

Duration: 1m2s
```

## Conclusion
Pulumi rocks!

I don't know how else to say it.

I can use a multitude of languages to create my cloud resources - languages that I'm familiar with and like. 

This example is python. So far I've done an example in YAML, and now one in python. They create the exact same things. This means that no matter which programming language I'm familiar with, I can create a stack and have that stack work the way I want it to. There is no need for me to learn a complete new DSL just to do my infrastructure as code.

Look out for my next one where I will create the exact same stack in javascript.


