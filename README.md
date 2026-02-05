# Hijack Security Blog

A Jekyll-based technology blog focused on security, DevSecOps, AI, and development.

## Features

- **Dark/Light Theme** - Modern theme with automatic system preference detection
- **Search** - Client-side search with live results dropdown (press `/` to focus)
- **Code Blocks** - Syntax highlighting with language labels and copy button (16+ languages)
- **Series Support** - Group related posts into series with navigation
- **Social Sharing** - Share posts to Twitter/X, LinkedIn, or copy link
- **Table of Contents** - Auto-generated TOC for long articles (desktop)
- **Reading Progress** - Visual progress indicator while reading
- **Responsive Design** - Optimized for desktop and mobile

## Local Development

### Prerequisites
- Ruby 3.1+
- Bundler

### Setup
```bash
# Install dependencies
bundle install

# Start development server
bundle exec jekyll serve --livereload

# Server will be available at http://127.0.0.1:4000/
```

### Creating Posts
Create new posts in `_posts/` directory with format:
```
YYYY-MM-DD-title.md
```

Example front matter:
```yaml
---
layout: post
title: "Your Post Title"
date: 2025-07-12 10:00:00 -0000
tags: [Security, AWS, Kubernetes]
series: "Optional Series Name"
series_part: 1.0
---
```

### Project Structure
```
├── _includes/          # Reusable components
│   ├── header.html     # Site header with nav and search
│   ├── post-sharing.html   # Social sharing buttons
│   └── series-badge.html   # Series indicator badge
├── _layouts/           # Page templates
│   └── post.html       # Blog post layout
├── _pages/             # Static pages
│   ├── posts.html      # All posts archive
│   ├── search.html     # Search page
│   └── tags.html       # Tags page
├── _posts/             # Blog posts
├── assets/css/         # Stylesheets
│   └── custom.css      # Main theme styles
└── search.json         # Search index template
```

### Configuration
- Main config: `_config.yml`
- Custom domain: `CNAME`
- GitHub Pages deployment: `.github/workflows/jekyll.yml`

## Deployment
Push to main branch - GitHub Actions will automatically build and deploy to GitHub Pages at `blog.hijacksecurity.org`.
