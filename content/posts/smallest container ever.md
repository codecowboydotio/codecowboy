+++
title = "Smallest Container Ever 17.7kB"
date = "2020-07-09"
aliases = ["smallest_container"]
tags = ["containers", "minimal"]
categories = ["containers", "testing"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I have a side hobby where I try to create minimal container images.
There are a number of reasons for this, but the primary one is because I'm a complete geek.

Other reasons include:

- It's a neatness thing
- There's an engineering imperative in me to be minimalist
- I don't like waste
- The process of creation helps me understand the technology a little better
- Smaller images are great for testing at scale

## Why
The why I do this is mainly because it's a hobby, but in this case, I started the ultra minimal container image build
because someone I work with wanted to build a container for testing Istio latency at scale. He needed a container image that was ultimately small, so that he could spin up in his words 'about 10,000' at once. The other goal was to minimise any application response time so that he could test Istio rather than the application behind Istio.

## Building a minimal image
Building a minimal image is easy, and I've spoken about it before at Container Camp, and you can see the [video](https://www.youtube.com/watch?v=SWwd4uTVeF0) if you like. 

What I have realised is that there are two parts to building a minimal image:

- Using SCRATCH and only including your application in your image.
- Statically linking your binary - but doing so in a super strict and optimised way.

When I spoke at Container Camp a few years ago, I only used default options during my compile. I have since realised that I can go even smaller by optimising my compiler options!

## The Code
This is the easy part. 

I needed an "application" that didn't introduce any latency at all so that my colleague could perform Istio testing. I decided to go as minimal as possible and create an "application" that actually doesn't do anything.

In order for the application to work within a containerised environment, it needs to continue to run, and never exit. If the application ever exits, the container will stop running.

C is my chosen language of choice today, and I created the following application.

```
#include <unistd.h>

int main(void) { 
    return pause();
}
```

This tiny little code block does a single thing - it pauses the thread until a signal is received. 

In container terms, this means the container will run, and will wait for a valid signal, and then honour that signal. This is the perfect behaviour for container testing. The "application" in the container doesn't do anything, so doesn't add any latency to the transaction, and is perfect for testing.

## The compilation 
In order to meet my goal of making a really small container image, I need to compile the code. I need to compile the code in such a way that the code does not require external libraries. That is, the code is **statically linked**. The second thing to make the application smaller is to compile the code using compiler options that strip out everything that is extraneous.

### Linking
There are generally two types on linking when compiling code, and it's relevant for containers. 

#### Dynamic Linking
When using dynamic linking, shared libraries within the operatig system are used. This has a good side benefit of making the executable smaller (typically). The rationale behind dynamic linking is that when you are running multiple applications on a single operating system, it is possible for multiple applications to use common shared libraries. This makes each individual application smaller.o

The code snippet below creates a dynamically linked binary from my C code above. I've named the application "pausle" because all it does is "pause".

```
[root@fedora]# gcc pausle.c -o pausle-dynamic
```

If I use the **ldd** command to see the dynamically linked libraries as part of my binary, you can see that there are three shared libraries used by the **pausle-dynamic** application.:1

```
[root@fedora]# ldd pausle-dynamic
        linux-vdso.so.1 (0x00007ffc93da0000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f32a782c000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f32a7a06000)

```

Finally, we can see that the size of the dynamically linked binary is 25K.

```
[root@fedora]# ls -lh pausle-dynamic
-rwxr-xr-x. 1 root root 25K Jul  9 21:48 pausle-dynamic
```

One would imagine that this is a very small application.

- It's dynamically linked - **should** be smaller
- It's not doing anything, and is minimal anyway

**What if I were to tell you that we could reduce the size of this application by 60%?**

#### Static Linking

### Compiler Options


