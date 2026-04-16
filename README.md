# Minimal Blog

A minimalist static blog built with [Eleventy](https://www.11ty.dev/) and Markdown.

The theme is intentionally simple, uses a dark mode palette by default, keeps `#4169e1` as the main accent color, and puts the post list front and center on the homepage.

## Quick start

```bash
npm install
npm run dev
```

Open the local URL shown by Eleventy to preview the site.

To create a production build:

```bash
npm run build
```

The generated site is written to `_site/`.

## Project structure

```text
src/
  _data/site.json        # Site title and description
  _includes/             # Shared layouts
  css/site.css           # Minimal theme
  index.njk              # Home page and post listing
  posts/                 # Markdown blog posts
```

## Writing a post

Add a new Markdown file in `src/posts/`:

```md
---
title: My New Post
date: 2026-04-16
description: A short summary for the index page.
---

Write your post in Markdown here.
```

Posts are automatically listed on the home page and rendered as static HTML.
Each post also gets the closing signature `- Alvaro GJ` from `src/_data/site.json`.

## Customizing the theme

The dark theme tokens and main accent color live in `src/css/site.css`:

```css
--accent: #4169e1;
```

You can also update the site title, author signature, and description in `src/_data/site.json`.
