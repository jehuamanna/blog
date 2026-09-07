---
title: Why Astro Fits a Solo Blog
description: A practical note on keeping a writing site fast, simple, and easy to deploy.
pubDate: 2026-09-07
tags:
  - astro
  - publishing
---

Astro is a good default for a solo blog because it keeps the main workflow close to writing.
Posts live in the repository, pages build to static files, and the site does not need a server for ordinary publishing.

That matters when the goal is consistency. A blog is easier to keep alive when updates are just a Markdown file, a commit, and a deploy.

## The stack

- Astro for the site shell and routing
- Markdown or MDX for posts
- Content collections for typed frontmatter
- GitHub for source control
- Cloudflare Pages for hosting

This keeps the system small while leaving room for interactive posts later.
