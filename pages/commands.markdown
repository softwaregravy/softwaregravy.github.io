---
layout: page
title: Useful Commands
last_modified_at: 2024-01-24
---

# OS X

## Stop Spotlight from indexing certain directories

```bash
sudo mdutil -i off /path/to/directory
```

# Find

## Match file name and execute command


```bash
# Find files which contain the word 'swell'
find . -type f -exec grep -l "swell" {} \;
# and execute a command on them
find . -type f -exec grep -l "swell" {} \; -exec rm {} \;
```





