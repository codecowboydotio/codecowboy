+++
title = "Github dockerfile service using AI - Part 2"
date = "2025-11-26"
aliases = ["ai"]
tags = ["ai", "dev", "claude"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Intro
I have been fooling around a lot with ai recently, and I thought I would write something about what I've been doing. There are a few things that I've been doing, and they're all fascinating. 

- Writing average looking code using minimum viable prompts.
- Getting this average looking code refactored to be a robust service.
- Generating API documentation - and having Claude fix this code.

This is part two of a small series that I have created to walk through the process I went through to get decent code.

## A crazy idea - update my dockerfiles
I had a crazy idea. I thought to myself, let's write something that will go through my git repos and automagially update my dockerfiles so that the dockerfile uses a fixed but more recent version of the base image.

In my first post, I looked at the codebase, which frankly was very average, but worked. I thought to myself "I wonder how much of a refactor it would take to get a real service working".

As it turns out.... not very much.

## Refactor my code ... please
I have been using Claude (not Claude code) for a little while now, and thought I might try to get it to refactor my **terrible** code to be well, better.

I use a simple prompt

```Shell
refactor the following code to make it more robust
```

Then I pasted in my awful code. 

![Code refactor 1](/images/code-refactor-1.jpg)

What came out was generally pretty good, however, Claude initially generated another script that I had to run that was relatively static. On the plus side, my code was now object oriented, less procedural, and had a lot of features that I didn't have originally.

![Code refactor 1](/images/refactor-1-improvements.jpg)

The biggest change here was that the code that was generated was robust and was object oriented. An example is below - the environment manager class that manages environment variables. 

This is good practice, and is not something that I would have done myself.

```Python
class EnvironmentManager:
    """Manages environment variables and API keys."""
    
    @staticmethod
    def get_github_token() -> str:
        """Get GitHub token from environment or prompt user."""
        token = os.environ.get("GIT_PAT_AI")
        if not token:
            token = getpass.getpass("Enter API key for GitHub: ")
            os.environ["GIT_PAT_AI"] = token
        return token
    
    @staticmethod
    def get_anthropic_key() -> str:
        """Get Anthropic API key from environment or prompt user."""
        key = os.environ.get("ANTHROPIC_API_KEY")
        if not key:
            key = getpass.getpass("Enter API key for Anthropic: ")
            os.environ["ANTHROPIC_API_KEY"] = key
        return key
```

{{< notice info >}}
I am being honest here about what I would and wouldn't do without the assitance of another pAIr programmer.
{{< /notice >}}

I realised I didn't want a static python script. I wanted a service.

![Code refactor 1](/images/refactor-1-how-to-use.jpg)


## I want a REST service
I used the prompt below to get Claude to refactor my code to make it more robust, and so that it had a REST based interface that I could send requests to.

```Shell
refactor the code to accept the repo as an input via a rest interface
```

**This was the piece that blew me away.**

The code that was generated was even better. I ended up with a full service that ran, and had a job manager and logger that would take all of the incoming requests and manage them.


![Code refactor 1](/images/refactor-2-outputs.jpg)

I realised that I didn't know how to use this, so I also asked for example usage using the prompt

```Shell
write a curl example of how to use the api
```

![Code refactor 1](/images/refactor-2-example-usage.jpg)

Asking for usage examples is the fastest way to get an idea of code you never wrote. It's also a great way to walk through usage.

## The REST service
The codebase is now a **BEAST**, but it worked first go!!!

### Sequence Diagram
I got claude to also generate a sequence diagram for me. It's large, but shows the interaction between the front end of the service, the github service and the user.

{{<mermaid align="left">}}
sequenceDiagram
    autonumber
    actor Client
    participant API as FastAPI Server
    participant JobMgr as Job Manager
    participant BgTask as Background Task
    participant AI as AI Analyzer<br/>(Claude)
    participant GitHub as GitHub API
    
    Note over Client,GitHub: Initial Request Phase
    Client->>+API: POST /update-dockerfile<br/>{owner, repo, github_token, ...}
    API->>API: Validate Request<br/>(Pydantic Models)
    API->>JobMgr: create_job(job_id)
    JobMgr->>JobMgr: Store job with<br/>status=PENDING
    API->>BgTask: add_task(update_dockerfile)
    API-->>-Client: 202 Accepted<br/>{job_id, status: "pending"}
    
    Note over Client,GitHub: Client Polls for Status
    Client->>+API: GET /jobs/{job_id}
    API->>JobMgr: get_job(job_id)
    JobMgr-->>API: Job details
    API-->>-Client: {status: "pending", ...}
    
    Note over BgTask,GitHub: Background Processing
    activate BgTask
    BgTask->>JobMgr: update_job(status=RUNNING)
    
    BgTask->>+GitHub: GET /repos/{owner}/{repo}/contents/
    GitHub-->>-BgTask: Repository contents array
    BgTask->>JobMgr: update_job("Retrieved repo contents")
    
    BgTask->>+AI: find_dockerfile_url(contents, path)
    AI->>AI: Search array for Dockerfile<br/>using structured output
    AI-->>-BgTask: download_url
    BgTask->>JobMgr: update_job("Found Dockerfile")
    
    BgTask->>+GitHub: GET {download_url}
    GitHub-->>-BgTask: Current Dockerfile content
    
    BgTask->>+AI: update_dockerfile_base_image(content)
    AI->>AI: Analyze FROM command<br/>Check for latest version<br/>Generate updated Dockerfile
    AI-->>-BgTask: Updated Dockerfile content
    
    BgTask->>BgTask: Compare current vs updated<br/>(first line comparison)
    
    alt No Changes Needed
        BgTask->>JobMgr: update_job(status=COMPLETED,<br/>changed=false)
    else Changes Detected
        alt Dry Run Mode
            BgTask->>JobMgr: update_job(status=COMPLETED,<br/>dry_run=true, changes shown)
        else Commit Mode
            BgTask->>+GitHub: GET /repos/.../contents/{path}<br/>?ref={branch}
            GitHub-->>-BgTask: File SHA (if exists)
            
            BgTask->>BgTask: Base64 encode<br/>updated content
            
            BgTask->>+GitHub: PUT /repos/.../contents/{path}<br/>{message, content, sha, branch}
            GitHub->>GitHub: Create commit
            GitHub-->>-BgTask: {commit: {sha}, content: {html_url}}
            
            BgTask->>JobMgr: update_job(status=COMPLETED,<br/>commit_sha, file_url)
        end
    end
    deactivate BgTask
    
    Note over Client,GitHub: Client Checks Final Status
    Client->>+API: GET /jobs/{job_id}
    API->>JobMgr: get_job(job_id)
    JobMgr-->>API: Job with status=COMPLETED
    API-->>-Client: 200 OK<br/>{status: "completed",<br/>result: {changed, commit_sha, ...}}
    
    Note over Client,GitHub: Optional: View Updated File
    Client->>+GitHub: GET {file_url}<br/>(from result)
    GitHub-->>-Client: View updated Dockerfile
{{< /mermaid >}}


# Let's run it
The proof is in the pudding here. I start the server on my laptop and I can see that it's running as a service on port 8000.

{{< notice info >}}
I placed the usage before the code walkthrough because I recognise not everyone will be interested in the code walkthrough - there is a conclusion at the end of the article.
{{< /notice >}}

## Run the Server
I start up the server and the following is displayed on my terminal (stdout).

```Shell
./git-test2.py
2025-11-27 17:18:04,515 - INFO - Initialized AI model: claude-opus-4-1-20250805
INFO:     Started server process [1966]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

## First Request - no changes
I send my first request to the service. I need to include a data section that includes my git **PAT TOKEN**. This is for access to the repo. I also include the repo owner, the repo name, the branch, the path of the file and a commit message.

The last thing is a dry run variable. It is possible to use the service in dry run mode where no changes will actually be made.

```Shell
curl -X POST "http://localhost:8000/update-dockerfile"   -H "Content-Type: application/json"   -d '{
 "owner": "codecowboydotio",
 "repo": "swapi-json-server",
 "branch": "main",
 "github_token": "GIT_PAT_TOKEN",
 "dockerfile_path": "Dockerfile",
 "commit_message": "Updated Dockerfile FROM via AI",
 "dry_run": false
}'
```

The output on the console from my shell command is as follows. You can see that a job is created. The job has a unique ID, and is queued for processing.
```Json
[
  {
    "job_id":"109b7804-61c3-4f36-8b51-ea35642b41c2",
    "status":"pending",
    "message":"Job created and queued for processing",
    "timestamp":"2025-11-27T17:19:02.302591"
  }
]
```

Once the request is made, I can see the service fetch the contents, find the file, make the call to anthropic, and finally analyse the Dockerfile.

```Shell
2025-11-27 17:19:02,305 - INFO - Fetching repository contents from https://api.github.com/repos/codecowboydotio/swapi-json-server/contents/
2025-11-27 17:19:02,831 - INFO - Downloading file from https://raw.githubusercontent.com/codecowboydotio/swapi-json-server/main/Dockerfile
2025-11-27 17:19:11,522 - INFO - HTTP Request: POST https://api.anthropic.com/v1/messages "HTTP/1.1 200 OK"
2025-11-27 17:19:11,533 - INFO - Successfully analyzed Dockerfile for base image updates
```

When I query the list of jobs using the **/jobs** endpoint, I can see that the status of the job is now completed, and there is a message that says no changes were needed.

```Shell
curl -X GET http://localhost:8000/jobs  -H "Content-Type: application/json" | jq
```

The output shows the time, the ID of the request, and a message saying no changes were needed. Importantly, thecurrent and updated FROM lines are displayed.

```Json
[
  {
    "job_id": "109b7804-61c3-4f36-8b51-ea35642b41c2",
    "status": "completed",
    "message": "No changes needed - Dockerfile is already up to date",
    "timestamp": "2025-11-27T17:19:11.533974",
    "result": {
      "changed": false,
      "current_from": "FROM docker.io/library/ubuntu:24.04",
      "updated_from": "FROM docker.io/library/ubuntu:24.04"
    },
    "error": null
  }
]
```

## Second request - perform an update
When I reset my Dockerfile to require a change and re-run the request, I see that there is a new job ID created.

```Json

{
  "job_id": "6691c4ac-e232-43fd-86f9-073440d32bac",
  "status": "pending",
  "message": "Job created and queued for processing",
  "timestamp": "2025-11-27T17:26:40.359356"
}
```

I then check my new job to see what the status is. 
```Shell
curl -X GET http://localhost:8000/jobs  -H "Content-Type: application/json" | jq
```

As I am using the **/jobs** endpoint, this lists **ALL** jobs, not an individual job. I can see my original job that had no change, and I can see my new job, which has successfully updated the dockerfile in my repository.

```Json
[
  {
    "job_id": "109b7804-61c3-4f36-8b51-ea35642b41c2",
    "status": "completed",
    "message": "No changes needed - Dockerfile is already up to date",
    "timestamp": "2025-11-27T17:19:11.533974",
    "result": {
      "changed": false,
      "current_from": "FROM docker.io/library/ubuntu:24.04",
      "updated_from": "FROM docker.io/library/ubuntu:24.04"
    },
    "error": null
  },
  {
    "job_id": "6691c4ac-e232-43fd-86f9-073440d32bac",
    "status": "completed",
    "message": "Successfully updated and committed Dockerfile",
    "timestamp": "2025-11-27T17:26:50.300875",
    "result": {
      "changed": true,
      "current_from": "FROM docker.io/library/ubuntu:20.04",
      "updated_from": "FROM docker.io/library/ubuntu:24.04",
      "dry_run": false,
      "commit_sha": "4304b404050e15293ed7d017752c252593cbc102",
      "file_url": "https://github.com/codecowboydotio/swapi-json-server/blob/main/Dockerfile"
    },
    "error": null
  }
]
```

We can see from the commits in the repo that it has indeed been updated!

![commits](/images/docker-service-commits.jpg)


# Code Walkthrough
Let's walk through the code base, and how to use it. 

The codebase is documented but did not contain an easy way to run it. For my own sanity I went and pasted in one of the example REST calls so that I could cut and paste it at any time.

```Python
#!/usr/bin/env python3
"""
Dockerfile Base Image Updater REST API

This service provides a REST API to automatically update Dockerfile base images
to their latest versions using AI analysis and commits the changes to GitHub.
"""

# curl -X POST "http://localhost:8000/update-dockerfile"   -H "Content-Type: application/json"   -d '{
#  "owner": "codecowboydotio",
#  "repo": "swapi-json-server",
#  "branch": "main",
#  "github_token": "XXX",
#  "dockerfile_path": "Dockerfile",
#  "commit_message": "Updated Dockerfile FROM via AI",
#  "dry_run": false
#}'
```

## Libraries
There are a lot more imports of libraries this time. Claude chose FastAPI, and urllib3, but kept my original langhain and requests portions of code. 

```Python
import asyncio
import base64
import json
import logging
import os
import sys
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Optional, Dict, Any, List
from enum import Enum
import uuid

import requests
from fastapi import FastAPI, HTTPException, BackgroundTasks, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, validator, field_validator
from langchain.chat_models import init_chat_model
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
```

## Logger
The logger configuration from the second iteration was kept.

```Python
# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('dockerfile_updater_api.log')
    ]
)
logger = logging.getLogger(__name__)
```

## Classes and data classes
What was interesting was the job manager class that has been introduced. This is not something that I had thought about, however, in terms of scaling this tiny program into a more robust and larger service, this is definitely something that would be needed.  Below are both the classes that were instantiated and the dataclass objects. All of the classes and data classes that have been added are good to have, and provide a clearer definition of the "what" each type of data is.

```Python
class JobStatus(str, Enum):
    """Job status enumeration."""
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class Config:
    """Configuration class for the application."""
    api_version: str = "2022-11-28"
    timeout: int = 30
    max_retries: int = 3
    ai_model: str = "claude-opus-4-1-20250805"


class DockerfileUpdateRequest(BaseModel):
    """Request model for Dockerfile update."""
    owner: str
    repo: str
    branch: str = "main"
    github_token: str
    dockerfile_path: str = "Dockerfile"
    commit_message: Optional[str] = "Updated Dockerfile FROM via AI"
    dry_run: bool = False

    @field_validator('owner', 'repo')
    def validate_not_empty(cls, v):
        if not v or not v.strip():
            raise ValueError('Field cannot be empty')
        return v.strip()

    @field_validator('github_token')
    def validate_github_token(cls, v):
        if not v or len(v) < 10:
            raise ValueError('Invalid GitHub token')
        return v


class DockerfileUpdateResponse(BaseModel):
    """Response model for Dockerfile update."""
    job_id: str
    status: JobStatus
    message: str
    timestamp: datetime


class JobStatusResponse(BaseModel):
    """Response model for job status."""
    job_id: str
    status: JobStatus
    message: str
    timestamp: datetime
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None


@dataclass
class JobResult:
    """Job result data class."""
    job_id: str
    status: JobStatus
    message: str
    timestamp: datetime
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None


class DockerfileUpdaterError(Exception):
    """Custom exception for Dockerfile updater errors."""
    pass
```

## HTTP Client class
The HTTP client class is client that that is used by the service to update the dockerfile. This is performed via a REST API call from the client to the github API. Claude has correctly added session handling, error handling and exception code.

```Python
class HTTPClient:
    """HTTP client with retry logic and proper error handling."""

    def __init__(self, timeout: int = 30, max_retries: int = 3):
        self.session = requests.Session()
        self.timeout = timeout

        # Configure retry strategy
        retry_strategy = Retry(
            total=max_retries,
            status_forcelist=[429, 500, 502, 503, 504],
            allowed_methods=["HEAD", "GET", "OPTIONS", "PUT"],
            backoff_factor=1
        )

        adapter = HTTPAdapter(max_retries=retry_strategy)
        self.session.mount("http://", adapter)
        self.session.mount("https://", adapter)

    def get(self, url: str, **kwargs) -> requests.Response:
        """Make GET request with error handling."""
        try:
            kwargs.setdefault('timeout', self.timeout)
            response = self.session.get(url, **kwargs)
            response.raise_for_status()
            return response
        except requests.exceptions.RequestException as e:
            logger.error(f"GET request failed for {url}: {e}")
            raise DockerfileUpdaterError(f"HTTP GET failed: {e}")

    def put(self, url: str, **kwargs) -> requests.Response:
        """Make PUT request with error handling."""
        try:
            kwargs.setdefault('timeout', self.timeout)
            response = self.session.put(url, **kwargs)
            response.raise_for_status()
            return response
        except requests.exceptions.RequestException as e:
            logger.error(f"PUT request failed for {url}: {e}")
            raise DockerfileUpdaterError(f"HTTP PUT failed: {e}")
```

## GithubAPI Class
This class is specifically for dealing with the Github API. This is slightly different from the HTTP client that provides the **transport**. 

There are four different methods that are to:
- Get the repo contents
- Get the file contents
- Get the file SHA
- Commit the file (with return and error codes)

All of this code is relatively robust, object oriented, and has appropriate error handling. 

From where we started with a static procedural script, to get to here is quite amazing.

```Python
class GitHubAPI:
    """GitHub API client with proper error handling."""

    def __init__(self, token: str, owner: str, repo: str, config: Config, http_client: HTTPClient):
        self.token = token
        self.owner = owner
        self.repo = repo
        self.config = config
        self.http_client = http_client
        self.base_url = "https://api.github.com"
        self.headers = {
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": config.api_version,
            "Content-Type": "application/json"
        }

    def get_repository_contents(self) -> List[Dict[str, Any]]:
        """Get repository contents."""
        url = f"{self.base_url}/repos/{self.owner}/{self.repo}/contents/"
        logger.info(f"Fetching repository contents from {url}")

        response = self.http_client.get(url, headers=self.headers)

        if response.status_code == 204:
            raise DockerfileUpdaterError("Repository is empty or deployment in progress")

        try:
            return response.json()
        except json.JSONDecodeError as e:
            logger.error(f"Failed to parse JSON response: {e}")
            raise DockerfileUpdaterError("Invalid JSON response from GitHub API")

    def get_file_content(self, download_url: str) -> str:
        """Download file content from GitHub."""
        logger.info(f"Downloading file from {download_url}")
        response = self.http_client.get(download_url)
        return response.text

    def get_file_sha(self, file_path: str, branch: str) -> Optional[str]:
        """Get the SHA of an existing file."""
        url = f"{self.base_url}/repos/{self.owner}/{self.repo}/contents/{file_path}"
        params = {"ref": branch}

        try:
            response = self.http_client.get(url, headers=self.headers, params=params)
            return response.json()["sha"]
        except DockerfileUpdaterError:
            # File doesn't exist
            return None

    def commit_file(self, file_path: str, content: str, commit_message: str, branch: str) -> Dict[str, Any]:
        """Commit file to repository."""
        url = f"{self.base_url}/repos/{self.owner}/{self.repo}/contents/{file_path}"

        # Encode content to base64
        try:
            content_encoded = base64.b64encode(content.encode('utf-8')).decode('utf-8')
        except UnicodeEncodeError as e:
            raise DockerfileUpdaterError(f"Failed to encode file content: {e}")

        payload = {
            "message": commit_message,
            "content": content_encoded,
            "branch": branch
        }

        # Get existing file SHA if file exists
        existing_sha = self.get_file_sha(file_path, branch)
        if existing_sha:
            payload["sha"] = existing_sha
            logger.info(f"Updating existing file {file_path}")
        else:
            logger.info(f"Creating new file {file_path}")

        response = self.http_client.put(url, headers=self.headers, data=json.dumps(payload))
        return response.json()
```

## AI Class
This class is the class that makes the called to the AI model. There are two methods here: 
- Find the dockerfile url in the repo
- Update the dockerfile base image

The two interesting things here are that clause has not altered the prompts, json schemas or changed the use the langchain.

Claude has continued to provide robust error handling and has turned my code into a class.

```Python
class AIAnalyzer:
    """AI-powered Dockerfile analyzer."""

    def __init__(self, model_name: str = "claude-opus-4-1-20250805"):
        try:
            # Ensure Anthropic API key is available
            if not os.environ.get("ANTHROPIC_API_KEY"):
                raise DockerfileUpdaterError("ANTHROPIC_API_KEY environment variable is required")

            self.model = init_chat_model(model_name, model_provider="anthropic")
            logger.info(f"Initialized AI model: {model_name}")
        except Exception as e:
            logger.error(f"Failed to initialize AI model: {e}")
            raise DockerfileUpdaterError(f"AI model initialization failed: {e}")

    def find_dockerfile_url(self, repository_contents: List[Dict[str, Any]], dockerfile_path: str) -> str:
        """Find Dockerfile URL in repository contents using AI."""
        # First try to find the file directly by name
        for item in repository_contents:
            if item.get("name") == dockerfile_path and item.get("type") == "file":
                return item.get("download_url")

        # If not found directly, use AI to search
        dockerfile_schema = {
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

        structured_model = self.model.with_structured_output(dockerfile_schema)

        try:
            response = structured_model.invoke(
                f'Find dockerfile named "{dockerfile_path}" in array {repository_contents} return only value do
wnload_url'
            )
            logger.info("Successfully found Dockerfile URL using AI")
            return response["fileurl"]
        except Exception as e:
            logger.error(f"AI analysis failed for finding Dockerfile: {e}")
            raise DockerfileUpdaterError(f"Failed to find Dockerfile '{dockerfile_path}': {e}")

    def update_dockerfile_base_image(self, dockerfile_content: str) -> str:
        """Update Dockerfile base image using AI."""
        dockerfile_schema = {
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

        structured_model = self.model.with_structured_output(dockerfile_schema)

        try:
            prompt = (
                f'Update the FROM command to be the latest baseimage version for {dockerfile_content}, '
                'return the updated dockerfile. Make no changes if the baseimage is already at the latest versi
on'
            )
            response = structured_model.invoke(prompt)
            logger.info("Successfully analyzed Dockerfile for base image updates")
            return response["dockerfile"]
        except Exception as e:
            logger.error(f"AI analysis failed for updating Dockerfile: {e}")
            raise DockerfileUpdaterError(f"Failed to update Dockerfile: {e}")
```

## Job Manager class
This is the one thing that I just didn't even think of. Given that I have an API calling out to another API, and this is likely to be asynchronous, job handling is required. Again, the methods here are impressive. 

- Create job
- Update job
- Get job
- List all jobs

This includes having multiple status types for jobs, including **PENDING** and **COMPLETED**.

```Python
class JobManager:
    """Manages background jobs."""

    def __init__(self):
        self.jobs: Dict[str, JobResult] = {}

    def create_job(self, job_id: str) -> None:
        """Create a new job."""
        self.jobs[job_id] = JobResult(
            job_id=job_id,
            status=JobStatus.PENDING,
            message="Job created",
            timestamp=datetime.now()
        )

    def update_job(self, job_id: str, status: JobStatus, message: str,
                   result: Optional[Dict[str, Any]] = None, error: Optional[str] = None) -> None:
        """Update job status."""
        if job_id in self.jobs:
            self.jobs[job_id].status = status
            self.jobs[job_id].message = message
            self.jobs[job_id].timestamp = datetime.now()
            self.jobs[job_id].result = result
            self.jobs[job_id].error = error

    def get_job(self, job_id: str) -> Optional[JobResult]:
        """Get job by ID."""
        return self.jobs.get(job_id)

    def list_jobs(self) -> List[JobResult]:
        """List all jobs."""
        return list(self.jobs.values())
```

## Dockerfile Update Service
This is a wrapper class that pulls together using the HTTP client above, and the AI analyser class to make the changes to the dockerfile. This class also performs implementation of the job handling code.

```Python
class DockerfileUpdaterService:
    """Main service class."""

    def __init__(self, config: Config):
        self.config = config
        self.http_client = HTTPClient(config.timeout, config.max_retries)
        self.ai_analyzer = AIAnalyzer(config.ai_model)
        self.job_manager = JobManager()

    async def update_dockerfile(self, job_id: str, request: DockerfileUpdateRequest) -> None:
        """Update Dockerfile in background."""
        try:
            self.job_manager.update_job(job_id, JobStatus.RUNNING, "Starting Dockerfile update process")

            # Initialize GitHub API client
            github_api = GitHubAPI(
                request.github_token,
                request.owner,
                request.repo,
                self.config,
                self.http_client
            )

            # Get repository contents
            contents = github_api.get_repository_contents()
            self.job_manager.update_job(job_id, JobStatus.RUNNING, "Retrieved repository contents")

            # Find Dockerfile URL
            dockerfile_url = self.ai_analyzer.find_dockerfile_url(contents, request.dockerfile_path)
            self.job_manager.update_job(job_id, JobStatus.RUNNING, f"Found Dockerfile at: {dockerfile_url}")

            # Download current Dockerfile
            current_dockerfile = github_api.get_file_content(dockerfile_url)
            current_first_line = current_dockerfile.split('\n', 1)[0] if current_dockerfile else ""

            # Update Dockerfile using AI
            updated_dockerfile = self.ai_analyzer.update_dockerfile_base_image(current_dockerfile)
            updated_first_line = updated_dockerfile.split('\n', 1)[0] if updated_dockerfile else ""

            # Check if changes are needed
            if current_first_line == updated_first_line:
                self.job_manager.update_job(
                    job_id,
                    JobStatus.COMPLETED,
                    "No changes needed - Dockerfile is already up to date",
                    result={
                        "changed": False,
                        "current_from": current_first_line,
                        "updated_from": updated_first_line
                    }
                )
                return

            result_data = {
                "changed": True,
                "current_from": current_first_line,
                "updated_from": updated_first_line,
                "dry_run": request.dry_run
            }

            # Commit updated Dockerfile (unless dry run)
            if not request.dry_run:
                commit_result = github_api.commit_file(
                    file_path=request.dockerfile_path,
                    content=updated_dockerfile,
                    commit_message=request.commit_message,
                    branch=request.branch
                )

                result_data.update({
                    "commit_sha": commit_result['commit']['sha'],
                    "file_url": commit_result['content']['html_url']
                })

                self.job_manager.update_job(
                    job_id,
                    JobStatus.COMPLETED,
                    "Successfully updated and committed Dockerfile",
                    result=result_data
                )
            else:
                self.job_manager.update_job(
                    job_id,
                    JobStatus.COMPLETED,
                    "Dry run completed - changes detected but not committed",
                    result=result_data
                )

        except Exception as e:
            error_message = str(e)
            logger.error(f"Job {job_id} failed: {error_message}")
            self.job_manager.update_job(
                job_id,
                JobStatus.FAILED,
                "Job failed with error",
                error=error_message
            )
```

## API
This portion of the code implements the API and reverse proxy / middleware layer of my API. This uses the config class and passes that to the updater service.

```Python
# Initialize FastAPI app
app = FastAPI(
    title="Dockerfile Updater API",
    description="REST API for updating Dockerfile base images using AI",
    version="1.0.0"
)

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
config = Config()
service = DockerfileUpdaterService(config)

# Security
security = HTTPBearer(auto_error=False)


async def get_api_key(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Validate API key (optional)."""
    # You can implement API key validation here if needed
    return credentials
```

## Routes
The code below implements a number of routes. These are:
- /health **GET**
- /update-dockerfile **POST**
- /jobs **GET**
- /jobs/{job} **GET**
- /jobs/{job} **DELETE**

Each of these routes can be called and with the appropriate headers is the primary way to interact with the service.

```Python
@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {"status": "healthy", "timestamp": datetime.now()}


@app.post("/update-dockerfile", response_model=DockerfileUpdateResponse)
async def update_dockerfile(
    request: DockerfileUpdateRequest,
    background_tasks: BackgroundTasks,
    credentials: HTTPAuthorizationCredentials = Depends(get_api_key)
):
    """Update Dockerfile base image."""
    try:
        # Generate job ID
        job_id = str(uuid.uuid4())

        # Create job
        service.job_manager.create_job(job_id)

        # Start background task
        background_tasks.add_task(service.update_dockerfile, job_id, request)

        return DockerfileUpdateResponse(
            job_id=job_id,
            status=JobStatus.PENDING,
            message="Job created and queued for processing",
            timestamp=datetime.now()
        )

    except Exception as e:
        logger.error(f"Failed to create update job: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/jobs/{job_id}", response_model=JobStatusResponse)
async def get_job_status(job_id: str):
    """Get job status by ID."""
    job = service.job_manager.get_job(job_id)

    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    return JobStatusResponse(**asdict(job))


@app.get("/jobs", response_model=List[JobStatusResponse])
async def list_jobs():
    """List all jobs."""
    jobs = service.job_manager.list_jobs()
    return [JobStatusResponse(**asdict(job)) for job in jobs]


@app.delete("/jobs/{job_id}")
async def delete_job(job_id: str):
    """Delete job by ID."""
    if job_id not in service.job_manager.jobs:
        raise HTTPException(status_code=404, detail="Job not found")

    del service.job_manager.jobs[job_id]
    return {"message": "Job deleted successfully"}
```

## Main function
The main function is very standard. It starts a uvicorn server on port 8000 that hosts my code.
```Python
if __name__ == "__main__":
    import uvicorn

    # Ensure required environment variables
    if not os.environ.get("ANTHROPIC_API_KEY"):
        logger.error("ANTHROPIC_API_KEY environment variable is required")
        sys.exit(1)

    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="info")
```

# Summary
This has been a long journey so far for the reader, but for me as a developer it's been very quick. It took me only a few minutes to refactor the code into a service using Claude. I have a robust service now that allows me to get Dockerfiles updated without too much difficulty. I just need to feed in the right paramters and away it goes, and performs the checks on my behalf, does the analysis, and updates the repository.

This is a neat way to keep your Dockerfiles updated and your repos clean!

