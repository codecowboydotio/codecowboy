+++
title = "Autonomous AI agents"
date = "2026-03-11"
aliases = ["ai"]
tags = ["ai", "dev"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Intro
I've been messing around with AI and "agents" recently. I thought I'd share the architecture that I've come up with. I was working with some orchestration systenms and thought to myself "I want something that is autonomous, but also able to be completely distributed".

I have used the fantastic library [libp2p](http://www.libp2p.io). This gives me the ability to have peer discovery, a pub / sub message bus and more. I thought to myself **"what if I combine my favourite network stack with the experiments I'm running with AI"**

## What is this?
I built out a multi agent architecture that allows me to start an agent, and optionally connect it to either an LLM (either claude or openai), an MCP server (any one), give it a prompt that gets executed, and topics that the agent will either publish to or subscribe to... or both.


![Architecture](/images/mcp-report-architecture.jpg)

![Agent Architecture](/images/autonomous-ai-agent-diagram.svg)
## The MCP Server
The MCP server is my standard Star Wars API (SWAPI) server and MCP that I have used previously. The MCP server has two tools 

- get_swapi_character (Get character information from SWAPI by ID)
- get_all_swapi_people (Get all people from SWAPI)

![Tools](/images/mcp-report-tools.jpg)

While this is a very simple implementation of tools within an MCP server, it serves the purpose of having data available to me.

{{< notice info >}}
I have inserted myself into the API as a "fake" character.
{{< /notice >}}

The intention here is that I want my agents to get data, write code and execute the code based on the data they see. This is a very standard use case. 


## 

# Summary

