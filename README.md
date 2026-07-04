# arthurpanhku.me

Personal homepage — a Jekyll site served by GitHub Pages at [arthurpanhku.me](https://arthurpanhku.me).

## Structure

```
_config.yml            site config
_layouts/default.html  shared shell (nav, head/OG, footer)
_layouts/post.html     article layout
index.html             homepage (hero, projects, latest writing)
blog.html              /blog/ — full post list
_posts/                blog posts, in Markdown
assets/style.css       all styles
banner.jpg             hero image
favicon.svg
```

## Write a new post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-slug.md`:

```markdown
---
layout: post
title: "Your title"
date: 2026-07-10
tags: [agentic-ai, security]
description: "One-line summary (used on the list page and for link previews)."
---

Write in Markdown. Code blocks, images, and headings all work.
```

Commit and push to `main` — GitHub Pages rebuilds automatically. The post shows up on
`/blog/` and, if recent, in the "Writing" section of the homepage.

## Preview locally (optional)

```
bundle exec jekyll serve      # http://localhost:4000
```
