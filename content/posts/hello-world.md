---
title: "Hello, World"
date: 2026-07-05T11:20:00+10:00
draft: false
tags: ["meta"]
summary: "The first post on my new blog."
---

Welcome to my blog. This site is built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, and it's
deployed automatically to GitHub Pages whenever I push to `main`.

## Writing a new post

Create a Markdown file under `content/posts/`:

```bash
hugo new content posts/my-next-post.md
```

Edit the front matter, set `draft: false` when you're ready, write in Markdown,
then commit and push. GitHub Actions rebuilds and deploys the site for you.

## Previewing locally

```bash
hugo server -D
```

That serves the site at <http://localhost:1313> with live reload (the `-D` flag
includes drafts).

More to come.
