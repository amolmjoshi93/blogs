---
layout: page
title: About
permalink: /about/
---

This blog records how Smart School is built and why.

**What gets written here**

- **Performance work:** where the time went, how it was measured, and what fixed it.
- **Decisions:** the options on the table, the one we picked, and the reason.
- **How the app evolved:** why a part is shaped the way it is, so the next change starts from the reasoning, not
  just the code.

**How the posts are written**

Every post starts from a measurement and ends with one. Claims of "faster" come with the before and after. When
an idea from the plan didn't survive contact with the real system, the post says so and explains why.

**The stack, briefly**

Rails 8 on Ruby 4 with PostgreSQL, a Vue 3 single-page app served by Rails, Solid Queue for background jobs,
and an Android WebView shell. Deployed with Kamal to a single ARM server that hosts both staging and
production.
