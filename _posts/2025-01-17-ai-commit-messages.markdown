---
layout: post
title: "Using AI to Draft Git Commit Messages"
date: 2025-01-17 14:50:00 -0400
---

Writing good commit messages is crucial for maintainable code, but it's often overlooked in the rush to push changes.
Recently, I automated a process using GPT to draft my commit messages.

## The Problem

We all know we should write meaningful commit messages, but in practice:

- It's easy to default to vague messages like "fix bug" or "update code"
- Writing good messages takes mental energy
- The bigger the commit, the harder it is to remember all the changes
- Early in a project, you might be the only developer and feel the cost-benefit is not there

## The Solution: AI-Assisted Commit Messages

I adapted [Dustin Davis's approach](https://dustindavis.me/blog/use-ai-to-write-your-git-commit-messages/) to create a Git hook that automatically suggests commit messages based on staged changes.

Now when I type `git commit` it opens up my editor and has a good draft of a commit message ready to go.

Here's how it works:

```bash
#!/bin/bash

# Get only the diff of what has already been staged
git_diff_output=$(git diff --cached)

# Check if there are any staged changes to commit
if [ -z "$git_diff_output" ]; then
  echo "No staged changes detected. Aborting."
  exit 1
fi

# Limit the number of lines sent to AI to avoid overwhelming it
git_diff_output_limited=$(echo "$git_diff_output" | head -n 1000)

# Prepare the AI prompt for the chat model
messages=$(jq -n --arg diff "$git_diff_output_limited" '[
  {"role": "system", "content": "You are an AI assistant that helps generate git commit messages based on code changes."},
  {"role": "user", "content": "
    Suggest an informative commit message by summarizing code changes from the shared command output. 
    The commit message should follow the conventional commit format and provide meaningful context for future readers.
    Be concise.  
    Use bullet points.
    \n\nChanges:\n" + $diff)}
]')

# Send the request to OpenAI API using the correct chat endpoint
response=$(curl -s -X POST https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d "$(jq -n \
        --argjson messages "$messages" \
        '{
          model: "gpt-4",
          messages: $messages,
          temperature: 0.5,
          max_tokens: 5000
        }'
)")

# Extract the AI-generated commit message
commit_message=$(printf '%s\n' "$response" | jq -r '.choices[0].message.content' | sed 's/^ *//g')

# Check if we got a valid commit message from the AI
if [ -z "$commit_message" ] || [[ "$commit_message" == "null" ]]; then
  echo "Failed to generate a commit message from OpenAI."
  echo "API Response: $response"
  exit 1
fi

# Write the commit message to the commit message file
COMMIT_MSG_FILE=$1
echo "$commit_message" > "$COMMIT_MSG_FILE"

```

## Making it Global

Instead of setting this up per repository, I made it global using Git's `core.hooksPath`:

```bash
# Create global hooks directory
mkdir -p ~/.git-hooks

# Copy the script contents above into this file
vim ~/.git-hooks/prepare-commit-msg  # Or use your preferred editor

# Make the script executable
chmod +x ~/.git-hooks/prepare-commit-msg

# Configure Git to use global hooks
git config --global core.hooksPath ~/.git-hooks
```

Now every repository automatically gets AI-assisted commit messages, just use `git commit`.

## Why This Works

1. **Reduces Friction**: The AI analyzes changes and suggests a message before I even start thinking about what to write. Even if I modify the suggestion, starting with a draft is easier than starting from scratch.

2. **Consistency**: The AI consistently includes relevant details and follows conventional commit formats, making the commit history more useful.

3. **Context Preservation**: The AI can notice and document subtle changes that I might overlook when writing commit messages manually.

## Conclusion

Using AI to draft commit messages is a small automation that makes a meaningful difference in code quality.
It's not about replacing human judgment but about making it easier to maintain good practices.
The best tools are often the ones that reduce friction in doing the right thing - and this is definitely one of those tools.
