---
layout: post
title:  "Launched my first Bolt App"
date:   2025-01-26 11:42:23 -0500
---

# Building with Bolt.new: A Stack Trace Sanitizer

Today I solved a common pain point in my development workflow using bolt.new, a platform that generates complete applications from natural language prompts.

## The Problem
When filing GitHub issues with stack traces, I needed to scrub my username from the logs. The challenge wasn't just finding and replacing text - it was that once the stack trace was in my clipboard, I couldn't easily pipe it through command-line tools like sed. This made sanitizing logs more tedious than necessary.

## The Solution
Using bolt.new, I created a simple web app that handles this username scrubbing process. The platform's natural language interface made development surprisingly straightforward, and the automatic Netlify deployment meant I had a working solution in minutes.

## The Experience
The development process with bolt.new was refreshingly simple. The platform understood my requirements and generated a functional application that addressed my specific needs. Netlify's integration made hosting seamless, though I did encounter one notable limitation: there's no direct GitHub integration. I had to manually download and migrate the code, which raises questions about how easily I can return to Stackblitz for future modifications.

## Looking Forward
Despite the GitHub workflow hiccup, this experience demonstrated the power of AI-assisted development for creating practical utility apps. The platform delivered on its promise of rapid development and deployment, making it an attractive option for future projects that need quick implementation. Next time I need to sanitize a stack trace, I'll have a purpose-built tool ready to go.

## Links
* App: [https://sanitizer.softwaregravy.dev/](https://sanitizer.softwaregravy.dev/)
* Code: [https://github.com/softwaregravy/stack_trace_sanitizer](https://github.com/softwaregravy/stack_trace_sanitizer)
