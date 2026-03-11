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

![Agent Architecture](/images/autonomous-ai-agent-diagram.svg)

## Peer to peer topology
I am using [libp2p](http://www.libp2p.io) to provide networking services for my stack. This gives me a full peer to peer network where each node or agent can perform discovery. I am using multicast as my discovery mechanism. This allows me to keep it simple and prove out the discovery portion of my architecture. The library has other discovery mecahnisms, but I specifically chose multicast for ease of use. This also allows the nodes to form a mesh network where all nodes can see all other nodes.

This is important because each node sees both join and leave messages from nodes when they either join or leave the network.

I am also using a message bus on top of the mesh network. I am using floodsub as a publish / subscribe mechanism. Again this is an easy implementation. The documentation for the protocol says the following: 

**With flooding, routing is almost trivial: for each incoming message, forward to all known peers in the topic.**

This topology is simple and flexible - new agents can be added at any time and begin participating in the network. Agents that are done can finish and leave the network.

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

## Agent A
Agent A has one job - get the data from the MCP server and don't truncate it. In order to do this, I get the agent to make an MCP call using a prompt.

{{< notice info >}}
Use the mcp server to get all swapi people. List all of the people, do not cut the list short.
{{< /notice >}}

The agent then calls the MCP server, selects the appropriate tool, and gets a dump of data. This could easily have been a REST call, but it's a lot more flexible to simply feed the agent a prompt.

You can see from the agent configuration that this agent uses the prompt when it makes an MCP call.

![Agent A](/images/autonomous-ai-agent-mcp-agent.jpg)

Agent A will broadcast this data as a message to the bus, and all other agents that are subscribed to the same topic will pick up the message and process it. If other agents are not subscribed to the same topic then they will simply ignore the message, but will receive it over the network.

## Agent B
Agent B does a little more work. Agent B picks up the broadcast message, which is a list of all star wars characters. Agent B then sends this message to the LLM using the following prompt.

{{< notice info >}}
If there are any characters that look fake, list all of their attributes as json. Create a python script that will list the attributes from the json payload. Do not include a shebang at the beginning of the script. I will be using python exec() for execution. Return only the script without any other commentary. The script will be streamed as is and run, as such it should be returned as raw code only, no markdown formatting.
{{< /notice >}}

This instructs the agent to use the LLM to slice the data while also looking for anomalies. Importantly, the agent will **only** take action if there appear to be fake users. 

![Agent B](/images/autonomous-ai-agent-code-gen-agent.jpg)

If there are fake users, then the LLM will generate a script that can be passed to agent C, and run. 

![Agent B](/images/autonomous-ai-agent-code-received.jpg)

The code that is received is displayed using the "code received" in the UI.

Note that it is agent B that performs the evaluation of whether or not there are fake characters, and whether or not code needs to be generated.

## Agent C
Agent B publishes its code to the agent C. Agent C is subscribed to the "code-exec" topic and consumes the message (which is a complete script that can be run using the python **exec()** 
function). Agent C then runs the code. Agent C has no smarts and just runs whatever is sent to it. 

![Agent C](/images/autonomous-ai-agent-code-executed.jpg)

The executed result correctly identifies that I have been inserted into the star wars universe and should not be there.

{{< notice info >}}
I am thinking about using the chalk project to place chalk marks as attestations of the code that is generated by the agent that can be checked by agent C before being run - but we aren't there yet!
{{< /notice >}}

# The code
The code is relatively simple, it's roughly 1700 lines of python. 

1700 I hear you say - why!?!?!?!?!

This allows me to have a single code base and run the same code for all agents. Essentially, I pass different options to the agent to configure them. This is the essence of operating in a peer to peer network. There are no single task anything - everything is the same and uniform.

# Summary
This is a flexible architecture that allows me to run multiple agents that will autonomously talk to each other and perform work. The only things that I need to change are the prompts and the pub/sub topics. 

As an added benefit, I can farm work out to multiple AI providers. While this example uses claude, I can equally send work to openAI and get the same result. 
