---
layout: post
title: "Premature Optimization: Sometimes not so bad"
date: 2024-08-20
---

Premature optimization is often seen as a trap for dvelopers—spending time fine-tuning code that may never need to be optimized. But sometimes, what’s labeled
as "premature" might actually be the mark of thoughtful, high-quality code. Recently, I found myself in this gray area while working on a new feature that
involved database queries. Let me walk you through the changes I made and why they were worth the effort.

## The Optimizations

**First Change: Avoiding N+1 Queries**

Originally, I fetched a chat. This was the default setup in my controller.

```ruby
@chat = Chat.find(params[:id]) 
```

As the data model grew, this resulted in two separate N+1 query problems—one for fetching the messages and another for fetching the audio attachments. To fix this, I included
the necessary associations in the initial query:

```ruby
@chat = Chat.includes(messages: {audio_attachment: :blob}).find(params[:id]) 
```

**Second Change: Simplifying Message Rendering**

Initially, I was iterating through messages on my chats but needed to do something special for the last message in my list. This was my first implementation:

```ruby
<% chat.messages.each_with_index do |message, index| %> 
  <% last_message = index == (chat.messages.count - 1) %> 
  <%= render partial: "chat_message", locals: {message: message, autoplay: last_message} %> 
<% end %>
```

This approach worked fine, but it called `count` on every iteration. Even though this call was cached, it really cluttered up the logs. I changed it to:

```ruby
<% last_message = chat.messages.last %> 
<% chat.messages.each do |message| %> 
  <%= render partial: "chat_message", locals: {message: message, autoplay: (message.id == last_message.id)} %> 
<% end %>
```

This avoided unnecessary `count` calls and simplified the comparison logic.

## Why not do these when the feature is complete?

1. **Problem Freshness**: We were in the middle of building this feature, so the context was fresh. Making these optimizations now was straightforward and
painless. Coming back to find and then implement these changes later would have taken significantly more time. Also, lets not underestimate the organizational
overhead of creating tasks, scoping, prioritizing, etc.

2. **Low Hanging Fruit**: These were small, easy changes that didn’t require significant refactoring. The problems were obvious from the logs. The fixes
were either close to trivial or easy to Google. (Whats the new word to "ChatGPT it"?)

3. **Minimal Time Investment**: These optimizations took less than 5% extra time vs. total task time. If an optimization will take less than 5%-10% additional time, it’s
often worth it.

## Conclusion: Premature or High-Quality?

What some people might call premature optimization can actually be seen as the mark of high-quality code. By making these small, thoughtful changes early on, we
ensure that our code is both efficient and maintainable. It’s a balancing act—knowing when to optimize and when to leave things as they are—but in these cases, I felt it
was worth the effort.
