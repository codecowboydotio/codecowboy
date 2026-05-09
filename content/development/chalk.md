+++
title = "Chalk project"
date = "2026-04-28"
aliases = ["chalk"]
tags = ["chalk", "dev", "security"]
categories = ["ai", "security", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Intro
I recently came across a very interesting open source project called the chalk project. It's headline is "like an airtag for your development environment". This got me interested enough to investigate it and explore it a litte.

[https://chalkproject.io/](https://chalkproject.io/)

## What is this?
The chalk project aims to provide a few things, security and provenance. There are a bunch of others, but these are the two things that drew me to the project initially.

## Installation
Installation is very straight forward. You run the commands to download the binary and away you go.

First we run curl to download and install chalk. Then we make it executable, and move it to a sensible location while adding it to our path.

```Bash
version=$(curl -fsSL https://dl.crashoverride.run/chalk/current-version.txt)
wget https://dl.crashoverride.run/chalk/chalk-$version-$(uname -s)-$(uname -m){,.sha256}

chmod +x chalk-$version-$(uname -s)-$(uname -m)

cp chalk-$version-$(uname -s)-$(uname -m) ~/.local/bin/chalk
export PATH=~/.local/bin:$PATH
```

Once all of this is done, you can validate everything by simply checking the chalk version.

```Bash
chalk version


 ┌┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┬┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┐
 ┊ Chalk Version  ┊ 0.6.8                                       ┊
 ┊ Chalk ID       ┊ CDH6CS-HRC8-T64R-9RCCS3                     ┊
 ┊ Commit ID      ┊ dc27f1d75336502e584bc7d86f600f6bbdf3967a    ┊
 ┊ Build OS       ┊ linux                                       ┊
 ┊ Build CPU      ┊ amd64                                       ┊
 ┊ Build Date     ┊ 2026-02-23                                  ┊
 ┊ Build Time     ┊ 13:29:31                                    ┊
 ┊ Docker Client  ┊ 29.1.2                                      ┊
 ┊ Docker Server  ┊ 29.1.2                                      ┊
 ┊ Buildx         ┊ 0.30.1                                      ┊
 ┊ Cosign         ┊ 2.2.3                                       ┊
 └┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┴┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┘


```

## Initial configuration
The very first thing you will want to do is to run a **chalk setup**. This configures chalk and also downloads cosign. More importantly, it gives you a chalk password which you need for commands later on.

```Bash
[root@foo content]# chalk setup
------------------------------------------
CHALK_PASSWORD=some_password
------------------------------------------
Write this down. In future chalk commands, you will need
to provide it via CHALK_PASSWORD environment variable.

info:  cosign: signing file /root/bin/chalk-0.6.5-Linux-x86_64
info:  Configuration replaced in binary: /root/bin/chalk-0.6.5-Linux-x86_64
info:  /root/.local/chalk/chalk.log: Open (sink conf='default_out')
info:  Full chalk report appended to: ~/.local/chalk/chalk.log
```

From here you should be ready to go and start adding chalk marks.


## Chalk Environment
You can check the chalk environment by using the **chalk env** command. This allows you to see key information about the current chalk environment.

```Bash
# chalk env

info:  Full chalk report appended to: ~/.local/chalk/chalk.log
[
  {
    "_OPERATION": "env",
    "_DATETIME": "2026-02-13T19:10:03.226+11:00",
    "_ENV": {
      "PWD": "/root/chalk-test",
      "XDG_SESSION_TYPE": "tty",
      "USER": "root",
      "PATH": "/usr/local/go/bin:/root/.local/bin:/root/bin:/usr/local/bin:/usr/bin:/opt/mssql-tools/bin:/root/.pulumi/bin:/root/bin:/usr/local/go/bin:.:/opt/mssql-tools/bin",
      "SSH_TTY": "/dev/pts/2"
    },
    "_OP_ARGV": [
      "/root/bin/chalk-0.6.5-Linux-x86_64",
      "env"
    ],
    "_OP_CHALKER_VERSION": "0.6.5",
    "_OP_CHALK_COUNT": 0,
    "_OP_UNMARKED_COUNT": 0
  }
]
```

## Chalking a binary
The docs have a really nice way to get started, which is chalking a binary. This is interesting for a number of reasons, least of which is that when you chalk binary files, you end up placing data inside the binary.

First we copy the ls command to the current diretory. Mainly so that we do not modify the system one.

```Bash
cp $(which ls) ./
chalk insert ls
```

This inserts a chalk mark inside the ls binary.



```Bash
[root@foo chalk-test]# chalk commands
warn:  Could not find or install cosign; cannot sign or verify.
[root@foo chalk-test]# chalk extract ./ls
warn:  Could not find or install cosign; cannot sign or verify.
info:  /root/chalk-test/ls: Chalk mark extracted

info:  /root/.local/chalk/chalk.log: Open (sink conf='default_out')
info:  Full chalk report appended to: ~/.local/chalk/chalk.log
[
  {
    "_OPERATION": "extract",
    "_DATETIME": "2026-03-22T16:18:59.349+11:00",
    "_CHALKS": [
      {
        "CHALK_ID": "CRR62D-K5CM-SP6E-9R6GSK",
        "CHALK_VERSION": "0.6.8",
        "ARTIFACT_TYPE": "ELF",
        "METADATA_ID": "0PQNZ9-W3YE-1P9V-ZPYPXZ",
        "_OP_ARTIFACT_PATH": "/root/chalk-test/ls",
        "_OP_ARTIFACT_TYPE": "ELF",
        "_CURRENT_HASH": "4fbb16e63435787b42a21907d5e72d08f219d6e2e977be5be003c760553230e8"
      }
    ],
    "_ENV": {
      "PWD": "/root/chalk-test",
      "XDG_SESSION_TYPE": "tty",
      "USER": "root",
      "PATH": "/usr/local/go/bin:/root/.local/bin:/root/bin:/usr/local/bin:/usr/bin:/opt/mssql-tools/bin:/root/.pulumi/bin:/root/bin:/usr/local/go/bin:.:/opt/mssql-tools/bin",
      "SSH_TTY": "/dev/pts/1"
    },
    "_OP_ARGV": [
      "/root/bin/chalk-0.6.8-Linux-x86_64",
      "extract",
      "./ls"
    ],
    "_OP_CHALKER_VERSION": "0.6.8",
    "_OP_CHALK_COUNT": 1,
    "_OP_UNMARKED_COUNT": 0
  }
]
```

## Signing and attestation

One of the very useful use cases for chalk is signing and attestation of artifacts. This allows me to absolutely know that the artifact I am using is the same artifact that I have previously signed. You can sign binaries, container images and more. 

An example of signing an artifact is below.

```Bash
 CHALK_PASSWORD=XXX chalk insert ./ls
info:  Test sign successful.
info:  Test verify successful.
info:  /root/chalk-test/ls: Existing chalk mark extracted
info:  cosign: signing file /root/chalk-test/ls
info:  /root/chalk-test/ls: chalk mark successfully added
info:  /root/.local/chalk/chalk.log: Open (sink conf='default_out')
info:  Full chalk report appended to: ~/.local/chalk/chalk.log
[
  {
    "_OPERATION": "insert",
    "_DATETIME": "2026-05-07T19:58:31.195+10:00",
    "_CHALKS": [
      {
        "CHALK_ID": "CRR62D-K5CM-SP6E-9R6GSK",
        "PRE_CHALK_HASH": "19504e2afd1817da6edfab46f61e4842eabb7043eee68b125aca4                               20ec4397734",
        "PATH_WHEN_CHALKED": "/root/chalk-test/ls",
        "ARTIFACT_TYPE": "ELF",
        "CHALK_VERSION": "0.6.8",
        "METADATA_ID": "QF2P96-R00C-KKSJ-E59FD3",
        "SIGNATURE": "MEYCIQD44GDn3Mvp59RL2jA+o0bd2pj6cCKownoQ0fsR89Z8FAIhAJrf6M                               2kC4nvlb+u2sbTzW9dKzDfJFXTH2332/IaawRC",
        "_VIRTUAL": false,
        "_CURRENT_HASH": "19504e2afd1817da6edfab46f61e4842eabb7043eee68b125aca42                               0ec4397734"
      }
    ],
    "_ENV": {
      "PWD": "/root/chalk-test",
      "XDG_SESSION_TYPE": "tty",
      "USER": "root",
      "PATH": "/usr/local/go/bin:/usr/local/go/bin:/usr/local/go/bin:/root/.loca                               l/bin:/root/bin:/usr/local/bin:/usr/bin:/opt/mssql-tools/bin:/root/.pulumi/bin:/                               root/bin:/usr/local/go/bin:.:/opt/mssql-tools/bin:/opt/mssql-tools/bin:/root/.pu                               lumi/bin:/root/bin:/usr/local/go/bin:.:/opt/mssql-tools/bin:/opt/mssql-tools/bin                               :/root/.pulumi/bin:/root/bin:/usr/local/go/bin:.:/opt/mssql-tools/bin",
      "SSH_TTY": "/dev/pts/0"
    },
    "_OP_ARGV": [
      "/root/bin/chalk-0.6.8-Linux-x86_64",
      "insert",
      "./ls"
    ],
    "_OP_CHALKER_VERSION": "0.6.8",
    "_OP_CHALK_COUNT": 1,
    "_OP_UNMARKED_COUNT": 0
  }
```

If I want to validate the signature, I can run the chalk extract command.

```Bash
# chalk extract ./ls 2>/dev/null | jq -r '.[]|._CHALKS[]|._VALIDATED_SIGNATURE'
true
```
Chalk will return a value of true or null depending on whether or not the siganture returns a value. This is very useful is you're wanting to validate something before running it, or before deploying it.

All of these commands can of course be embedded into a pipeline.


# Summary
Chalk is a good option for signing artifacts at build time, and or attesting to their identity. There are a myriad of use cases that you can do here. 

One that I have used is signing code that is generated by an AI agent, and validating it before it is run. This answers the questions "is this the same code" and "is it the right code" before it is being run.
