# Denkstein.github.io

Blog served by GitHub Pages at https://denkstein.github.io.

## Writing a post

Add a file to `_posts/` named `YYYY-MM-DD-slug.md` starting with:

```yaml
---
layout: post
title: "Post title"
date: YYYY-MM-DD
---
```

Pushing to `main` publishes it.

## Local preview (optional)

Requires Ruby and Bundler:

```sh
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000.
