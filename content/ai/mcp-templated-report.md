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
I am using a claude project here to make sure that context and instructions are all self contained. This makes it easier for me to ensure that the all of the templates and history are kept together.

![mcp-report-project.overview](/images/mcp-report-project-overview.jpg)

You can read about claude projects here: [https://www.anthropic.com/news/projects](https://www.anthropic.com/news/projects)

Part of creating a project in claude is creating instructions. The instructions tell claude what it needs to do whenever the project is used. 

![Project Instructions](/images/mcp-report-project-instructions.jpg)

The project instructions specify that we should:
- Create a document by using our template doc.
- Give the project a default name.
- Read the additional instructions within the template.
- Don't use the default star wars brand style!!!
- Ask the user if they want a document as a ppt or a doc.

All of these instructions are executed when this project is used. 

The two most important ones are to read the instructions within the template file, and to not use the black and yellow star wars branding (it makes the graphs undreadable).

## Template file
The template file is a standard word document. It is noteworthy because it has additional instructions embedded within it. 

The template uses a variable to set the project name, however it also has further prompts within it that will be read and executed upon invocation (as per the project instructions).

![Report Template](/images/mcp-report-template.jpg)

I have used relatively simple and straight forward prompts in my template, I am sure that you can use more complex prompts to get different functionality. 

One thing that I did not try, is placement of the explicit tool calls in the template. You will note that there is an explicit tool call in the project instructions though.

## Report generation
In order to generate a report, I simply open claude, navigate to the project and say **generate report**.

![step one](/images/mcp-report-step1.jpg)

The interesting thing here is that even though the project instruction to ask the user for a preferred output is the **last** project instruction, it is executed first. Claude knows that it cannot move forward without this data, so asks for it first. 

As part of the thought process that is used, we can see that claude has read the template, and the instructions and synthesised these. Claude also knows which tool to call. I have found where multiple MCP servers are connected, it is faster and useful to specify either the connection by name or the tool.

![step one](/images/mcp-report-step1.1.jpg)


The next step we can see that the correct tool call is used, and that the data is returned. This is where the client (claude) makes the tool call, and gets an API response from the the backend API.

![step three](/images/mcp-report-step2.jpg)

![step two](/images/mcp-api-response.jpg)

The next two steps are interesting. The first step is where the planet data is pulled out of the existing dataset that claude queried. Essentially it's a way of rearranging or re-keying the existing data. This is because the planet data is included in the person data. 

![step four](/images/mcp-report-step2.2.jpg)

The last piece of the puzzle, claude uses the inbuilt skill to export a pptx file for me to check.

![step five](/images/mcp-report-step2.3.jpg)

The outputs are simple. There is a graph of the planet data, and a table of the people with the fields that I requested in the template. Most amazingly, the branding is NOT in traditional black and yellow.

![step six](/images/mcp-report-step3.jpg)

![step seven](/images/mcp-report-step4.jpg)

## MCP Logs

## Output
On the server side we can see that the client (claude) made a call to list the tools, and then called the get_all_swapi_people tool. I will delve more into this in another post. 

```Shell
2026-01-16 03:09:20 - mcp.server.lowlevel.server - INFO - Processing request of type ListToolsRequest
2026-01-16 03:10:39 - mcp.server.lowlevel.server - INFO - Processing request of type CallToolRequest
2026-01-16 03:10:39 - root - INFO - Tool called: get_all_swapi_people with arguments: {}
2026-01-16 03:10:39 - root - INFO - Fetching all people from SWAPI
2026-01-16 03:10:39 - root - INFO - Making request to: http://localhost:3000/people/
2026-01-16 03:10:39 - httpx - INFO - HTTP Request: GET http://localhost:3000/people/ "HTTP/1.1 200 OK"
2026-01-16 03:10:39 - root - INFO - Successfully fetched data from: http://localhost:3000/people/
2026-01-16 03:10:39 - root - INFO - Successfully fetched 83 people
```

# Summary
If you're not using claude and MCP for automatic report generation (and more) you're missing out. The amount of effort to set all of this up was minimal. The template took roughly 5 minutes to create. The project instructions were similar. I did iterate the project instructions however I found that this was also minimal. 

I see the use cases for this being plentiful. These include, generating reports for regulators, and any other report that you need to have regularly.

I highly recommend using MCP and claude projects for your report generation needs. It's easy, and amazingly flexible!
