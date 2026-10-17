# blog.ishrastogi.in

Source code for my personal blog at **[blog.ishrastogi.in](https://blog.ishrastogi.in)**,
where I write about .NET, cloud, building production AI systems, and life outside code.

I built it from an empty folder. The full step-by-step write-up is here:
[From Markdown file to live web page: building my blog with Astro](https://blog.ishrastogi.in/blog/markdown-to-web-page-with-astro/).

## How it works

- **[Astro](https://astro.build)** generates a static site at build time.
- **Posts are Markdown files** in `src/content/blog/`, validated against a typed schema.
- **Dynamic routing** (`src/pages/blog/[slug].astro`) creates one page per post.
- **RSS feed and sitemap** are generated automatically.
- **Light and dark mode** use CSS variables, with no JavaScript.
- **Azure Static Web Apps** (Free plan) hosts the site.
- **GitHub Actions** deploys every push to `main`.

## Project structure

```text
/
├── public/                   # static files (favicon, images)
├── src/
│   ├── content/blog/         # blog posts (Markdown)
│   ├── layouts/              # shared page layout
│   ├── pages/                # routes: home, about, posts, RSS
│   ├── styles/               # global CSS
│   └── content.config.ts     # post schema
└── astro.config.mjs
```

## Run locally

Requires Node.js 22.22 or later.

```bash
npm install
npm run dev
```

Then open http://localhost:4321.

## Writing a post

Add a Markdown file to `src/content/blog/`:

```markdown
---
title: "My post title"
description: "A one-sentence summary."
pubDate: 2026-10-17
draft: false
---

Post content here.
```

The filename becomes the URL, e.g. `my-post.md` → `/blog/my-post`.
Set `draft: true` to keep a post off the site while you work on it.