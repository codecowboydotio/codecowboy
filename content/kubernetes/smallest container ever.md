+++
title = "Small Containers for fun "
date = "2021-07-09"
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

```C++
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

```Bash
[root@fedora]# gcc pausle.c -o pausle-dynamic
```

If I use the **ldd** command to see the dynamically linked libraries as part of my binary, you can see that there are three shared libraries used by the **pausle-dynamic** application.:1

```Bash
[root@fedora]# ldd pausle-dynamic
        linux-vdso.so.1 (0x00007ffc93da0000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f32a782c000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f32a7a06000)

```

Finally, we can see that the size of the dynamically linked binary is 25K.

```Bash
[root@fedora]# ls -lh pausle-dynamic
-rwxr-xr-x. 1 root root 25K Jul  9 21:48 pausle-dynamic
```

One would imagine that this is a very small application.

- It's dynamically linked - **should** be smaller
- It's not doing anything, and is minimal anyway

**What if I were to tell you that we could reduce the size of this application by 60%?**

### Compiler Options
What I realised is that if I **aggressively** tune my compiler options, I can go even smaller again.
Using the compiler options below I can compile my code to the smallest possible statically linked binary that I can.

```Bash
gcc  -Os -fdata-sections -ffunction-sections -fipa-pta  -Wl,--gc-sections -Wl,-O1 -Wl,--as-needed -Wl,--strip-all paus
le.c -o pausle-dynamic-aggressive
```

```Bash
[root@fedora]# ls -lh pausle-dynamic-aggressive
-rwxr-xr-x. 1 root root 15K Aug  3 20:34 pausle-dynamic-aggressive
```

Just by using agressive compiler options, the size of the code is smaller again.



#### Static Linking
Static linking is where I include all of the libraries for the executable "inside" the binary itself. This is useful when operating inside a container as it removes the need to have shared libraries (and hence an entire operating system). This in theory should make the entire container image smaller, even if the size of the binary is slightly larger.

To statically link the binary, I pass the -static option to the compiler.

```Bash
gcc -Os -s -static -ffunction-sections -fipa-pta  -Wl,--gc-sections pausle.c -o pausle-static
```

This creates a slightly larger binary size

```Bash
[root@fedora]# ls -lh pausle-static
-rwxr-xr-x. 1 root root 708K Aug  3 20:38 pausle-static
```

**WOAH** 708K.

If I look at the linking using ldd again, I get the following message.

```Bash
[root@fedora]# ldd pausle-static
        not a dynamic executable
```

This means that all of my libraries are included in the binary itself.

## Building containers

In order to build containers, I'm going to use the dockerfile format, it's simple and is mostly uderstood.

I use the following file 

```Bash
FROM registry.fedoraproject.org/fedora-minimal

ADD pausle-dynamic /
CMD ["/pausle-dynamic"]
```

I'm using a minimal fedora image here to build out my container image.
I add my dynamically linked binary into my image as a layer.

I can build this using the following command.

```Bash
[root@fedora]# podman build --tag=pausle-dynamic .
STEP 1: FROM registry.fedoraproject.org/fedora-minimal
Getting image source signatures
Copying blob 033c7516884e done
Copying config 241281a93a done
Writing manifest to image destination
Storing signatures
STEP 2: ADD pausle-dynamic /
--> 54ab00ee58c
STEP 3: CMD ["/pausle-dynamic"]
STEP 4: COMMIT pausle-dynamic
--> 96800d278c6
96800d278c61d69101791d416fd139a3e70afff2033063354c08307fa2eb118c
```

This builds me a container that we can see below.

```Bash
[root@fedora]# podman images
REPOSITORY                                   TAG           IMAGE ID      CREATED         SIZE
localhost/pausle-dynamic                     latest        96800d278c61  11 seconds ago  119 MB
```

This comes out to 119MB - remember that the original binary was only 25K in size!

### Using Scratch
Using the SCRATCH keyword, I can create a container that only has the required binary inside it.
It's still a container, but doesn't have any of the ancilliary "operating system" requirements. 
It doesn't have shared libraries, nor does it have any operating system tooling that you may expect to find.

```Bash
FROM scratch

ADD pausle-static /
CMD ["/pausle-static"]
```

I need to add my statically compiled binary here because there are no shared libraries available for a dynamically linked library to use.

Let's build this out.

```Bash
[root@fedora]# podman build --tag=pausle-static .
STEP 1: FROM scratch
STEP 2: ADD pausle-static /
--> 87ee8170167
STEP 3: CMD ["/pausle-static"]
STEP 4: COMMIT pausle-static
--> d6e9daeb1ce
d6e9daeb1ce4211e3f04dd535494a3e78228c3642d22d350d6e9d4f2241e5861
```

If I check my container image sizes we see the following.

```Bash
[root@fedora]# podman images
REPOSITORY                                   TAG           IMAGE ID      CREATED        SIZE
localhost/pausle-static                      latest        d6e9daeb1ce4  3 seconds ago  727 kB
```

You will notice that this container is only 727KB in size.
It doesn't have anything inside it EXCEPT the binary or the application that I'm going to run.

In this way, I can build very small and minimal container images.
Even though my initial binary size is larger when I statically compile, it much reduces the overall size of my container image.

### Other Benefits

Other benefits are that only having a single binary inside your container reduces the attack surface that's available from a security perspective. There are no dependencies that need to be individually lifecycle managed, and no real provenance in terms of ancialliary tooling inside the container.

The boot time of my container is much reduced because it's smaller - this is a good thing.

Next up I'll talk a little about telemetry and how this can be included in your application stacks while keeping a minimal container footprint.


