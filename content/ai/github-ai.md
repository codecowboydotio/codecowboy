+++
title = "Github dockerfile service using AI - Part 1"
date = "2025-11-24"
aliases = ["ai"]
tags = ["ai", "dev", "claude"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
+++

# Intro
I have been fooling around a lot with ai recently, and I thought I would write something about what I've been doing. There are a few things that I've been doing, and they're all fascinating. 

- Writing average looking code using minimum viable prompts.
- Getting this average looking code refactored to be a robust service.
- Generating API documentation - and having Claude fix this code.

This is part one of a small series that I have created to walk through the process I went through to get decent code.

## A crazy idea - update my dockerfiles
I had a crazy idea. I thought to myself, let's write something that will go through my git repos and automagially update my dockerfiles so that the dockerfile uses a fixed but more recent version of the base image.

Most Dockerfiles have a fixed base image line that looks something like this:

```Shell
FROM docker.io/library/ubuntu:20.04
```

This is painful to trawl through and update. Not least because I don't actually know what the latest version is, and I'm not particularly keen on just using the latest tag. In addition to this, depending on the version of OS that I'm using (Alpine, Ubuntu etc) I don't want to have to keep on top of every single image and tag.

I wanted something that would do this for me. 

This seemed like the perfect use for an LLM and the ability to get the LLM to simply tell me what the latest version for a given base image is.

## Version 1 - the make it work code
My very first version of code, was **hand written** and was **very procedural**. This is the "make it work" code, where I experimented with several different ways of making calls to the LLM and github.

This is not the full code base, but rather the relevant parts for version one.

Initially, I make a REST call to the github API to get the file contents of my github repository. Note that the repo is static in this version.

The appropriate headers are passed, and the appropriate token. After performing a GET request, the answer is a list of files from the repository. 

```Python
url='https://api.github.com/repos/codecowboydotio/swapi-json-server/contents/'

headers = {
  "Accept": "application/vnd.github+json",
  "X-GitHub-Api-Version": "2022-11-28",
  "Authorization": f'Bearer {GIT_TOKEN}'
}

query_answer = requests.get(url, headers=headers)
query_response_code = query_answer.status_code
if query_response_code  == 204:
  print("Please check github actions for the status of the deployment")
else:
  print("Contacting git....")
  print("Got response...")
```

The response from the code above is a JSON response of files that looks something like this:

```Json
[{"name":".file","path":".file","sha":"XXX","size":0,"url":"https://api.github.com/repos/codecowboydotio/swapi-json-server/contents/.file?ref=main","html_url":"https://github.com/codecowboydotio/swapi-json-server/tree/main/.file","git_url":"https://api.github.com/repos/codecowboydotio/swapi-json-server/git/trees/XXX","download_url":null,"type":"dir","_links":{"self":"https://api.github.com/repos/codecowboydotio/swapi-json-server/contents/.file?ref=main","git":"https://api.github.com/repos/codecowboydotio/swapi-json-server/git/trees/XXX","html":"https://github.com/codecowboydotio/swapi-json-server/tree/main/.file"}}
```

The JSON blob contains all of the information that I need to determine if my file is a Dockerfile, and the location of that file.

## Enter the first LLM call
What happens next is that I use Langchain to instantiate the claude opus 4.1 model. The reason for langchain here is that I have been experimenting with Langchain and how it fits into my development processes. It does provide an interesting way of allowing for structured output - which allows a model to return data in a predictable way.

[https://docs.langchain.com/oss/python/langchain/structured-output](https://docs.langchain.com/oss/python/langchain/structured-output)

```Python
from langchain.chat_models import init_chat_model

model = init_chat_model("claude-opus-4-1-20250805", model_provider="anthropic")

dockerfiles_json_schema = {
    "title": "dockerfile",
    "description": "Format of dockerfile links",
    "type": "object",
    "properties": {
        "textresponse": {
            "type": "string",
            "description": "The text response portion",
        },
        "fileurl": {
            "type": "string",
            "description": "The actual url of the file",
        },
    },
    "required": ["textresponse", "fileurl"],
}

structured_model = model.with_structured_output(dockerfiles_json_schema)
print("Initializing model...")
try:
  response = structured_model.invoke(f'find dockerfiles in array {query_answer.text} return only value download_url')
except Exception as e:
  print(f"LLM error: {e}")
  exit(1)
print("Found dockerfile...")
```

Structured output allows me to define a JSON schema of what I would like the output to look like. This allows me to pass a schema of what I would like the output to look like, and than when I instanitate the model, reference the schema.

I essentially split the structured output into "textresponse" or junk, and "fileurl" which is the information I really want.

Finally, I use a prompt that invokes the model using my structured output schema.

```Python
structured_model = model.with_structured_output(dockerfiles_json_schema)
print("Initializing model...")
try:
  response = structured_model.invoke(f'find dockerfiles in array {query_answer.text} return only value download_url')
```

One important thing to note here is that I have used a "cut down prompt". Not content with learning one thing at a time, I decided to also experiment with smaller prompts, which reduces the context window and tokens that I use. The simple prompt:

```Shell
find dockerfiles in array {query_answer.text} return only value download_url
```

Is missing the word **the** among some other syntactic values if you were asking a person the same thing. The meaning is still 100% clear, but I have saved on some tokens. There is another blog post on this coming soon.

The output from this section if raw output and debug were enabled would look like this:

```Shell
Found dockerfile...
{'textresponse': 'Found 1 Dockerfile in the repository:', 'fileurl': 'https://raw.githubusercontent.com/codecowboydotio/swapi-json-server/main/Dockerfile'}
```

This contains the fileurl and the textresponse as items that I can call as part of the response.


## Getting the Dockerfile
The next piece of the code takes the dockerfile url, performs an HTTP GET on the file, and isolates the first line. The purpose here is to check the **FROM** command and again, feed this to an LLM via another Langchain structured reponse and a schema. This will allow me to ask the LLM to find the latest image tag.

```Python

print("Pulling dockerfile from git....")
dockerfile = requests.get(response["fileurl"])
docker_response = dockerfile.text
first_line = docker_response.split('\n', 1)[0]
print("Original: " + first_line)

dockerfile_json_schema = {
    "title": "dockerfile",
    "description": "the dockerfile",
    "type": "object",
    "properties": {
        "textresponse": {
            "type": "string",
            "description": "The text response portion",
        },
        "dockerfile": {
            "type": "string",
            "description": "the dockerfile",
        },
    },
    "required": ["textresponse", "dockerfile"],
}
print("Sending entire dockerfile to LLM to determine latest baseimage...")
dockerfile_structured_model = model.with_structured_output(dockerfile_json_schema)
response = dockerfile_structured_model.invoke(f'Update the FROM command to be the latest baseimage version for {dockerfile.text}, return the updated dockerfile make no changes if the baseimage is already at the latest version')
```

The thing to note here is that I am explicitly asking that no changes should be made if the baseimage is already at the latest version. This is to ensure that nothing odd like the addition of the **latest** tag get added.

The other thing this prompt does is to explicitly ask to update the FROM command, and no other part of the Dockerfile.

The response below is the entire Dockerfile, not just the FROM line. This means that the entire Dockerfile is sent back. An if statement checks to make sure that the FROM line is not the same. If it is, no further action is taken. If it is not, then I call a class to commit the Dockerfile to github.

```Python

llm_docker_response = response["dockerfile"]
llm_first_line = llm_docker_response.split('\n', 1)[0]
print("Replacement: " + llm_first_line)

if (llm_first_line == first_line):
  print("Original and replacement are the same..... doing nothing")
  exit(1)
else:
  # File details
  file_path = "Dockerfile"
  file_content = response["dockerfile"]

  commit_message = "Updated Dockerfile FROM via AI"
  branch = "main"  # or "master" depending on your default branch

  try:
      # Create committer instance
      committer = GitHubCommitter(GITHUB_TOKEN, OWNER, REPO)
```

## Let's run it
The proof is in the pudding here. A run of program where no changes are to be made looks like this:


```Shell
./git-test.py
Contacting git....
Got response...
Initializing model...
Found dockerfile...
Pulling dockerfile from git....
Original: FROM docker.io/library/ubuntu:24.04
Sending entire dockerfile to LLM to determine latest baseimage...
Replacement: FROM docker.io/library/ubuntu:24.04
Original and replacement are the same..... doing nothing
```

In this case, the original and latest are at the same version. Therefore there are no changes to be made.


When I update the FROM command to be an older version of the dockerfile baseimage, then the replacement is found, and the file is automatically committed to git.

```Shell
Contacting git....
Got response...
Initializing model...
Found dockerfile...
Pulling dockerfile git....
Original: FROM docker.io/library/ubuntu:20.04
Sending entire dockerfile to LLM to determine latest baseimage...
Replacement: FROM docker.io/library/ubuntu:24.04
Committing file 'Dockerfile' to codecowboydotio/swapi-json-server...
✅ Success!
Commit SHA: XXX
File URL: https://github.com/codecowboydotio/swapi-json-server/blob/main/Dockerfile
```

The actual commit in github looks like this. You can see the commit message that says it was updated by AI. 

![Dockerfile Update in Github](/images/dockerfile-update-commit.jpg)

## Github code
Once I had my very very basic working code to give me an update by calling an LLM, I asked claude to generate some python code to perform a github update, and just pasted it into my codebase.

It worked straight away.

```Python
#!/usr/bin/python

import getpass
import os
import requests
import json
from langchain.chat_models import init_chat_model
import base64
from typing import Optional


if not os.environ.get("GIT_PAT_AI"):
  os.environ["GIT_PAT_AI"] = getpass.getpass("Enter API key for Github: ")
else:
  GIT_TOKEN=os.environ["GIT_PAT_AI"]

if not os.environ.get("ANTHROPIC_API_KEY"):
  os.environ["ANTHROPIC_API_KEY"] = getpass.getpass("Enter API key for Anthropic: ")


GITHUB_TOKEN = GIT_TOKEN # Create at https://github.com/settings/tokens
OWNER = "codecowboydotio"  # Your GitHub username or organization
REPO = "swapi-json-server"  # Repository name

class GitHubCommitter:
    def __init__(self, token: str, owner: str, repo: str):
        """
        Initialize GitHub committer.

        Args:
            token: GitHub personal access token
            owner: Repository owner (username or organization)
            repo: Repository name
        """
        self.token = token
        self.owner = owner
        self.repo = repo
        self.base_url = "https://api.github.com"
        self.headers = {
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github.v3+json",
            "Content-Type": "application/json"
        }

    def get_file_sha(self, file_path: str, branch: str = "main") -> Optional[str]:
        """Get the SHA of an existing file (needed for updates)."""
        url = f"{self.base_url}/repos/{self.owner}/{self.repo}/contents/{file_path}"
        params = {"ref": branch}

        response = requests.get(url, headers=self.headers, params=params)

        if response.status_code == 200:
            return response.json()["sha"]
        elif response.status_code == 404:
            return None  # File doesn't exist
        else:
            response.raise_for_status()

    def commit_file(self, file_path: str, content: str, commit_message: str,
                   branch: str = "main", update_existing: bool = True) -> dict:
        """
        Create or update a file and commit it.

        Args:
            file_path: Path to the file in the repository
            content: File content as string
            commit_message: Commit message
            branch: Branch to commit to
            update_existing: Whether to update if file exists

        Returns:
            API response as dictionary
        """
        url = f"{self.base_url}/repos/{self.owner}/{self.repo}/contents/{file_path}"

        # Encode content to base64
        content_encoded = base64.b64encode(content.encode('utf-8')).decode('utf-8')

        # Prepare the payload
        payload = {
            "message": commit_message,
            "content": content_encoded,
            "branch": branch
        }

        # Check if file exists and get its SHA if it does
        if update_existing:
            existing_sha = self.get_file_sha(file_path, branch)
            if existing_sha:
                payload["sha"] = existing_sha

        # Make the request
        response = requests.put(url, headers=self.headers, data=json.dumps(payload))

        if response.status_code in [200, 201]:
            return response.json()
        else:
            response.raise_for_status()



url='https://api.github.com/repos/codecowboydotio/swapi-json-server/contents/'

headers = {
  "Accept": "application/vnd.github+json",
  "X-GitHub-Api-Version": "2022-11-28",
  "Authorization": f'Bearer {GIT_TOKEN}'
}

query_answer = requests.get(url, headers=headers)
query_response_code = query_answer.status_code
if query_response_code  == 204:
  print("Please check github actions for the status of the deployment")
else:
  print("Contacting git....")
  print("Got response...")
 # print(query_answer.text)


model = init_chat_model("claude-opus-4-1-20250805", model_provider="anthropic")
#model = init_chat_model("claude-3-5-sonnet-latest", model_provider="anthropic")

dockerfiles_json_schema = {
    "title": "dockerfile",
    "description": "Format of dockerfile links",
    "type": "object",
    "properties": {
        "textresponse": {
            "type": "string",
            "description": "The text response portion",
        },
        "fileurl": {
            "type": "string",
            "description": "The actual url of the file",
        },
    },
    "required": ["textresponse", "fileurl"],
}

structured_model = model.with_structured_output(dockerfiles_json_schema)
print("Initializing model...")
try:
  response = structured_model.invoke(f'find dockerfiles in array {query_answer.text} return only value download_url')
except Exception as e:
  print(f"LLM error: {e}")
  exit(1)
print("Found dockerfile...")
#print(response)

print(f"Pulling dockerfile {response["fileurl"]} from git....")
dockerfile = requests.get(response["fileurl"])
docker_response = dockerfile.text
first_line = docker_response.split('\n', 1)[0]
print("Original: " + first_line)

dockerfile_json_schema = {
    "title": "dockerfile",
    "description": "the dockerfile",
    "type": "object",
    "properties": {
        "textresponse": {
            "type": "string",
            "description": "The text response portion",
        },
        "dockerfile": {
            "type": "string",
            "description": "the dockerfile",
        },
    },
    "required": ["textresponse", "dockerfile"],
}
print("Sending entire dockerfile to LLM to determine latest baseimage...")
dockerfile_structured_model = model.with_structured_output(dockerfile_json_schema)
#response = dockerfile_structured_model.invoke(f'Update the FROM command to be the latest baseimage version for {dockerfile.text}, return the updated dockerfile')
response = dockerfile_structured_model.invoke(f'Update the FROM command to be the latest baseimage version for {dockerfile.text}, return the updated dockerfile make no changes if the baseimage is already at the latest version')
#print("===========")
#print(response["dockerfile"])


llm_docker_response = response["dockerfile"]
llm_first_line = llm_docker_response.split('\n', 1)[0]
print("Replacement: " + llm_first_line)

if (llm_first_line == first_line):
  print("Original and replacement are the same..... doing nothing")
  exit(1)
else:
  # File details
  file_path = "Dockerfile"
  file_content = response["dockerfile"]
  #file_content = """Hello, World!
  #This is a test file created via GitHub API.
  #Current timestamp: """ + str(requests.get("http://worldtimeapi.org/api/timezone/Australia/Melbourne").json().get("datetime", "unknown"))

  commit_message = "Updated Dockerfile FROM via AI"
  branch = "main"  # or "master" depending on your default branch

  try:
      # Create committer instance
      committer = GitHubCommitter(GITHUB_TOKEN, OWNER, REPO)

      # Commit the file
      print(f"Committing file '{file_path}' to {OWNER}/{REPO}...")
      result = committer.commit_file(
          file_path=file_path,
          content=file_content,
          commit_message=commit_message,
          branch=branch
      )

      print("✅ Success!")
      print(f"Commit SHA: {result['commit']['sha']}")
      print(f"File URL: {result['content']['html_url']}")
  except requests.exceptions.HTTPError as e:
      print(f"❌ HTTP Error: {e}")
      print(f"Response: {e.response.text}")
  except Exception as e:
      print(f"❌ Error: {e}")
```

## Conclusion
This is still really really nasty code. It works, but it's a single shot procedural nightmare. What comes next is even more exciting where I turn this into a complete working service, that can be run on any repository. The way that I do this is even more fascinating than the original exploration.

