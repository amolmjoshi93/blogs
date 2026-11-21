# Building Smart School — engineering notes

Published at **https://amolmjoshi93.github.io/blogs/** by GitHub Pages (Jekyll, `minima` theme). GitHub builds
the site on every push to `main`; there is no build step to run.

## Adding a post

1. Create `_posts/YYYY-MM-DD-short-title.md` with front matter:

   ```yaml
   ---
   layout: post
   title: "What changed, in a few words"
   description: "One sentence for the index and link previews."
   tags: [performance, deploys]
   ---
   ```

2. Write it. Start from a measurement, end with one, and keep a "what we didn't do" section when an option was
   ruled out.
3. Push to `main`. The site updates in a minute or two.

## Before publishing

The app's repository is private and this blog is public. Do not include:

- credentials, tokens, keys, server addresses or internal host names, or access/firewall details;
- names or data of real schools, students, parents or staff (use made-up examples);
- unreleased or not-yet-public product lines.

## Local preview (optional)

```bash
bundle install
bundle exec jekyll serve --baseurl /blogs
```
