---
date: '2025-04-06T18:15:07-06:00'
draft: false
title: 'MCP Server exposing Contextual RAG and API'
summary: Build two MCP servers exposing contextual RAG and local API.
tags: [ "tech", "LLM", "MCP", "RAG", "Contextual RAG"]
author: ['Rahul Guha']
draft: false
aliases: [/tech/mcp-server-rag-api]
Mermaid: true
weight: 3
---

### Github Repos for reference

- <a href="https://github.com/rahulguha/school_list" target="_blank">School List API</a>
- <a href="https://github.com/rahulguha/school_rag" target="_blank">School RAG</a>
- <a href="https://github.com/rahulguha/school_mcp" target="_blank">MCP Servers</a>

> :bulb: <font color="red">All tools and models used here are exchangeable with other models and libraries.</font>

### Purpose of this post:

MCP - <a href="https://www.anthropic.com/news/model-context-protocol" target="_blank">Model Context Protocol</a> is relatively new entrant in the world of AI Agents but acquired people's mindset because of it's initiative to standardize the interfaces. It became even more important as Anthropic competitors (like Open AI) announced support towards MCP and multiple other tools (like Cursor IDE) immediately embraced the concept by becoming an MCP Client.

I think the power of MCP lays in connecting the definitiveness of APIs (typically exposing structured data) with unstructured LLM chat (like in this case Claude Desktop) to get a much powerful response.

### Use case

> <font color="blue"> In this post, we will try to help a highschooler who is trying to gather information on his target schools and majors.
> </font>

### Components

- I created a json file (in place of a real database) that captures various important urls of schools and tag each of the urls with meningful categories. Example below -

```
{
    "school": "TAMU",
    "urls": [
      {
        "url": "https://www.tamu.edu/",
        "tags": ["homepage", "overview"]
      },
      {
        "url": "https://www.tamu.edu/admissions/how-to-apply/apply-as-freshman.html",
        "tags": ["application", "timeline"]
      },
      {
        "url": "https://www.tamu.edu/academics/colleges-schools/college-of-engineering.html",
        "tags": ["engineering", "overview"]
```

- I built an API that exposes this data over HTTP which will be used for one of the MCP servers (get_schools)
- I crawled all the urls and created a Contextual RAG pipeline which also exposes an API that will be used by the other MCP servers (ask_rag).

  > :bulb: <font color="red">In case needed feel free to refer to my post on <a href="https://rahulguha.com/tech/building-a-contextual-rag/" target="_blank">Contextual RAG</a> .</font>

- I created MCP Servers in a separate python project and configured those to Claude Desktop (a MCP client) so that claude can actually use the local tools to query first before sending to anthropic for final response.

#### Context Diagram

<img src="/tech/img/school_mcp.png" alt="School MCP"  height="700">

### How MCP Works and How this was built?

There are two parts of MCP

- <b>MCP Server:</b> This is the functional piece that the application will expose. There is a list of reference servers available in <a href="https://github.com/modelcontextprotocol/servers" target="_blank">Anthropic MCP Server Repo</a>. However this is the place where application developers will expose their data capabilities. In this application I built two servers that essentially exposes my data (for illustration purpose)
  - <b>Get School Info:</b>Schools Url and categorization information as a NestJS API
  - <b>Ask RAG:</b>Contextual RAG pipeline exposed as API that already has following capabilities:
    - Pre process the school related urls as follows:
      - Chunk them
      - Generate context by sending the Chunk with whole document to LLM (in this case anthropic)
      - Store locally
    - Load the chunks in Vector DB (in this case VoyagerAI)
    - Use rerank to narrow down and rank the chunks after similarity/sementic search (use cohere in this case)
    - Generate the response and source Urls by sending chunks to LLM (anthropic in this case)
    - Finally expose the whole chat capability in an API interface
- <b>MCP Client: </b> I use Claude desktop in this example (and tested with Claude Code and Cursor). However it can be used in any client tool that follows MCP Client standards. Configuration is quite simple as below:
  ```
        {
        "mcpServers": {
            "get_schools": {
            "command": "FULL/PATH/TO/UV",
            "args": [
                "--directory",
                "/FULL/PATH/TO/LOCAL/PROJECT",
                "run", # command to run
                "t_schools.py" # file to run
            ]
            },
            "ask_rag": {
            "command": "/FULL/PATH/TO/LOCAL/PROJECT",
            "args": [
                "--directory",
                "/FULL/PATH/TO/LOCAL/PROJECT",
                "run",
                "chat_api.py"
            ]
            }
        }
        }
  ```
  > :bulb: <font color="red">You may need to provide full path for the commands to run - relative path may not work</font>

Once properly setup these tools start showing up in Claude desktop (you will have to quite and restart claude desktop). From that point onwards as you ask questions to Claude, it will look into tools first, ask you permission before using the tool and then finally generate a comprehensive answer using LLM.
As seen in the screenshot, claude desktop first uses ask_rag server to get chunks and then also uses get_schools. Then it sends to LLM but also negotiates MCP servers multiple times.
<img src="/tech/img/claude-desktop-screen-shot.png" alt="Claude desktop screen shot"  height="500">

Finally it actually gives a much more comprehensive response that makes an effort to use all fragments of information coming out of the tools to summarize in a usable form.
<img src="/tech/img/claude-desktop-screen-shot-2.png" alt="Claude desktop screen shot"  height="500">

> :bulb: <font color="red">Important thing to note here is - the information is coming from the RAG not from LLM. This reduces/avoids hallucination significantly</font>

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
