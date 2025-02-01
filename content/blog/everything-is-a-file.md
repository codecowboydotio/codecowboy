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

All this does is print out its name and sleep for 20 seconds.

It is the building blocks and the basis for what we are about to do next.

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

I can compile my program using gcc as below.
```C 
gcc test.c
```

When I run my newly compiled program, it prints its name, waits 20 seconds and then exits. 
This is perfect. 

The name a.out is the default name given to a program when you don't specify an output file as a compile option.
```C
[root@fedora ~]# ./a.out
Program Name: ./a.out
```

The sleep for 20 seconds is important here because I can validate that my program is running by using the Linux **ps** command.

```Bash
[root@fedora development]# ps -ef | grep a.out | grep -v grep
root        4094    3237  0 20:38 pts/0    00:00:00 ./a.out
```

We can see that the program is running, and it has a name of a.out


# Change the name of the process
What if I want to change the name of the process **after** it has started?

This is relatively easy to achieve in C.

I add the following lines of code to my program. 
I add a **#define** to just define the name that I want to change my program to.
I add a **memset** to set the size of the program name to the size of the string innocuous name.
I add a **strcpy** to actually copy the innocuous name over the top of the program name.

```C 
#define INNOCUOUS_NAME "everything_is_a_file"
    memset(argv[0],0,strlen(argv[0]));
    strcpy(argv[0], INNOCUOUS_NAME);
```

The full code ow looks like this.
I have added another sleep command, and an additional print command so that I have time to run some **ps** commands and show that the program, once started renames itself.

```C
#include <stdio.h>
#include <string.h>

#define INNOCUOUS_NAME "everything_is_a_file"

int main(int argc, char *argv[]) {
    printf("Program Name: %s\n",argv[0]);
    sleep(20);
    memset(argv[0],0,strlen(argv[0]));
    strcpy(argv[0], INNOCUOUS_NAME);
    printf("Resetting program name: %s\n",argv[0]);
    sleep(60);
    return 0;
}
```

## Let's see it work.
In action, I compile my program again.

```C 
gcc test.c
```

Then I run the program.
The program prints out its initial name, which is just the binary name.
After 20 seconds, I overwrite the program name with the name **everything_is_a_file** which is the string in my **#define** in my code.

```C
[root@fedora ~]# ./a.out
Program Name: ./a.out
Resetting program name: everything_is_a_file
```

While running a **ps** on the output initially, I see that process **4128** is named **a.out**
```C
[root@fedora development]# ps -ef | grep a.out | grep -v grep
root        4128    3237  0 20:45 pts/0    00:00:00 ./a.out
```

After 20 seconds, the program renames itself. I run the **ps** command again, but this time I use the process id or **PID**, **4128**. I can see that the program name has changed to **everything_is_a_file**.

```C
[root@fedora development]# ps -ef | grep 4128 | grep -v grep
root        4128    3237  0 20:45 pts/0    00:00:00 everything_is_a_file
```

# Why do this?
There are two main use cases that I can think of here (there are probably more). The first is that if I am a nefarious actor, I can start a program and make it look like any other random process on my system.I Could make it look like a device driver for example by naming it "keyboard" or "udev" or something **innocouous**.

The second more useful non nefarious use case is where I may want to fork a process and name it in a particular way rather than simply using threads. This can be quite handy as a way of giving a human readable name to each thread.

# Conclusion

If I were nefarious and added some server side code, I could easily hide my running process in plain sight. I could start a remote shell, exfiltrate data, or do just about anything that I had access to do. I could do this while looking like an innocuous program that would be overlooked.

It is instructive and interesting to understand how things work on Linux systems. I hope this has given you an insight into how things work, and has piqued your interest in doing some further reading.

Happy programming!

@codecowboydotio
