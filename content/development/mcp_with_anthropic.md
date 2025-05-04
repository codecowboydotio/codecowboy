+++
title = "Creating and MCP server with Anthropic"
date = "2025-05-04"
aliases = ["dev"]
tags = ["dev"]
categories = ["software", "dev"]
author = "codecowboy.io"
+++

# Intro
There has been a lot of hype on socials recently about MCP or Model Context Protocol. Essentially it's a protocol that allows LLM AI to be extended. Most of the extensions relate to providing context using your own data. 

This is what I am going to walk through here. 

I will walk through:
- Creating an MCP
- What works and what works better
- My discoveries so far


# TLDR
If you just want the code - it's here:

[https://github.com/codecowboydotio/mcp-examples](https://github.com/codecowboydotio/mcp-examples)


# Why
The reason I decided to initially play around with MCP was because I had a **high level of scepticism** regarding it. I've seen a **LOT** of posts on the internet that talk about it, how it's transformational and so on. I thought to myself "I'm going to look into this, but I'm going to do it bottom up and by writing a server". 

# Where to start
I started at the following address:

[https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)

This describes the MCP protocol, but also describes some of the capabilities. 
I then decided to to read through the examples here:

[https://modelcontextprotocol.io/quickstart/server](https://modelcontextprotocol.io/quickstart/server)

This is a great example, because it covered my initial use case almost exactly. It's an example of a weather application that is a REST based API. 

After reading through the example, I decided to set up my development environment and start writing some code.

# Install the environment
In order to install the environment, you need to grab a copy of the Anthropic client, and then install all of the pre-requisite software.

[https://claude.ai/download](https://claude.ai/download)

Once installed, I needed to get the rest of the development environment up and running. I chose python as the language of choice to develop in - but other SDKs are available (Python, Typescript, Java, Kotlin and C#).

I am also running on windows, so my configuration examples will use windows nomenclature.

Next it's time to install uv for python, it's an alternative package manager, but you can use pip also if you're more comfortable with it.

```Shell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Then we create a virtual environment using uv, and activate it (it's the same as any other venv in the pythonic world).

```Shell
# Create a new directory for our project
uv init swapi
cd swapi

# Create virtual environment and activate it
uv venv
.venv\Scripts\activate

# Install dependencies
uv add mcp[cli] httpx json requests

# Create our server file
new-item swapi.py
```

From here we are ready to start writing some code.

# Basic MCP server

# The results

# SWAPI 

## Specific searching

## Generalised searching

## Datasets

# Things I learned

# Conclusion
