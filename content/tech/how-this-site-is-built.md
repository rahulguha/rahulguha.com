---
date: '2024-12-19T18:15:07-06:00'
draft: true
title: 'How This Site Is Built'
summary: How this was built.
tags: [ "tech", "cybersecurity"]
author: ['Rahul Guha']
draft: false
aliases: [/tech/how-this-site-was-built]
Mermaid: true
weight: 3
---

In this post I shall explain how this site is built.

### Requirements

- Simplicity - (I spent too much time before wrestling with Wordpress)
- SEO friendliness - obviously
- Have a fairly sophesticated mechanism for Tags, Categories and Menus. I would side with simoplicity over complex sophestication
- Write posts in VS code directly in markdown.
- Host free and sepend zero time managing deployments

### Technology Choices

Let me start from bottoms up.

**Hosting Service:** I knew I will host in one of the static site hosting services - so choice between netlify vs github pages. Chose Github only because of my familiarity, same platform (as code will be staying in github). Very limited exposure to other platforms except aws cloudfront (through s3) which was more complex to run and may cost you few dimes (very less but there is a billing).

**Framework Choice:** Again my previous experience in HUGO played a role. I liked it always because of performance, matured user base, wide template support and close integration with github. However I tried with few themes but finally sticked with Papermod - because of simplicity. I think HUGO is less opinianated in terms of order of config variables and felt too flexible for various theme builders make things more complex. But features drive complexity - and that's understandable. I also checked out few other frameworks like 11ty, jeykill, gatsby etc - but stayed with HUGO :)

**Theme Choice:** Spent sometime to find right theme in the collection of many many HUGO themes. This is where I have bit of dispair - as I moved from one theme to another, I expected they are mostly interchangable except few look and fill components. But they are NOT - From Config to location of static assets, HUGO is too flexible to allow theme developers to do things in their way. That complicates life of end users, who are using HUGO to essentially simplify their blogging life. But inconsistency in config and artifacts between themes caused me quite a bit time as I moved back to PaperMod from Clarity as the later was needing me to spend longer time to customize (although looks like it had more bells and whistles and I liked their color schema)

**Github workflow:**
The code for this site is available in this repo.

```

https://github.com/rahulguha/rahulguha.com

```

The workflow code is quite standard one - available under .github/workflows folder. It executes in below order

{{< mermaid >}}
flowchart TD
    subgraph Setup
        A[Setup ubuntu-latest]
        B[Install Hugo CLI]
        C[Install sass]
        A --> B
        B --> C
    end

    subgraph Build
        D[Get latest from main]
        E[Install node & dependencies]
        F[Build with hugo & copy to public]
        D --> E
        E -->|15min| F
    end

    subgraph Deploy
        G[Setup ubuntu]
        H[Copy to github pages]
        G --> H
    end

    C --> D
    F --> G

    %% Adding time annotations
    style E fill:#f9f,stroke:#333
    style H fill:#bbf,stroke:#333
{{< /mermaid >}}

**Github page and Domain setup:**
These are well defined steps as follows:
- [Setup Github pages:](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site) 
- [Configure Custom Domain:](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site) 
- [Configure Domain Registrar (Namecheap for me):](https://gist.github.com/notTag/4a60598d018124c9ac4a7b1f3e2bac9a) 

