---
date: '2025-01-26T11:13:35-06:00'
draft: false
author: ['Rahul Guha']
tags: ["tech", "cybersecurity", "LLM"]
category: ["Technology", "Artificial Intelligence"]
title: 'Podcast Summary Part 1: Basics'
aliases: ['/tech/podcast-summary-part-1-basics']
summary: This will explain various aspects of LLM be used to build a product
Mermaid: True
weight: 1
---
### Github Repo
``` https://github.com/rahulguha/podcast_monitor ```
<img src="/tech/overall-picture.png" alt="Component Diagram"  height="500">
### Background and Intent
I am not here to build a real product (although it may go in that direction 😄) - but main intents are the followings:
- Wanted to familiarize with LLM as a technology
- Bring in my decades of industry and software development experience togather to see how those can marry and live together - and hopefully become almost happy partners (why not ?)
- Teach myself what to expect and where to be cautious and patient while dealing with LLM and agentic world
- Execute thought experiement on how this can help an Architect - especially with cybersecurity in mind
Simply put - these series of posts won't be a detailed howto guides (although I will include github repos) - also don't expect high quality code as coding is NOT the topmost interest. What you can expect is connecting dots across various architectural constructs and puzzles. For example - running a model locally or codelab is great - but how it can mesh with established cloud and devops processes. 
It will be multipart - I expect to build on top of one theme and add more scope over weeks.

### Requirements ##

**High Level Story:**
As an avid podcast listener, I want to monitor few podcasts for new episodes so that I can get notified when a new episode is dropped.

**Given**
I have a list of podcasts in a config file
**When**
Schedule comes
**Then**
New podcasts to be transcribed, summarized and send over email to me

As a final result I want to get an email like this 
<img src="/tech/podcast_email.png" alt="Alt Text"  height="400">
<!-- ![Email Picture](/tech/podcast_email.png "Final Outcome") -->

### Architecture and Technology Choices ##

Let's divide the whole project in few major pieces:

**Big Chunks / Epics**
- Get new episodes
- Transcribe those in text files
- Summarize those and store summaries
- Send email
Also make sure don't re-transcribe or re-summarize same content as those are time consuming and potentially expensive

**Overall Guardrails** that I followed:
- Use local and open source models as much possible
- Use aws S3 for file storage and retrieval
- Use well known python libraries to perform text manipulation tasks
- Use  <a href="https://openai.com/index/whisper/" target="_blank">Whisper</a> for transcription 
- Deploy code in github and use Github action to run this periodically

  
``` I absolutely plan to experiment with other closed and frontier models ```

**How it will work:** Simply put, a driver script will first look for new episodes, transcribe them using Ollama, store transcription in S3, Summarize them (and convert into nicer html), store them for reuse and finally send over email. All these will run in Github actions instance at 6 am Central Time

**High Level Components**
<img src="/tech/overall-picture.png" alt="Component Diagram"  height="500">

**Flowchart:**
{{< mermaid>}}

graph TD;
	__start__([Scheduled Job triggering<br/> at 6 am CST daily]):::first
	driver(driver)
	monitor(monitor and find <br/> latest episides)
    transcribe(transcribe with <br/> Whisper)
    summarizer(summarize <br/>transcriptions)
    email(email with <br/>MailJet)
	__end__(end):::last
	__start__ --> driver;
	driver --> monitor;
    monitor --> localstorage;
	driver --> transcribe;
    transcribe -.->localstorage
	transcribe <-->s3
    driver --> summarizer;
    summarizer<-->s3
    email -->s3
    driver --> email;
	driver --> __end__;
	
{{</ mermaid>}}
**Sequence Diagram**
{{< mermaid >}}
sequenceDiagram
    participant actions as Github Actions
    participant feeds as Feed Stores
    participant m as Monitor
    participant t as Transcription
    participant w as Whisper
    participant s3 
    participant summ as Summarizer
    participant o as Ollama @ <br/>GitHub Actions

    participant mail as Mailjet <br/>Engine
    
    actions -->actions: Wake up at 6 am central
    actions -->actions: Run Scheduled Workflow which prepares the env
    Note over m, feeds: Check for new feeds
    actions ->m: starts the driver script
    m -> m: If new Feed found <br/> create local file
    
    
    m ->>t: Start Transcription
    t->s3: check if transcription already done
    Note over t, w: Transcribe new audio files
    t->>s3: store transcribed files as json.dump <br/>preserving episode metadata
    t->>m: transcription complete

      
    m->>summ: Summarize
    summ ->> s3: compare transcription and <br/>summary file names <br/>to decide what <br/> needs to be summarized
    Note over summ, o: Use local Ollama <br/> with LLama 3.2 to summarize
    summ ->> m: Summarization Complete
    m->>mail: Send Email

{{< /mermaid >}}

### Github Repo ##
This is good time to introduce **<a href="https://github.com/rahulguha/podcast_monitor/" target="_blank">My Repo</a>**. All code used in this project are available there. I will explain code in high level in next sections

- **Config Files:** content_monitor.json lists the podcasts that are monitored in this project. Structure is very obvious as you see here:
```
{
        "category": [
                        "history",
                        "npr"
                    ],
        "name": "NPR - Throughline",
        "FeedLink": "https://feeds.npr.org/510333/podcast.xml"
    }
```
Other configs available in environment variable are:
```
CUTOFFDATE="2025/01/21"
TRANSCRIPTIONFOLDER="transcriptions"
S3_BUCKET="podcast.monitor"
```
- **Monitor:** ```monitor_podcast.py``` - This loads the config json, looks at FeedLink field for new episodes and if found writes into a file called ```to_be_processed.json```. The link points to the RSS feed for the podcast. In nutshell, RSS is a standard format for describing Blogs and Podcasts. It is typically published by the producer and has various fields that describes the production. <br/><sub>In case anyone wants to know more about RSS - here is <a href="https://en.wikipedia.org/wiki/RSS" target="_blank">WiKiPedia link</a>. </sub>

- **Transcription:** ```transcribe.py```- This file is responsible for 
  - Looking into S3 bucket to see if the particular audio file is already transcribed or not
  - If not, transcribe using whisper API call
  - Store in s3 bucket (with date prefix) along with Metadata (Podcast Name, Episode Name etc ) for future use
- **Summarizer:** ```summerizer.py```- This file is responsible for 
  - Get a list of files to be summarized - it's done by looking into s3 bucket (```transcription``` folder) for all date prefix
  - Find if summary already exists for the specific transcription. This is also done by looking into ```summary``` folder 
  - If files to be summarized, this issues a prompt to ```local ollama engine with Llama 3.2``` for summary. 
  - Once summary is available, it calls another python library (```beautifulsoup```) to convert text to HTML 
  - Store the HTML fragment into s3 bucket (with date prefix) along with Metadata (Podcast Name, Episode Name etc ) for email
- **EMail:** ```email_client.py```- This file is responsible for 
  - Get all summary files from S3 buckets (obviously excluding the folders beyond cutoff date)
  - Create a connection with mail server (in this case mailjet)
  - Use designated template, send all summary file text for email to be sent
- **Driver Script:** ```monitor.py```- This is a simple driver script that calls all of the above functions sequentially 
- **Deployment:** ```.github/workflows/scheduled-job.yml```- I chose github actions for running the whole project. In future iteration we will deploy this to other platforms too. This step following are done:
  - **Schedule:** Define cron for running the workflow daily at 6 am central 
  - **Setup Environment:** Because of nature of the project, along with regular steps few extra steps needed to be added as follows:
    - Install ffpg - this is required for transcription
    - Install Ollama - for running locally
    - Setup API keys and secret for OpenAI, AWS and MailJet
  - **Execute driver:** Run ```monitor.py```  
### Outcome
Every morning I receive email from this job with all episodes that were published after cutoff date. The job typically takes around 4-30 mins based on how many files need to be transcribed and summarized

### Observations
This is first time I coded with significant help from Claude, OpenGPT, Gemini etc. My observations are documented in <a href="https://rahulguha.com/rants/llm-coding/" target="_blank">Code with LLM - first impressions</a>


### Next Steps

Following are the plans (not in same order) to enhance the scope of the project:
- Include Youtube Videos in the list of podcasts
- Implement RAG with the transcripts to build a chat bot
- Experiment with other models to summarize the transcripts
- Accomplish same using agentic frameworks
- Look at other deployment options







