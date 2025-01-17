# Jekyll Blog Workflow Guide

This repository contains a Jekyll-based blog. Follow this guide for creating and managing blog posts.

## Prerequisites

- Ruby installed (version specified in `.ruby-version`)
- Bundler gem installed
- Git configured

## Initial Setup

If setting up for the first time on a new machine:

```bash
# Install dependencies
bundle install
```

## Creating and Managing Posts

### Creating a New Post

1. Create a new post with proper naming convention:
```bash
# Posts should be named: YYYY-MM-DD-title.markdown
touch _posts/$(date +%Y-%m-%d)-your-post-title.markdown
```

2. Add the required front matter to your new post:
```yaml
---
layout: post
title:  "Your Post Title"
date:   YYYY-MM-DD HH:MM:SS -0800
categories: category1 category2
---
```

### Working with Drafts

- Create a draft:
```bash
touch _drafts/YYYY-MM-DD-title.markdown
```

- Move draft to published posts:
```bash
git mv _drafts/YYYY-MM-DD-title.markdown _posts/
```

## Development Workflow

### Local Development

1. Start the local server with drafts enabled:
```bash
bundle exec jekyll serve --drafts
```

This will:
- Build your site
- Watch for changes
- Include draft posts
- Make the site available at http://localhost:4000

2. Edit your post and view changes in real-time
   - The server automatically rebuilds when you save changes
   - Refresh your browser to see updates

### Quality Checks

Before publishing, run:
```bash
# Check spelling
bin/spelling

# Run linter
bin/lint
```

### Building and Publishing

1. Ensure your post is in `_posts/` (not `_drafts/`)

2. Build the site:
```bash
bundle exec jekyll build
```

3. Commit and push your changes:
```bash
git add .
git commit -m "Add new post: Your Post Title"
git push origin main
```

## Project Structure

```
.
├── _posts/          # Published blog posts
├── _drafts/         # Draft posts (not public)
├── _site/           # Generated site (don't edit directly)
├── bin/             # Helper scripts
│   ├── lint        # Linting script
│   └── spelling    # Spell check script
├── dictionary.dic   # Custom dictionary for spell checker
└── _config.yml      # Site configuration
```

## Best Practices

1. Post Management
   - Use drafts for work in progress
   - Follow the YYYY-MM-DD-title.markdown naming convention
   - Include appropriate front matter in all posts
   - Use consistent categories across posts

2. Quality Control
   - Always run spelling and lint checks before pushing
   - Preview changes locally before publishing
   - Test all links and images

3. Git Workflow
   - Make atomic commits (one logical change per commit)
   - Write clear commit messages
   - Push regularly to avoid conflicts

## Troubleshooting

If the local server isn't working:
1. Stop the server (Ctrl+C)
2. Remove the _site directory: `rm -rf _site`
3. Rebuild dependencies: `bundle install`
4. Start the server again: `bundle exec jekyll serve --drafts`

## Maintenance

- Regularly update dependencies: `bundle update`
- Keep your dictionary.dic updated with new technical terms
- Periodically review and update categories for consistency
- Clean up old drafts that are no longer needed
