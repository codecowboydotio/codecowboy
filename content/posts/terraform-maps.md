+++
title = "Terraform Maps"
date = "2021-09-26"
aliases = ["terraform_maps"]
tags = ["terraform", "iac"]
categories = ["software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I've been doing a lot with terraform lately, and I've been looking for ways to make my terraform configurations a lot simpler and have less repetition. Like a lot of people, I've found myself repeating the same code over and over. An example is where I repeat the same resource over and over but with different configuration parameters. It's essentially the same resource. Why should I do this? **There has to be a better way.**

## What is a map?
In looking for a way to make my codebase cleaner and simpler, I started looking at using maps. I've not delved into them in depth before, so it was a fun thing to do on a rainy afternoon. 

The terraform docs do a better job than I can of telling you what a map is - see [here](https://www.terraform.io/docs/language/expressions/types.html#maps-objects)

## Why are maps cool?
Maps are cool because they allow you to have groups of key value pairs that can be accessed in a neat way. 

## A simple example
A very simple example of a map is as follows:

```
variable "tcp_lb" {
  type = map
  default = {
    unit-config-origin = "8888"
    unit-git-origin = "8080"
  }
}
```
You will notice that the map above is actually a variable. That's right **you can use a map as a variable**. 

In this case, I am using a map to assign different ports to origin servers within a [volterra](http://volterra.io) resource.
In this way, I don't need to declare the same resource over and over, I can use a loop within my resource to access all of the items within my map.

## The resource
The resource that uses the map above looks like this:

```
resource "volterra_tcp_loadbalancer" "unit-config" {
  for_each  = var.tcp_lb
  name      = "${each.key}"
  namespace = var.ns

  listen_port = "${each.value}"
  dns_volterra_managed = true
  domains = ["${var.domain_host}.${var.domain}"]
  advertise_on_public_default_vip = true

  retract_cluster = true

  origin_pools_weights {
    pool {
      name = "${each.key}"
      namespace = var.ns
    }
  }

  hash_policy_choice_round_robin = true
}
```

There are two pieces to this resource that I need to point out.

- **The for loop**
- **Accessing map keys and values**


## Conclusion
