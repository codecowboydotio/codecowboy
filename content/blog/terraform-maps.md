+++
title = "Terraform Maps"
date = "2021-09-26"
aliases = ["terraform_maps"]
tags = ["terraform", "iac"]
categories = ["automation", "software", "dev"]
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

```Bash
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

```Bash
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

- **The for_each loop**
- **Accessing map keys and values**

### The for_each loop
The for_each loop is used to loop through the map within the context of the resource. 
The for_each loop in terraofrm is documented [here](https://www.terraform.io/docs/language/meta-arguments/for_each.html#basic-syntax). The interesting thing is that a for_each loop can **accept a map** as an input!

```Bash
  for_each  = var.tcp_lb
```
Note that the for_each loop references the variable that I defined above. The variable is actually a map, which the for_each loop can accept as an input. 
The for_each loop will loop through each key within the variable tcp_lb. In my case, the variable has two values.
The for_each loop will run twice as it iterates over my map variable.

### Accessing map keys and values
The map that I have defined has two keys and two values.
Each key and each value can be acess separately within the context of the **for_each** loop.

In order to access each of the map values or names, I can use the following syntax:

```Bash
  name      = "${each.key}"
```
and
```Bash
  listen_port = "${each.value}"
```

The **each.key** and **each.value** keywords are used to access either the key or the value within the map.

In my case, on each iteration, the following will be true:

```Bash
variable "tcp_lb" {
  type = map
  default = {
    unit-config-origin = "8888"
    unit-git-origin = "8080"
  }
}
```
One first iteration:

```Bash
each.key = unit-config-origin
each.value = 8888
```

On the second iteration:
```Bash
each.key = unit-git-origin
each.value = 8080
```

## Conclusion
Using terraform maps simplifies your code by reducing the number of resources that you need to duplicate. It also makes your code a lot more readable.

There are traps and pitfalls using this method if you have dependant resources, but I'll cover this in another post. :)
