# My Blog

A [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll blog hosted on GitHub Pages, built and deployed via GitHub Actions (`.github/workflows/pages-deploy.yml`).

## Adding a new post

Create a file in `_posts/` named `YYYY-MM-DD-title.md` with front matter like:

```yaml
---
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS +0530
categories: [TOP_CATEGORY, SUB_CATEGORY]
tags: [tag1, tag2]
image:
  path: /assets/images/<post>/banner.jpg
  alt: description
---
```

Then write the post content below the front matter in Markdown, commit, and push to `main`. GitHub Actions rebuilds and deploys the site automatically — check progress under the repo's **Actions** tab.

Images go in `assets/images/<post-slug>/` and are referenced with root-relative paths, e.g. `/assets/images/steel-mountain/homepage.png`.
