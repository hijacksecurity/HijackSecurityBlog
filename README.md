# Hijack Security Blog

A Jekyll-based technology blog focused on security, DevSecOps, AI, and development.

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

### Development in PyCharm
1. Install Ruby plugin (if not already installed)
2. Open project in PyCharm
3. Open terminal in PyCharm
4. Run: `bundle exec jekyll serve --livereload`

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
categories: security devsecops
---
```

### Configuration
- Main config: `_config.yml`
- Custom domain: `CNAME`
- GitHub Pages deployment: `.github/workflows/jekyll.yml`

## Deployment
Push to main branch - GitHub Actions will automatically build and deploy to GitHub Pages at `blog.hijacksecurity.org`.