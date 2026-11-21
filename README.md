# Amol Joshi — Engineering notes

Published at **https://amolmjoshi93.github.io/blogs/** by GitHub Pages (Jekyll, custom layouts). GitHub builds
the site on every push to `main`; there is no build step to run.

The look comes from the portfolio itself: `_layouts/default.html` loads
`https://amolmjoshi93.github.io/assets/style.css` and its fonts, and `assets/blog.css` adds only what posts
need (long-form text, tables, code, the contact card). Change the portfolio's style and the blog follows.

## Adding a post

1. Create `_posts/YYYY-MM-DD-short-title.md`:

   ```yaml
   ---
   title: "What changed, in a few words"
   description: "One sentence for the index and link previews."
   project: smart-school        # a file name in _projects/ (optional)
   tags: [performance, deploys]
   ---
   ```

2. Write it. Start from a measurement, end with one, and keep a "what we didn't do" section when an option was
   ruled out. The contact card and previous/next links are added by the layout.
3. Push to `main`. The site updates in a minute or two.

## Adding a project

Create `_projects/<slug>.md` with `title`, `order`, `kicker`, `status`, `icon`, `live_url`, `tags` and a short
description in the body. Its page lists every post whose `project:` is that slug.

## Before publishing

The source of these products is private and this blog is public. Do not include:

- credentials, tokens, keys, server addresses or internal host names, or how the infrastructure is laid out;
- names or data of real schools, students, parents or staff (use made-up examples);
- unreleased or not-yet-public product lines.

## Local preview (optional)

```bash
bundle install
bundle exec jekyll serve --baseurl /blogs
```
