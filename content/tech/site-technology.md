+++
author = "Rahul Guha"
title = "How this site was built"
date = "2024-12-19"
description = "How this site was built"
tags = [
    "tech",
    "Hugo",
    "Blog",
    "SSR"
]
categories = [
    "General",
    "SSR",
]
series = ["Blogging Technology"]
aliases = ["SSR"]
thumbnail = "images/building.png"
+++

I will describe how this site was built and rendered for free. <br/>
There were many options - So decision making was bit complex. Also as I went deep and wanted to implement certain diversion from standard framework and styles, I needed to go deep in terms of how the framework and call flow works. 
<!--more-->




| Steps |Sequence Diagram  |
|----------|---------|
|[• Requirements](#requirements) <br>[• SSG - Static Site Generation](#static-site-generator) <br>[• Sequence Diagram](#sequence-diagram) <br>[• Install Hugo & Theme](#install-hugo-and-theme)  <br>[• Update and Test Locally](#update-and-test-locally)  <br>[• Create & Setup Github Repo](#create-and-setup-github-repo) <br>[• Setup Github Pages](#setup-github-pages) <br>[• Create Workflow file](#create-workflow-file) <br>[• Add Commit Push](#add-commit-push) <br>[• Action Build and Deploy ](#action-build-and-deploy) <br>[• Test deployment in browser](#test-deployment-in-browser)  | [![High Level Architecture:inline](/images/ssg-architecture.png)](/images/ssg-architecture.png)


## Requirements
I spent few mins to write down my Non Functional Requirements :
In [GHERKIN](https://blog.redrockresearch.com/post/2023/02/17/where-did-the-term-gherkin-originate-and-get-used-with-agile-user-stories#:~:text=The%20Gherkin%20syntax%20is%20a,technical%20and%20non%2Dtechnical%20people.) terms: 
``` 
I want to build my blogging website so that I can publish various technology articles.
Given 
  I want to publish my article - around 4-6 times in a month (may be once in a week)
When
  I want to write it in markdown
  And
  I want to include one or multiple images 
  And 
  I want to use developer standard editor (e.g. VS Code)
  And
  I want to store the writing in github personal account
  And 
  I push it to github repo
Then
  It should be automatically built and deployed to github pages
  And
  Available for consumption from anywhere
```
Based on all of these I decided to use one of the SSG (Static site generator) framework where the pages are pre rendered as html pages (along with js and css) so that browser rendering is extremely fast. I also plan to use Github Actions and Pages to build and deploy the rendered HTML for contineous deployment.
<br/>

## Static Site Generator
```
What is SSG - Static Site Generator ?
It's a way to generate static webpages (html, css and js) and serve them without any server side dynamic processing. Pages are pre-built and stored in a public folder for consumption.
```
There are many great SSG frameworks ([Top SSG Frameworks](https://hygraph.com/blog/top-12-ssgs))
 <br/>
From a high level all of them support certain templating, theming and build to public distributable formats. They also implement webpage best practices as well as standard responsive designs for mobile web and various devices. <br/>
With many options, comes too many choices. I tried to timebox my selection process as well as stay with popular frameworks (specifically looked at how active their github repo are). Criteria that I looked at are as follows:
* Features ( sidebar, dark vs light theme etc.)
* Design (Aesthetics)
* Performance ( Google [Pagespeed](https://pagespeed.web.dev/) number)
* Github activity and freshness

<b>Note:</b> A good place to look for themes is [Official Hugo themes site](https://themes.gohugo.io/)

<br/>
Finally I decided to: <b>
* Use Hugo for SSG framework
* Use Hugo clarity as hugo theme
* Use github as code repository
* Use github action and github pages for build and deploy respectively
</b>

Below sequence diagram explains various steps: 
## Sequence Diagram
[![Sequence Diagram:inline](/images/sequence.png)](/images/sequence.png) <br/>


## Install Hugo and Theme
[Use instructions here to install hugo locally for various platforms](https://gohugo.io/installation/). <br/>
Check hugo install by checking version 
```hugo version```. 
Mine was v0.140.0. <br/>
You can also check hugo install folder by 
```which hugo ```

### Theme Setup
Typical steps are:
* Create a Hugo Site (```hugo new site sitename ```)
* Install theme - this is done by either cloning the theme from github or by adding it as submodule.<br/>[Hugo Clarity Setup](https://github.com/chipzoller/hugo-clarity)

## Update and Test Locally
* <b>Familiarize with configs</b>: For Clarity there is a config folder which has multiple config files. <br/>
  - <b>hugo.toml</b>: Site level parameters - like Title, BaseUrl, Copyright, theme etc  
  - <b>param.toml</b>: This is the main config file - that holds most of the values. As there are many configs, suggest consulting [this documentation](https://github.com/chipzoller/hugo-clarity?tab=readme-ov-file#table-of-contents) <br/>
  However this is where themes go in different directions. I have used PaperMod before which uses one config.yml file instead of multiple config files. So this step and file location may differ based on theme choice.
* <b>Update images (e.g. Logo, favicon etc)</b>: This is typically done by adding / updating files in ```/static``` folder. Also this is implemented differently in different themes. In clarity there are three folders (icons, images and logos). Logo and favicons need to go to corresponding folders.   
* <b>Update Menu</b>: Typically menu items are updated in config file along with their weight and pointing link. Clarity has an advanced menu system that creates a language specific ```menu.toml```  for menus. 
* <b>Content Planning</b>: At this step folder structure under ```\content ``` needs to be planned. I decided to have my posts under ```Tech``` and ```Book Review ```. So I created those folders under content. 
  - <b>[Front Matter:](https://gohugo.io/content-management/front-matter/) </b> Every post can have few fields to help SEO and hugo engine to build and pre-render contents. An example would be:
  ```
  +++
  author = "Rahul Guha"
  title = "How this site was built"
  date = "2024-12-19"
  description = "How this site was built"
  tags = [
      "tech",
      "Hugo",
      "Blog",
      "SSR"
  ]
  categories = [
      "General",
      "SSR",
  ]
  series = ["Blogging Technology"]
  aliases = ["SSR"]
  thumbnail = "images/building.png"
  +++
  ```
<br/>
While this is quite self explanetory, it's powerful - more details in theme documentation

  - <b>_index.md:</b> This is an optional file in the folder. I want to think this as a front matter for all the posts in certain folder. Example:
  ```
  +++
  aliases = ["tech", "tech-articles", "tech-blog"]
  title = "Tech Blogs"
  author = "Rahul Guha"
  tags = ["index", "tech"]
  +++
  ```
  <!-- - <b>index.md:</b>This file is optional.  -->
## Create and Setup Github Repo
Configuration details...

## Setup Github Pages
Customization options...

## Create Workflow file
Concluding remarks...

## Add Commit Push
Concluding remarks...

## Action Build and Deploy
Concluding remarks...

## Test deployment in browser



<!-- 
<p><span class="nowrap"><span class="emojify">🙈</span> <code>:see_no_evil:</code></span>  <span class="nowrap"><span class="emojify">🙉</span> <code>:hear_no_evil:</code></span>  <span class="nowrap"><span class="emojify">🙊</span> <code>:speak_no_evil:</code></span></p>
<br>

The [Emoji cheat sheet](http://www.emoji-cheat-sheet.com/) is a useful reference for emoji shorthand codes.

***

**N.B.** The above steps enable Unicode Standard emoji characters and sequences in Hugo, however the rendering of these glyphs depends on the browser and the platform. To style the emoji you can either use a third party emoji font or a font stack; e.g.

{{< highlight html >}}
.emoji {
  font-family: Apple Color Emoji, Segoe UI Emoji, NotoColorEmoji, Segoe UI Symbol, Android Emoji, EmojiSymbols;
}
{{< /highlight >}}

{{< css.inline >}}
<style>
.emojify {
	font-family: Apple Color Emoji, Segoe UI Emoji, NotoColorEmoji, Segoe UI Symbol, Android Emoji, EmojiSymbols;
	font-size: 2rem;
	vertical-align: middle;
}
@media screen and (max-width:650px) {
  .nowrap {
    display: block;
    margin: 25px 0;
  }
}
</style>
{{< /css.inline >}} -->
