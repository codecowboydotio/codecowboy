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

# High level overview
The following diagram represents a high level overview of what has been built and how it works.

![MCP high level overview](/images/MCP-highlevel-1.jpg)

...and don't worry there are more diagrams that show more detail further on :)

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

# Star Wars API
I am running a **local** version of the star wars API as a container.
I also **inserted myself** into the API as an easy way to validate that my MCP server is asking my API and not deriving data from other sources or its trained dataset.

{{< notice info >}}
There is a detailed explanation of the API I am using, as this feeds directly into the question of "specific" versus "generalised" for LLM AI models and MCP.
{{< /notice >}}


The API that I use is quite specific and has a number of routes:

![SWAPI swagger](/images/swapi-swagger.jpg)

You will note that each of the routes has two possible requests:

/route/ - Return **ALL** records
/route/{id} - Return a specific record

## All records
All records looks like this:

```Shell
curl -s 10.1.1.150:3000/people/ | more
```
```Json
[
  {
    "edited": "2014-12-20T21:17:56.891Z",
    "name": "Luke Skywalker",
    "created": "2014-12-09T13:50:51.644Z",
    "gender": "male",
    "skin_color": "fair",
    "hair_color": "blond",
    "height": "172",
    "eye_color": "blue",
    "mass": "77",
    "homeworld": 1,
    "birth_year": "19BBY",
    "image": "luke_skywalker.jpg",
    "id": 1,
    "vehicles": [
      14,
      30
    ],
    "starships": [
      12,
      22
    ],
    "films": [
      1,
      2,
      3,
      6
    ]
  },
  {
    "edited": "2014-12-20T21:17:50.309Z",
    "name": "C-3PO",
    "created": "2014-12-10T15:10:51.357Z",
    "gender": "n/a",
    "skin_color": "gold",
    "hair_color": "n/a",
    "height": "167",
    "eye_color": "yellow",
    "mass": "75",
    "homeworld": 1,
    "birth_year": "112BBY",
    "image": "c-3po.jpg",
    "id": 2,
    "vehicles": [],
    "starships": [],
    "films": [
      1,
      2,
      3,
      4,
      5,
      6
    ]
  },
```

It is returned as a JSON array.
Note the **id** element. The API is keyed on **id**. So when I get an individual people instance, I will see the following:

## Single record
A single record looks like this

```Shell
curl -s 10.1.1.150:3000/people/1 | more
```
```Json
{
  "edited": "2014-12-20T21:17:56.891Z",
  "name": "Luke Skywalker",
  "created": "2014-12-09T13:50:51.644Z",
  "gender": "male",
  "skin_color": "fair",
  "hair_color": "blond",
  "height": "172",
  "eye_color": "blue",
  "mass": "77",
  "homeworld": 1,
  "birth_year": "19BBY",
  "image": "luke_skywalker.jpg",
  "id": 1,
  "vehicles": [
    14,
    30
  ],
  "starships": [
    12,
    22
  ],
  "films": [
    1,
    2,
    3,
    6
  ]
}
```

## My own record
I inserted myself as record **id** 84.
The idea here is that this is the ultimate test of not only the MCP server, but is also a test of generalised reasoning.

```Shell
 curl -s http://10.1.1.150:3000/people/84
```
```Json
{
  "edited": "2014-12-20T21:17:56.891Z",
  "name": "Scott",
  "created": "2014-12-09T13:50:51.644Z",
  "gender": "male",
  "skin_color": "fair",
  "hair_color": "blonde",
  "height": "184",
  "eye_color": "blue",
  "mass": "01",
  "homeworld": 1,
  "birth_year": "19BBY",
  "image": "luke_skywalker.jpg",
  "id": 84,
  "vehicles": [],
  "starships": [],
  "films": []
}
```

# Basic MCP server
Initially I thought like a REST API dev, and built the diagram below:

![MCP swapi single person](/images/MCP-swapi-single-person.jpg)

I wrote an MCP server (based on the example server) that pulls back a **single** person from the star wars database. 

## The code
I used the code from the example server, and **charged ahead** with typical REST based "query for the smallest thing" logic.

A diagram of the MCP tool flow is below:

![MCP tool flow](/images/MCP-tool-flow.jpg)

1. Client is asked "get swapi character id 1"
2. MCP tool calls **make_request**
3. make_request queries the external API
4. external API returns a JSON value for a single person with the ID of 1
5. The **make_request** function returns RAW JSON data to the MCP tool
6. The MCP tool called the **format_msg** function.
7. The **format_msg** function parses the JSON response and formats it
8. The formatted response is sent back to the client


```Python
from typing import Any
import httpx
import json
import requests
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("swapi")

# Constants
API_BASE = "http://10.1.1.150:3000"
USER_AGENT = "swapi-app/1.0"


async def make_request(url: str) -> dict[str, Any] | None:
    """Make a request to the API with proper error handling."""
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/json"
    }
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None


def format_msg(json_string):
    """Format an alert feature into a readable string."""
    props = json_string
    homeworld=json_string['homeworld']
    homeworld_url="http://10.1.1.150:3000/planets/" + str(homeworld)
    homeworld_name = requests.get(homeworld_url)
    homeworld_json=json.loads(homeworld_name.text)

    return f"""
Name: {props.get('name', 'Unknown')}
Gender: {props.get('gender', 'Unknown')}
Hair Colour: {props.get('hair_color', 'Unknown')}
Homeworld: {homeworld_json.get('name', 'Unknown')}
"""


@mcp.tool()
async def get_swapi_character(id: str) -> str:
    """Get name from swapi api.

    Args:
        id: exact name match
    """
    url = f"{API_BASE}/people/{id}"
    data = await make_request(url)

    if not data:
        return "Unable to fetch data from API."

    #alerts = [format_alert(feature) for feature in data["features"]]
    msg = format_msg(data)
    return msg

if __name__ == "__main__":
    # Initialize and run the server
    mcp.run(transport='stdio')
```

Note that the MCP tool is making a specific request for an ID.

In the desktop application it looks like this:

![MCP swapi Luke](/images/MCP-swapi-character-1.jpg)

# The results

# SWAPI 

## Specific searching

## Generalised searching

## Datasets

# Things I learned

# Conclusion
