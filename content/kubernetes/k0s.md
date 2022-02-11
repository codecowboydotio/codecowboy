+++
title = "k0s - small kubernetes - part 1"
date = "2022-02-10"
aliases = ["k0s"]
tags = ["kubernetes", "containers", "micro"]
categories = ["kubernetes", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

## Intro
I recently sat down and thought to myself "I wonder how the current market of mini or micro kubernetes distributions is going". I've been using minikube for a bit and thought to myself it was about time to start using a few other distributions.

So I have been experimenting with different micro or mini distributions. 

This post is about **k0s**.

## k0s
**k0s** is an open source project that you can find here: [https://k0sproject.io/](https://k0sproject.io/)

**k0s** looks like it's sponsored by Mirantis. They offer commercial support and so on if you want it too. In and of itself, **k0s** is a free and open source project. You can use it and it works without paying anyone :)

## Install
Installation was relatively easy for the most part. It's possible to install both a single node cluster and a multi node cluster. I'll step through both of these.

### Single Node Cluster
A single node cluster is perhaps the easiest to install and the fastest way to get up and running.

```Bash
curl -sSLf https://get.k0s.sh | sudo sh
```
This will install the k0s binary in **/usr/local/bin/**.

```Bash
sudo k0s install controller --single
```
Next install the cluster as a single node cluster.

You can check the status of your freshly installed cluster using the k0s command.

```Bash
[root@kube ~]# k0s status
Version: v1.23.3+k0s.0
Process ID: 1313
Role: controller
Workloads: true
SingleNode: true
```

It's that simple - from this point on, you have a single node cluster that can be used as normal.



### Multi Node Cluster
To install a multi node cluster, it's best to use a second binary called **k0sctl**. This can be downloaded and installed in the following way.

```Bash
curl -L https://github.com/k0sproject/k0sctl/releases/download/v0.13.0-beta.1/k0sctl-linux-x64 -o k0sctl
```

#### A Note on SSH

## Deployment

## Ingress

## Conclusion
I was very pleasantly surprised by **k0s** it was easy to deploy and get up and running in either a single node cluster or a multi node cluster. The documentation was very good, and **it just worked**. This is perhaps the most important part for me. As a developer, I want to have the ability to get up and running super fast, without a lot of fuss to be able to try out different configurations or deploy applications.
