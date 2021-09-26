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
variable "origins" {
  type = map
  default = {
    unit-config-origin = "8888"
    unit-git-origin = "8080"
    unit-app-origin = "8181"
  }
}
```

## A more complex example

## Conclusion
