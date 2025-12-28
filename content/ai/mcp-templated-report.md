+++
title = "MCP server templated reports"
date = "2025-12-28"
aliases = ["ai"]
tags = ["ai", "dev", "claude"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Intro
I have been fooling around a lot with ai recently, and one of the things I've found myself doing is integrating with MCP servers and then using the projects feature in claude to generate custom reports.

## What is this?
I have set up a remote MCP server that pulls data from the Star Wars API (swapi). This is a remote MCP server, so I am hosting it on one of my domains. To keep this post short and on topic, I will confine myself to the claude components and not talk about the specifics of the MCP server itself. It's there, it works, and I can connect to it.

![Architecture](/images/mcp-report-architecture.jpg)


## The MCP Server
The MCP server has two tools 

- get_swapi_character (Get character information from SWAPI by ID)
- get_all_swapi_people (Get all people from SWAPI)

![Tools](/images/mcp-report-tools.jpg)

While this is a very simple implementation of tools within an MCP server, it serves the purpose of generating a report using an MCP and the available tools.

## Claude project

## Template file

## Report generation

## Output


# Summary
This has been a long journey so far for the reader, but for me as a developer it's been very quick. It took me only a few minutes to refactor the code into a service using Claude. I have a robust service now that allows me to get Dockerfiles updated without too much difficulty. I just need to feed in the right paramters and away it goes, and performs the checks on my behalf, does the analysis, and updates the repository.

This is a neat way to keep your Dockerfiles updated and your repos clean!

