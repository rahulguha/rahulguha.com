---
date: '2025-05-19T18:15:07-06:00'
draft: false
title: 'Dockerized SSE MCP Servers and Clients - part 1'
summary: Dockerize SSE MCP Server, Build a Multi Protocol Client - deploy locally
tags: [ "docker", "sse", "tech", "LLM", "MCP", "RAG", "Contextual RAG"]
author: ['Rahul Guha']
draft: false
aliases: [/tech/sse-mcp-server-client-docker]
Mermaid: true
weight: 3
---

### Github Repos for reference

- <a href="https://github.com/rahulguha/mcp/tree/main" target="_blank">API, Servers and Client</a>

> :bulb: <font color="red">All tools and models used here are exchangeable with other models and libraries.</font>

#### Context Diagram

<img src="/tech/img/sse-mcp-client-server-docker.jpg/" alt="SSE MCP"  height="700">

### Purpose of this post:

Exploring further from <a href="https://rahulguha.com/tech/mcp-servers-school-rag/" target="_blank">my previous post</a>, I wanted to check out how MCP can be made more accessible over established HTTP protocol so that other security mechanism can be plugged in to that.
<a href="https://www.anthropic.com/news/model-context-protocol" target="_blank">Model Context Protocol</a> supports STDIO (local tool execution) and SSE and Streamable HTTP (remote execution). Below is a high level comparison:

|                                       | **STDIO**                                    | **SSE/Streamable HTTP**                                                                |
| ------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------- |
| _Execution<br>Compute_                | Process-based - tool runs on a local process | Network based - Typically exposed over HTTP layer                                      |
| _Implementation_                      | Easy                                         | Can be complex                                                                         |
| _Resilience <br>Production readiness_ | Basic                                        | Complex but closer towards enterprise grade <br>using cloud deployment and autoscaling |
| _State Management_                    | Complex to implement                         | SSE keeps server to client state                                                       |
| _Examples_                            | local tools, clis                            | web based http apis                                                                    |

### Use case

> <font color="blue"> In this post, we will try to help a highschooler who is trying to gather information on his target schools and majors.
> </font>

### Components

While we are continuing with previous example, this time we will implement the followings:

- Dockerize the APIs
- Create SSE MCP servers using <a href="https://gofastmcp.com/getting-started/welcome" target="_blank"> FASTMCP python sdk </a>
- Dockerize the MCP Server
- Deploy all three docker containers in local laptop.
- Build an MCP client (as Claude desktop doesn't support SSE servers yet) using <a href="https://github.com/langchain-ai/langchain-mcp-adapters" target="_blank"> Langchain MCP Adapters </a>

> <font color="blue"> In next post we will deploy them on cloud and hook up OAuth in that.
> </font>

#### Project Structure

```
├── api
│   ├── **pycache**
│   ├── README.md
│   ├── school_rag_api
│   └── school-list
├── Architecture
│   └── architecture.drawio
├── client
│   ├── main.py
│   ├── mcp_client.py
│   ├── pyproject.toml
│   ├── README.md
│   └── uv.lock
└── servers
└── school_server
```

Just to demonstrate flexibility, my `school-list` API is a nestJS project while `school_rag_api` is a FASTAPI one.
`servers` folder holds the MCP server and `client` has the code for mcp client.

#### SSE MCP Servers

Few key differences to note while building SSE MCP servers:

- The function needs to be an async not a regular function
- I am using FastMCP sdk where we define which port this server will listen to (this case 3501)
- Finally when running MCP server, indicate which transport (stdio, sse or streamable-http) to run the mcp server

```
# Initialize FastMCP server
mcp = FastMCP(
    name="School List",
    port=3501,
    on_duplicate_tools="error" # Set duplicate handling
    )

# MCP tool to fetch school data
@mcp.tool()
async def get_schools() -> dict:
    """This tool fetches school data from the schools API."""
    async with httpx.AsyncClient() as client:
        print("calling api")
        school_list_api_endpoint = os.getenv("SCHOOL_LIST_API_ENDPOINT", "http://school-list-api:3100/schools")
        response = await client.get(f"{school_list_api_endpoint}")
        response.raise_for_status()
        return response.json()
.....
.....
if __name__ == "__main__":
    # mcp.run(transport="streamable-http")
    mcp.run(transport="sse")
```

#### Containerization

Dockerization is quite standard practice for these components. Few key issues that to be noted:

- Ensure proper installation of dependencies as Python 13.2 is quite new ` RUN pip install --no-cache-dir uvicorn fastapi httpx mcp-server python-dotenv` - this gave me some grief

```
# Use a slim Python 3.13 base image
FROM python:3.13-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install uv
RUN pip install uv

# Copy uv configuration files
COPY pyproject.toml uv.lock ./

# Install dependencies with uv
RUN pip install --no-cache-dir uvicorn fastapi httpx mcp-server python-dotenv
RUN uv sync --frozen
# Copy the entire project, including data, website_content, and school_list.py
COPY . .
# COPY ../servers/school_server/school_list.py ./school_list.py
# COPY ../servers/school_server/.env ./.env

# Expose ports 3200 (FastAPI) and 3501 (MCP SSE)
EXPOSE 3501

# Run the MCP SSE server
CMD ["uv", "run", "python", "school_list.py"]

```

- To run this locally on localhost, I needed to create a local docker network and attach each of docker images in that. This is required to ensure MCP server can access the APIs.
  > <font color="blue"> This will be replaced by cloud deployment design (e.g. K8S egress/ingress etc later)
  > </font>

```
docker network create school-network
docker run --rm -d \
  --name school-list-api \
  -p 3100:3100 \
  --network school-network \
  school-list-api

docker run --rm -d \
  --name school-rag-api \
  -p 3200:3200 \
  --network school-network \
  school-rag-api

docker run --rm -d \
  --name school-mcp-server \
  -p 3501:3501 \
  --network school-network \
  -e SCHOOL_LIST_API_ENDPOINT=http://school-list-api:3100/schools \
  -e SCHOOL_RAG_API_ENDPOINT=http://school-rag-api:3200/chat \
  school-mcp-server

```

#### Custom MCP Client

Decided to use Langchain multi MCP adapter for creating MCP client.

```
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
...
...
url = os.getenv("SSE_SERVER_ENDPOINT")
    client = MultiServerMCPClient(
        {
            "school-server-sse": {
                "url":url,
                # "transport": "streamable_http",
                "transport": "sse",

            }
        })
    # tools = []
    tools = await client.get_tools()
    pprint.pprint ( tools, indent=3)
    agent = create_react_agent(llm, tools)
    result = await agent.ainvoke(
        {"messages": "Tell me about research opportunities for aersopace engineering"}
    )

    print(result["messages"][-1].content)

```

Few key call outs:

- OpenAI is used here - but it can be replaced by other LLMs easily as you see in the code langchain client actually abstracts the LLMs from execution
- Unlike Claude Desktop, this doesn't force user confirmation before executing tools.
- Below is a quick dump of tools object as get_tools is called.

```
[  StructuredTool(name='get_schools', description='This tool fetches school data from the schools API.', args_schema={'properties': {}, 'title': 'get_schoolsArguments', 'type': 'object'}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x105432f20>),
   StructuredTool(name='get_school_rag', description='\nThis tool executes the RAG pipeline and gets relevant text which will be sent to LLM.\nUse this tool for any school related query around cost, course curriculum, accomodation, location, admission etc\n', args_schema={'properties': {'query': {'title': 'Query', 'type': 'string'}}, 'required': ['query'], 'title': 'get_school_ragArguments', 'type': 'object'}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x1054cb740>)]
```

> <font color="red"> Like it or not, Langchain and MCP sdk and apis keep on changing. So it's better to always keep the documentations handy.
>
> <a href="https://gofastmcp.com/getting-started/welcome" target="_blank">FASTMCP Documentation</a> </font>
>
> <a href="https://github.com/langchain-ai/langchain-mcp-adapters" target="_blank"> Langchain MCP Adapter Documentation</a> </font>
>
> <a href="https://modelcontextprotocol.io/introduction" target="_blank"> MCP Official Documentation</a> </font>

## Architecture

### Chat Flow

{{< mermaid >}}
sequenceDiagram %% diagram
autonumber
participant u as user
participant mcp_client as MCP Client
participant ask_rag as RAG MCP Server
participant get_school as School List MCP Server
participant rag as RAG Pipeline
participant school_list as School Info API
participant a as Anthropic

     u->>mcp_client: question
    mcp_client->>ask_rag: Get chunks with context
    ask_rag->>rag: Semantic Search
    mcp_client->>get_school: School List
    get_school->>school_list: Invoke school list API
    mcp_client->>a: use LLM to get the final response
    mcp_client ->> u: Response

{{< /mermaid >}}
