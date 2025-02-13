---
layout: post
title: "Improving AI-Generated Git Commit Messages: Script Updates"
date: 2025-02-13 
---

[A month ago](http://localhost:4000/2025/01/17/ai-commit-messages.html), I shared my approach to using AI for generating Git commit messages. Here's what's new in the updated version.

## Key Improvements

### 1. Better Handling of Existing Messages

The script now preserves existing commit messages when:
- Using the `-m` flag with `git commit`
- Amending commits
- Returning to edit a previous attempt

### 2. Improved AI Prompting

The prompt engineering has been refined to:
- Generate more concise messages
- Ignore test file changes unless they're the primary changes
- Emphasize changes to shared files and data models
- Highlight changes affecting external systems (S3, caching, etc.)

### 3. Better Message Presentation

The new script preserves the original Git commit template and clearly marks AI suggestions as optional, making it easier to edit or discard the generated message.

## Why These Changes Matter

These improvements make the script more practical for daily use by:
- Respecting existing Git workflows
- Focusing on substantive changes that affect the codebase
- Making it clearer what's AI-generated and what's not

## The Complete Script

Here's the updated version in its entirety:

```bash
#!/bin/bash

# Get the name of the file containing the commit message
COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2
SHA1=$3

# Read the current commit message
current_message=$(grep -v '^#' "$COMMIT_MSG_FILE" | tr -d '[:space:]')

# Skip if commit message already exists (from -m flag or amend)
if [ "$COMMIT_SOURCE" = "message" ] || [ "$COMMIT_SOURCE" = "commit" ]; then
    exit 0
fi

# Skip if the commit message is not empty (user edited previous attempt)
if [ ! -z "$current_message" ] && [ "$current_message" != "$(grep -v '^#' "$COMMIT_MSG_FILE")" ]; then
    exit 0
fi

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
  {"role": "user", "content": ("Suggest a short commit message by summarizing code changes from the shared command output. 
    The commit message should follow the conventional commit format and provide meaningful context for future readers.
    Be very concise. 
    Use bullet points.
    Do not include descriptions for test and spec files unless most changes are to those files.
    Do include descriptions for changes to shared files and commits.
    Whitespace changes do not need mention unless all changes are whitespace.
    Emphasize changes to changes that will impact a database datamodel or data stored in external systems like S3 or caching technologies.
    Be very concise. 

    Repond in plaintext with no markup.
    \n\nChanges:\n" + $diff)}
]')

# Send the request to OpenAI API using the correct chat endpoint
response=$(curl -s -X POST https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d "$(jq -n \
        --argjson messages "$messages" \
        '{
          model: "gpt-4o-mini",
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

# Save the original template
template=$(cat "$COMMIT_MSG_FILE")

# Write the AI-generated commit message followed by the template
{
  echo "# AI-suggested commit message (edit or delete):"
  echo "$commit_message"
  echo ""
  echo "$template"
} > "$COMMIT_MSG_FILE"
```

You can find the latest version of this script in my [system files repository](https://github.com/softwaregravy/system_files/blob/master/git_hooks/prepare-commit-msg).
