---
date: '2025-02-28T18:15:07-06:00'
draft: false
title: 'Building a Contextual RAG - Podcast Monitor Part 3'
summary: Contextual RAG - adding context to the query.
tags: [ "tech", "LLM", "RAG", "Contextual RAG"]
author: ['Rahul Guha']
draft: false
aliases: [/tech/building-a-contextual-rag-system]
Mermaid: true
weight: 3
---

<a href="https://github.com/rahulguha/contextual_rag" target="_blank">repo link</a>

> :bulb: <font color="red">All tools and models used here are exchangeable with other models and libraries.</font>

<img src="/tech/img/Contextual-Rag-chat-flow.webp" alt="Contextual RAG Chat flow"  height="500">

#### Chat flow

### Why Contextual RAG ?

Check out my previous post on <a href="https://rahulguha.com/tech/building-a-rag-system/" target="_blank">Standard Vector based RAG</a>

The challenge with standard RAG is - chunking strategy. Because of limited context window, the corpus needs to be chunked into multiple pieces to store in a vector database. When a "similarity search" is executed for final response to be created by LLM, typically we get partial data. So LLM responses may not be rich enough.

That accuracy can be significantly improved by this technique of contextual rag, where along with chunks, we also generate a context string by sending the whole document and the chunk to the LLM and store that with the chunks in the vector DB. On top of this, using reranking logic helps better search resultset ranking so that we can send a more effective context to the LLM for final response.
<img src="/tech/img/Anthropic-contextual-rag-numbers..webp" alt="Graph Accuracy"  height="500">

#### Source <a href="https://www.anthropic.com/news/contextual-retrieval" target="_blank">Anthropic Post </a>

### Pre Processing

Pre processing / chunking strategy is important. Steps used are:

- Split text using <a href="https://python.langchain.com/v0.1/docs/modules/data_connection/document_transformers/" target="_blank">Langchain Text Splitter </a>. This is used for it's ability to be sentence aware.
- Send to anthropic LLM along with whole document to get the context for the chunk.
  > :bulb: Sending whole document to LLM is bit expensive as it may use extra input token for same set of text for every chunk This can be minimized by using Anthropic's <a href="https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching" target="_blank">Prompt Caching feature</a>

### Embedding and Vector Store

Once chunked and context obtained from LLM, I store them in vector database. I use <a href=" https://www.voyageai.com/" target="_blank">VoyageAI </a> for embedding I use ` voyage-3-large`. Also use local file for storage using voyageai library.
<img src="/tech/img/contextual-rag-Contextual RAG Pre Processing.webp" alt="Architecture Layers"  height="500">

#### Pre-processing corpus

### Chat Flow

As the user input comes through, following steps are followed for every request:

- Get query embedding
- <strong>Sementic Search: </strong> Search in local vector store using embedding to receive chunk text and contexts.
- <strong>Rerank </strong> the result using <a href="https://cohere.com/rerank" target="_blank">Cohere Rerank</a>
- Send the reranked chunks and context to LLM.
- LLM returns generated response to be sent to the user

<img src="/tech/img/Contextual-Rag-chat-flow.webp" alt="Contextual RAG Chat flow"  height="500">

#### Chat flow

### Backstory

I built a <a href="https://rahulguha.com/tech/podcast-summary-part-1/" target="_blank">Podcast Monitor System</a> where I monitor new episodes from a list of podcasts, transcribe them, summarize them and finally send email with the summaries and links. It is deployed in Github Actions and scheduled to run every morning. The transcripts are stored in S3.

For reference, feel free to take a peek at <a href="https://github.com/rahulguha/podcast_monitor" target="_blank">My Github Repo</a>

So over time it built out some amount of corpus of transcribed podcasts. So I now want to build a RAG system on top of that so that I can chat with that corpus.

To note, it's an interesting corpus as the podcasts are quite diverse that includes from NyTimes to Indian podcasts covering topics completely disjoined from each other.

**Future Improvement:**

- Will integrate tools and agents in future.
- Queries can be cached for farther improved ranking and reranking step improvement
- Integrate BM25 keyword search to add some more accuracy.

**Choice of Embeddings:** In previous RAG example, I used OpenAI Embedding. Switched to voyageai for experimenting. This can be changed quite easily based on needs and requirements.

**Choice of Vector Database:** VoyageAI for this project. However this is quite abstracted capability - so can be easy to switch it with other data stores like Chroma or FAISS if needed.

**Choice of LLM for Q&A:** This project I used anthropic, but can be switched very easily

**Framework Choice:** In my previous post, I documented my observations on Langchain complexities. So in this one, I stayed away from that and used native LLM specific libraries (except text splitter) and actually liked those more than langchain.

**Logging:** This was simple choice. As I am already using S3 for files staging, using that for logs was logical.

**Source Repo:** GITHUB - Here is the <a href="https://github.com/rahulguha/contextual_rag" target="_blank">repo link</a>

## Architecture

### Pre processing Sequence

{{< mermaid >}}
sequenceDiagram %% diagram
autonumber
participant pre as Pre Loading
participant l as Local <br/>File Store
participant c as Chunking Process
participant e as voyager ai Embedding <br/> Store
participant a as Anthropic
participant v as Vector DB (Chroma)

    pre->>l: Files
    pre->>c: Chunk Files
    c->>c: Create Chunks
    c->>a: Get context for Chunk
    c->>e: Get Embedding
    c->>v: Store embedding in Vector Database

{{< /mermaid >}}

### Retrieval Sequence

{{< mermaid >}}

sequenceDiagram %% diagram
autonumber
% participant
actor u as User
participant q as Query <br/>Processor
participant s as Semantic Search
participant e as voyager ai Embedding <br/> Store
participant v as Vector DB
participant r as Reranking Engine <br/> Cohere
participant c as Chat Manager
participant p as Build Prompt

participant a as Anthropic

    u->>q: question
    q->>s: Semantic Search
    s->>e: Query Embedding
    s->>v: Search with embedding
    v->>s: Chunks with context and metadata
    s->>r: Rerank
    r->>c: Reranked chunks
    c->>c: add to memory
    c->>p: build prompt
    c->>a: Get response
    c->>c: Add to memory
    c->>u: chat response

{{< /mermaid >}}
