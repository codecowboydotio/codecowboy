+++
title = "Everything is a file"
date = "2022-10-23"
aliases = ["dev"]
tags = ["dev"]
categories = ["software", "dev", "programming"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I've been programming in C on unix for a long time - too long in fact. 

I was reminded recently by someone that **everything is a file** in Linux / Unix.

{{< notice tip >}}
I am going to use the term Linux to cover off both Unix and Linux from here on
{{< /notice >}}


I thought that I would write about this with some code examples, and given I'm currently working for a security company, I thought it would be useful to point out how to use this functionality for evil rather than good from a security perspective.

# Understanding Unix means understanding C

The rationale here is that if you understand how things work, you can better defend against them.

To really understand Linux, you need to understand C. 
It's the language that the operating system is written in, and it's the language that allows you to interact with Linux at a level that allows for a lot of fun.

I'm going to walk through how to do some things in Linux that might surprise you.

## Everything is a file
On a Linux system, everything is a file. **Everything**

This means, regular files, devices, memory, processes, **everything**

There's a great wikipedia page on this [here](https://en.wikipedia.org/wiki/Everything_is_a_file)

Essentially, everything is passeed through the filesystem namespace - **everything is a file**.


# A Simple program
A very simple program that I have is as follows.

```C
[root@fedora ~]# more test.c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    printf("Program Name: %s\n",argv[0]);
    sleep(20);
    return 0;
}

```

```C
[root@fedora ~]# ./a.out
Program Name: ./a.out
```

# Change the name of the process

# Server side code

# Hiding in plain site

# Conclusion

Look out for my next post where I will talk about pre-commit hooks and how they can be managed 
