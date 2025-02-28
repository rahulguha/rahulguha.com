---
date: '2025-02-28T18:15:07-06:00'
draft: false
title: 'Building a RAG System - Podcast Monitor Part 2'
summary: RAG - adding context to the LLM Query.
tags: [ "tech", "LLM"]
author: ['Rahul Guha']
draft: false
aliases: [/tech/building-a-rag-system]
Mermaid: true
weight: 3
---

<a href="https://github.com/rahulguha/hybrid-rag" target="_blank">repo link</a>
<img src="/tech/vector-rag-Page-2.png" alt="Architecture Layers"  height="500">

### What is RAG

Retreival Augmented Generation aka RAG is when a system query local data store to build better context and use LLM to compose response. This serves followings:

- Adds context to the query so that LLM can respond more suitable manner
- Bridge the gap between LLM training lag with more recent data
- Potentially solve the data security problem as RAG can be run within private network

### Backstory

I built a <a href="https://rahulguha.com/tech/podcast-summary-part-1/" target="_blank">Podcast Monitor System</a> where I monitor new episodes from a list of podcasts, transcribe them, summarize them and finally send email with the summaries and links. It is deployed in Github Actions and scheduled to run every morning. The transcripts are stored in S3.

For reference, feel free to take a peek at <a href="https://github.com/rahulguha/podcast_monitor" target="_blank">My Github Repo</a>

So over time it built out some amount of corpus of transcribed podcasts. So I now want to build a RAG system on top of that so that I can chat with that corpus.

To note, it's an interesting corpus as the podcasts are quite diverse that includes from NyTimes to Indian podcasts covering topics completely disjoined from each other.

### Requirements

- A Chat interface - (Where I can ask question and it will respond)
- Based on intent of question it should switch between giving me summary or answer
- It should always include sources (which podcast it refers to)

### Technology Decisions

There are few decision points of a typical RAG system.

**What Kind of RAG will it be?**

There are few RAG techniques:

Each are essentially makes context buildup more sophisticated for better result.

- **Vector/Sementic RAG** - This essentially uses embedding and vector database and then search for similar chunks of stored vector data to build out context. This is simple, cheaper and probably adequately usable.

- **Hybrid RAG** - This is a simpler concept building on top of Vector RAG. Here we deploy a keyword search capability (like bm25) along with vector search and build context with both results before sending to the LLM.

- **Graph RAG** - This is quite specialized where we create a knowledge graph from the corpus - that creates a graph of keywords used in the corpus. There are few very good libraries available. I tried with <a href="https://github.com/microsoft/graphrag" target="_blank">Microsoft's graphrag</a> to create knowledge graph for part of my corpus. However, using that with openai makes it very expensive and slow. This is why I didn't apply for the whole corpus (or around 70 transcripts and growing).

I started with Vector RAG, played with Graph RAG - which do increase response quality significantly but expensive. Then looked at hybrid RAG. In couple of days I spent with it, I didn't see drastic improvement in response quality. So I stayed with traditional Vector RAG. However I added capability to understand the intent of question (in quite simple manner which can be improved in future) and create different chains pointing to different LLMs for different tasks.

**Future Improvement:**

- Will integrate tools and agents in future.
- Evaluate concepts like Hybrid and contextual RAG more

**Choice of Embeddings:** OpenAI Embedding for now. It's bit expensive but fast. I tried local embeddings using ollama - but gets really slow for large corpus.

**Choice of Vector Database:** Chroma. This is quite abstracted capability - so can be easy to switch it with other data stores like FAISS if needed.

**Choice of LLM for Q&A:** OpenAI for now, mainly because of convenience. But plan to try out others too in future.

**Choice of LLM for Summary:** Ollama with Gemma2:Latest. Typically transcripts are larger than context window for any close-soruce LLMs, so again I have to chunk and manage chunks. Also running it locally is inexpensive, takes less than 1 mins in my Macbook.

**Framework Choice:** For this project, I used LangChain and it works. However few comments/observations:

- It's bit complex and not very intuitive.
- It changes frequently and you start seeing warnings. So you need to keep upgrading. To their credit, they have a tool that would keep versions and dependencies in sync. But I think building same project without langchain will be one of the next things I would take up.

**Logging:** This was simple choice. As I am already using S3 for files staging, using that for logs was logical.

**Source Repo:** GITHUB - Here is the <a href="https://github.com/rahulguha/hybrid-rag" target="_blank">repo link</a>

## Architecture

<img src="/tech/vector-rag-Page-1.png" alt="Architecture Layers"  height="500">

### Vector Generation Sequence

{{< mermaid >}}
sequenceDiagram %% diagram
autonumber
participant pre as Pre Loading
participant s3 as AWS S3
participant l as Local <br/>File Store
participant c as Chunking Process
participant oe as OpenAI Embedding <br/> Store
participant e as Embedding Process
participant v as Vector DB (Chroma)

    pre->>s3: Download Transcriptions
    s3->>l: Files
    c->>l: Get files
    c->>c: Create Chunks
    c->>oe: Get Embedding


    c->>e: Create embedding
    c->>v: Store embedding in Vector Database

{{< /mermaid >}}

### Retrieval Sequence

{{< mermaid >}}

sequenceDiagram %% diagram
autonumber
% participant
actor u as User
participant q as Query <br/>Processor
participant f as Find <br/> Intent
participant v as Vector DB
participant p as Prompt

    participant m as Metadata <br/>Manager
    participant l as Local <br/>File Store


    u->>q: question
    q->>f: Summary or QA
        alt QA
            q->>v: Get similar chunks <br/> Vector Search
            q->>p: Build Prompt with context (chunks)
            q->>m: Get Sources from metadata
            create participant openai as OpenAI
            q->>openai: Send Prompt
            destroy openai
            openai->>q: Response
            q->>u: Resposne + Sources

        else Summary
            q->>v: Get similar chunks <br/> Vector Search
            q->>m: Get Sources from metadata
            q->>l: Get Full Episode from Local File Store
            create participant o as Ollama
            q->>o: Get Summary
            destroy o
            o->>q: Summary

            q->>u: Summary + Sources
        end

{{< /mermaid >}}
