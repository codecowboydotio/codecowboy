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

I have used the fantastic library [libp2p](www.libp2p.org)

When I use the word agent, I generally mean a process that runs continuously.

## What is this?
![Architecture](/images/mcp-report-architecture.jpg)


## The MCP Server
The MCP server has two tools 

- get_swapi_character (Get character information from SWAPI by ID)
- get_all_swapi_people (Get all people from SWAPI)

![Tools](/images/mcp-report-tools.jpg)

While this is a very simple implementation of tools within an MCP server, it serves the purpose of generating a report using an MCP and the available tools.

# Summary

