+++
title = "Mermaid Test"
date = "2022-03-28"
aliases = ["terraform_maps"]
tags = ["terraform", "iac"]
categories = ["automation", "software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I've been doing a lot with terraform lately, and I've been looking for ways to make my terraform configurations a lot simpler and have less repetition. Like a lot of people, I've found myself repeating the same code over and over. An example is where I repeat the same resource over and over but with different configuration parameters. It's essentially the same resource. Why should I do this? **There has to be a better way.**

<div class="mermaid">
graph LR;
  A-->B;
</div>
<script async src="https://unpkg.com/mermaid@8.2.3/dist/mermaid.min.js"></script>
