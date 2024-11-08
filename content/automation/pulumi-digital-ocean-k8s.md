+++
title = "Pulumi - creating a kubernetes cluster in digital ocean"
date = "2024-11-08"
aliases = ["pulumi_k8s_digital_ocean"]
tags = ["pulumi", "iac"]
categories = ["automation", "software", "dev", "k8s"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
As many of you know, I've been using Pulumi for a while and I am a big fan.
Today, I'm going to walk through creating Kubernetes in digital ocean.

## Why digital ocean?
A lot of people will wonder **"why digital ocean"??** the answer is simple.
It's extremely well integrated, and this makes it very compelling to use. 

I'm going to walk through creating a simple kubernetes cluster (with three nodes), and then deploying a simple application and load balancer to the cluster.

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
pulumi new digitalocean-python --name do-k8s --description "Digital Ocean k8s example"
```

You should see some output that looks like this:

```Shell
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

Created project 'do-k8s'

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

```Shell
KEY               VALUE
cfg:cluster-name  do-pulumi-k8s
cfg:node-count    3
cfg:node-name     node
cfg:node-size     s-1vcpu-2gb
cfg:region        syd1
pulumi:tags       {"pulumi:template":"digitalocean-python"}
```

### Setting configuration values
In order to set my configuration values I do the following:

```Shell
pulumi config set cfg:cluster-name do-pulumi-k8s
```

This sets the configuration value of **cluster-name** to **do-pulumi-k8s** within the **cfg** namespace.
I can then reference this in my codebase.

## Pulumi program
In order to get going, I need to import some libraries and pull in my stack level configuration.

```Shell
"""A DigitalOcean Python Pulumi program"""

import pulumi
import pulumi_digitalocean as do


## Import configuration variables
stack_config = pulumi.Config("cfg")
var_cluster_name = stack_config.require("cluster-name")
var_node_name = stack_config.require("node-name")
var_node_size = stack_config.require("node-size")
var_node_count = int(stack_config.require("node-count"))
var_region = stack_config.require("region")
```

In the code above, I import the pulumi modules, but also import my stack configuration. These are the configuration values that I set using the **pulumi config set** command earlier. 

These are variables that I may want to change over time, and are maintained outside my codebase. 
These variables are imported and referenced within my pulumi codebase. 

## Create the cluster
In order to create the cluster, I need to supply a number of configuration parameters to the **do.KubernetesCluster** interface. 
These are: 
- name = The k8s cluster name (this will be reflected in the web UI)
- region = The region where you would like your cluster to live - see here: [https://docs.digitalocean.com/platform/regional-availability/](https://docs.digitalocean.com/platform/regional-availability/)
- version = The version of kubernetes. I choose "latest" but you can also use a defined version (see below)
- node_pool = This is the node pool using the KubernetesClusterNodePoolArgs interface
    name = The name of the node pool
    size = The size of the instances within the node pool. These are droplet sizes, remember in digital ocean terms, a worker node is a droplet or VM - see here: [https://slugs.do-api.dev/](https://slugs.do-api.dev/) 
    node_count = The number of nodes in the node pool. I'm going to start with three (this is the default maximum in digital ocean without raising a support ticket).

### Kubernetes version

To get the list of supported kubernetes versions, you can use the following command **doctl**. This is the ditigal ocean control command:

```Shell
 doctl kubernetes options versions
```

The slug version can be used in place of **latest** if needed.

```Shell
Slug           Kubernetes Version    Supported Features
1.31.1-do.3    1.31.1                cluster-autoscaler, docr-integration, ha-control-plane, token-authentication
1.30.5-do.3    1.30.5                cluster-autoscaler, docr-integration, ha-control-plane, token-authentication
1.29.9-do.3    1.29.9                cluster-autoscaler, docr-integration, ha-control-plane, token-authentication
```

## Cluster Code
The following code represents the code to create the cluster.

The code creates a cluster, and a node pool, with three nodes using the latest supported version of Kubernetes within digital ocean.

That's it!

```Shell
cluster = do.KubernetesCluster("do-cluster",
    name = var_cluster_name,
    region = var_region,
    version = "latest",
    node_pool = do.KubernetesClusterNodePoolArgs(
        name = var_cluster_name + "-" + var_node_name,
        size = var_node_size,
        node_count = var_node_count,
    ),
);
```

## Outputs
In order to use the cluster, we will need to export the kubeconfig.
Below we export two things, one is the cluster configuration, the other is the kube_config variable. This is the kubeconfig that we can write to a file and use as a variable to run **kubectl**.

```Shell
pulumi.export('cluster_info', cluster)
pulumi.export('kubeconfig', cluster.kube_configs)
```

This prints out a bunch of configuration items when the program runs and creates my infrastructure. 
You will note that the kubeconfig parameter is showing as a secret.

```Shell
kubeconfig: [secret]
```
We can unwrap this with a simple shell command.

```Shell
pulumi stack output kubeconfig --show-secrets | jq -r .[].raw_config > kubeconfig.json
```

This prints out the kubeconfig for my cluster as a json blob and pipes it to a file called **kubeconfig.json**.

I can then export my **KUBECONFIG** environment variable and use **kubectl** to query my cluster.

```Shell
export KUBECONFIG=/my_pulumui_directory/kubeconfig.json
```

Let's take a look at my cluster:

```Shell
kubectl get nodes

NAME                       STATUS   ROLES    AGE   VERSION
do-pulumi-k8s-node-gcqmf   Ready    <none>   25m   v1.31.1
do-pulumi-k8s-node-gcqmx   Ready    <none>   25m   v1.31.1
do-pulumi-k8s-node-gcqmy   Ready    <none>   25m   v1.31.1
```

As we can see my entire cluster is up and running inside digital ocean.

## Validation within the UI
I can also validate this within the UI

Within the UI, I can click on the **kubernetes** menu item, and I will see my cluster. 
The cluster has the name that I set in my configuration.

![K8s cluster](/images/do-k8s-cluster.jpg)

If I click on the cluster name, and select the **resources** tab, I will see the nodepool and the nodes within my cluster.
These correspond to the **kubectl** output above.

![K8s node pool](/images/do-k8s-nodepool.jpg)

## Deploy an application
This is where thngs get really fun, and it's possibly the thing I like the most about the digital ocean kubernetes experience so far.

I can use the following manifest to create an NGINX container.

```Yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-example
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
      - containerPort: 80
```

In order to apply this I use the following command:

```Shell
kubectl apply -f nginx.yaml
```

I can see that the pod is created by using the **kubectl describe** command. 
This shows me that the pod is created and has an internal IP address. 
This pod isn't yet exposed to the world yet. I need some sort of ingress in order for that to occur.

```Shell
kubectl describe pod nginx-pod

Name:             nginx-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             do-pulumi-k8s-node-gcyjf/10.126.0.3
Start Time:       Mon, 04 Nov 2024 13:28:09 +1100
Labels:           app=nginx-example
Annotations:      <none>
Status:           Running
IP:               10.244.0.141
```

## Ingress
This is the super super cool part. 
The Ingress in digital ocean **just works**. Unlike other providers, there is no need to mess around with creating roles, or creating ingress classes and so on. 

I can simply apply the following manifest:

```Yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-size-unit: "3"
    service.beta.kubernetes.io/do-loadbalancer-disable-lets-encrypt-dns-records: "false"
spec:
  type: LoadBalancer
  selector:
    app: nginx-example
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```
This gives me a load balancer and ingress that points to my cluster and workload.

```Shell
kubectl get svc

NAME         TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.245.0.1       <none>           443/TCP        75m
nginx        LoadBalancer   10.245.224.107   170.64.244.121   80:32104/TCP   41m
```

{{< notice info >}}
This is the piece that's really amazing
{{< /notice >}}

When I look in the digital ocean UI I see the following.
I first see a load balancer object, when I check the load balancer object, I see the three nodes or droplets.

![K8s lb](/images/do-k8s-lb.jpg)

![K8s lb nodes](/images/do-k8s-lb-nodes.jpg)

The amazing piece here is that I deploy my load balancer object using a standard kubernetes manifest. There is absolutely no need for me as a developer to wire anything togther. No need to deploy roles or IAM objects. No need to configure ingress classes. Everything is just pre-configured and ready to go. 

**This is definitely something that digital ocean have gotten right.**



# Conclusion
Pulumi works well with digital ocean, and has a great level of configurability while maintaining simplicity of configuration.

The ease of getting a cluster up and running makes digital ocean very attractive. 

Digital ocean have gotten the balance right between ease of use and simplicity. Everything just works out of the box. This is a huge plus in terms of getting up and running, and I highly recommend giving it a go.
