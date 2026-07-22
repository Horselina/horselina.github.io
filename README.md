# horselina.github.io

Experiments on Android kernel security, mistakes and learnings.

Source for the site published at **<https://horselina.github.io>**, built by GitHub Pages
with Jekyll and the bundled `minima` theme.

## Layout

```
├── _config.yml              # site config, theme, permalink scheme
├── index.md                 # homepage
├── about.md                 # /about/
├── _posts/                  # all articles, YYYY-MM-DD-slug.md
├── blog/
│   ├── index.md             # /blog/ — all posts + category list
│   └── android/index.md     # /blog/android/ — category listing
├── assets/images/<topic>/   # per-post images
└── Gemfile                  # local preview only; ignored by GitHub Pages
```

Posts live in `_posts/` but are published under `/blog/<category>/<slug>/` via the
`permalink` setting in `_config.yml`.

## Adding a post

1. Create `_posts/YYYY-MM-DD-slug.md` with front matter:

   ```yaml
   ---
   layout: post
   title: "Your title"
   date: YYYY-MM-DD
   category: android
   tags: [tag1, tag2]
   description: "One-line summary used for SEO and previews."
   ---
   ```

2. Put images in `assets/images/<topic>/` and reference them with absolute paths:
   `![alt](/assets/images/<topic>/file.png)`.
3. If the category is new, copy `blog/android/index.md` to `blog/<category>/index.md`
   and update `title`, `permalink`, and the `site.categories.<name>` reference.

**Note:** a post dated in the future will not be built. Keep `date` at or before today.

## Local preview

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://127.0.0.1:4000>.
