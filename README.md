# Storaged Project Website

Source for [storaged.org](https://storaged.org) website.

## Local Development

```bash
bundle install                 # Install dependencies
bundle exec jekyll serve       # Dev server at http://localhost:4000
bundle exec jekyll build       # Build static site to _site/
```

## Contributing

Blog posts go in `_posts/` as `YYYY-MM-DD-Title.md`.

Each blogpost starts with a metadata block:

```yaml
---
layout: post
title:  "Post Title"
date:   2026-01-01 10:00:00 +0100
categories: category
author: Author Name
---
```

Note: Jekyll will exclude posts with a future date from the build so don't forget to adjust the date before merging PR with the new post.

New authors need to be added to `_data/authors.yml` with their name, short bio, and GitHub username:

```yaml
Author Name:
  name: Author Name
  bio: Short description of the author.
  github: github-username
```
