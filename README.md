# guangyue-xu.com — Personal Website

Personal homepage and blog of **Guangyue Xu**, built with Jekyll and hosted on GitHub Pages.

🌐 **Live site**: [www.guangyue-xu.com](https://www.guangyue-xu.com)

---

## Overview

- **Homepage** (`index.html`) — Static HTML profile page (JonBarron-style template) with bio, research interests, and links.
- **Blog** (`_posts/`) — Jekyll-powered Markdown posts covering machine learning, NLP, and information retrieval.
- **Math support** — KaTeX rendering for inline `$...$` and display `$$...$$` equations.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Static site generator | [Jekyll](https://jekyllrb.com/) |
| Hosting | [GitHub Pages](https://pages.github.com/) |
| Math rendering | KaTeX |
| Syntax highlighting | Rouge |
| Fonts / Icons | Inter, Font Awesome 6 |

## Repository Structure

```
.
├── _layouts/          # Jekyll layout templates (post, blog_index, …)
├── _posts/            # Blog posts — named YYYY-MM-DD-slug.md
├── blog/              # Blog index page
├── data/              # Structured data (publications, etc.)
├── images/            # Static images
├── index.html         # Homepage (pure static, no Jekyll)
├── stylesheet.css     # Global styles
├── _config.yml        # Jekyll configuration
├── Gemfile            # Ruby dependencies
└── .gitignore
```

## Writing a New Post

1. Create a file in `_posts/` named `YYYY-MM-DD-your-slug.md`.
2. Add the required front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
tags: [tag1, tag2]
---
```

3. Write content in Markdown. Use `$...$` for inline math and `$$...$$` for display math.
4. Commit and push — GitHub Pages builds automatically (≈ 1 min).

## Local Development

```bash
# Full Jekyll preview (requires Ruby + Bundler)
cd xugy07-code.github.io
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

## Deployment

Push to the `main` branch of this repository. GitHub Pages detects the push, runs Jekyll, and publishes the site automatically.

---

© Guangyue Xu
