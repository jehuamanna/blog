# brokentablets

Computer programming notes by Jehu Amanna, a tinkerer.

This blog uses a lightweight solo-developer stack:

- Astro 7
- Markdown and MDX posts
- Typed content collections
- RSS and sitemap generation
- GitHub as the source repository
- Cloudflare Pages as the deployment target

## Start locally

```sh
npm install
npm run dev
```

## Write a post

Create a Markdown or MDX file in `src/content/posts/`:

```md
---
title: My Post
description: A short summary for the homepage, RSS, and metadata.
pubDate: 2026-09-07
tags:
  - notes
---

Your post starts here.
```

Draft posts are supported with `draft: true`.

## Build

```sh
npm run build
```

The static site is generated in `dist/`.

## Deploy on Cloudflare Pages

Connect this GitHub repository to Cloudflare Pages and use:

- Build command: `npm run build`
- Build output directory: `dist`
- Environment variable: `SITE_URL=https://brokentablets.citrus-elm-3499.chatgpt.site`

Set `SITE_URL` to your production domain so sitemap and RSS links are generated correctly.

## Profile Links

- X: https://x.com/jehuamanna
- LinkedIn: https://www.linkedin.com/in/jehuamanna/
