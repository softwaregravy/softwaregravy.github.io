---
layout: post
title: "AI-Driven Web Frameworks: The Config-First Approach"
date: 2025-02-27 10:30:00 -0500
categories: ai software-engineering web-development
---

In my previous post, I explored what a web framework purpose-built for AI might look like. Today I want to share another possible approach: what if we created a super-configurable framework that derives its functionality from JSON-based configurations?

## The Config-First Framework

Instead of having AI generate application code from scratch, what if we had a framework where functionality comes from modular libraries specified and configured entirely through JSON?

The mental model I have is Ruby on Rails and its ecosystem. Rails is highly opinionated, which is exactly what we'd want in an AI-first framework - but even more so.

Consider authentication with Devise in the Rails world. You install the gem, generate some files, edit the config, and you have a complete authentication system. In our JSON-based approach, you would specify Devise and its configuration directly:

```json
{
  "devise": {
    "version": "4.9.2",
    "configuration": {
      "provider": "local", 
      "passwordReset": true, 
      "confirmable": true
    }
  }
}
```

This maintains the modularity of libraries and doesn't require everything to be pre-installed, while still providing a structured configuration interface.

## The Application as Configuration

In this paradigm, your application becomes primarily a configuration file rather than custom code. The application-specific controllers and models would still need to be written or generated by AI, but the goal would be having as much functionality as possible compartmentalized into libraries.

This approach has several advantages:
- Less custom code to maintain
- More standardized implementations
- An interface that's perfect for AI interaction
- Faster development cycles

## The AI's Role

The AI driving this system would need to be aware of the available libraries and their configuration options. It would need to:
- Understand each library's capabilities and configuration schema
- Keep track of compatible versions and dependencies
- Generate appropriate configurations based on natural language descriptions

This creates a natural division of labor:
- Humans describe what they want
- AI selects appropriate libraries and generates configurations
- The framework assembles and implements the behavior

With this approach, the AI would need a way to reliably load and understand library documentation, or perhaps libraries could provide standardized schema files describing their configuration options.

I'm curious what other ideas people have for AI-first frameworks. Maybe "prompt-driven framework" is a better term than "config-first framework" since the ultimate interface is the natural language prompt to the AI, not the config file itself. What approach do you think would work best?
