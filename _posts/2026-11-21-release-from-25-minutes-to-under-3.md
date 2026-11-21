---
layout: post
title: "From a 25-minute release to under 3: Docker layer caching, one image for two environments, and a 1.96 GB image"
description: "Our release built the same code twice and reinstalled every gem every time. Moving one environment variable, proving staging and production builds were byte-identical, and deleting 27 unused PDF binaries took a release from ~25 minutes to ~2.5."
image: /assets/covers/2026-11-21-faster-release.png
image_alt: "Abstract cover: a stack of image layers, mostly cached, promoted to two environments"
project: smart-school
tags: [performance, deploys, docker, kamal]
---

A Smart School release means deploying staging, checking it, then deploying production with the same code.
Production only goes out once staging has been checked, so the two deploys happen one after the other.

In November a release took about **25 minutes**: 11–15 for staging, then the same again for production. This
post covers how it got to **about 2.5 minutes**, plus one idea from the plan that didn't survive contact with the
tools.

## How a deploy works here

We deploy with [Kamal](https://kamal-deploy.org/). For each environment it:

1. **Builds** a Docker image natively on a remote ARM build host (cross-building ARM on an x86 laptop was slow
   and ran out of memory),
2. **Pushes** it to a container registry,
3. **Pulls** it back, starts the new container, waits for the health check and switches traffic,
4. Runs **hooks** before and after.

Docker caches each step of the build. If a step and everything before it are unchanged, it reuses the saved
result (`CACHED`) instead of running it again. Most of this story is about getting more steps to hit that cache.

## Where the time went

A staging deploy, step by step:

| Step | Time | Note |
|---|---|---|
| `bundle install` | 165 s | on **every** deploy, even with an unchanged `Gemfile.lock` |
| Export + push to the registry | 168 s | |
| Boot, health checks | ~180 s | |
| `chown -R` in the final stage | 29 s | also wrote a second copy of several folders into the image |
| SPA build, `npm ci`, assets | ~60 s | |

Then production did all of it again.

## Problem 1: one variable at the top of the Dockerfile

```dockerfile
FROM ruby:4.0-slim AS base
ARG RAILS_ENV_ARG=production
ENV RAILS_ENV="$RAILS_ENV_ARG"   # ← staging and production pass different values
...
RUN bundle install               # ← sees RAILS_ENV, so it's a different step per environment
RUN npm ci
```

Docker's cache key for a step includes everything defined before it. Because `RAILS_ENV` was set at the top and
differed between staging and production, **every** later step had a separate cache per environment. The gems
installed for staging could never be reused for production, and the reverse.

Then a cleanup hook trimmed the build cache to 10 GB after each deploy, which evicted what little was left. So
every deploy installed every gem from scratch.

**The fix** was to set `RAILS_ENV` *after* the dependency installs, just before the first step that reads it:

```dockerfile
COPY Gemfile Gemfile.lock ./
RUN --mount=type=cache,id=bundle,target=/usr/local/bundle/ruby/4.0.0/cache \
    bundle install
COPY frontend/package.json frontend/package-lock.json frontend/
RUN --mount=type=cache,id=npm,target=/root/.npm \
    cd frontend && npm ci --prefer-offline

ENV RAILS_ENV=production   # ← only now
COPY . .
```

The `--mount=type=cache` lines are a second, smaller win. Downloaded gems and npm packages stay on the build
server between builds, outside the image, so when a lockfile *does* change, only the changed packages are
downloaded. One detail: don't `rm -rf` a cache mount's directory in the same step, or you've just emptied the
cache you meant to keep.

We also raised the build-cache trim to 20 GB and made it run after production deploys too, so the layers both
environments now reuse stay cached instead of being evicted after every deploy.

## Problem 2: a bigger image than needed

**The `chown -R` layer.** The final stage copied the app in as root, then ran
`chown -R rails:rails db log storage tmp public`. Changing a file's owner rewrites it, so that step stored a
**second copy** of `public/` (the built SPA and all assets) in a new layer, and every push uploaded it.

Instead, the build stage moves the folders the app writes to into their own directory, and the final stage
copies them with the right owner in one step:

```dockerfile
# build stage, at the end
RUN rm -rf frontend && mkdir /rails-writable && \
    mv db log storage tmp public /rails-writable/

# final stage
RUN groupadd --system --gid 1000 rails && useradd rails --uid 1000 --gid 1000
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /rails /rails                                   # code: root-owned, read-only to the app
COPY --from=build --chown=rails:rails /rails-writable/ /rails/    # writable folders: owned by the app
```

The security posture is unchanged: the app user still can't modify application code. We checked that on the
running container.

**Shipping things the server never reads.** The SPA's source code and `node_modules` were still in the final
image. The server only serves the built output, so they're now deleted after the build.

Image: **3.11 GB → 2.84 GB.**

## Problem 3: building the same thing twice

Production rebuilt from scratch what staging had just built. The plan was to **build once and promote**:
build for staging, then tell production to deploy the same image without building
(`kamal deploy --skip-push`).

That only works if the two builds really are the same. So we tested it. In a clean checkout we compiled the
Rails assets and the SPA once as staging and once as production, then compared every file:

```
$ diff -rq staging/ production/
$ diff -rq staging/ production/ | wc -l
0
```

22 MB of output, **zero differing files**. The only build setting that differed was the environment's name,
and nothing in the build used it. So the image no longer has an environment baked in. It always builds as
`production`, and each environment sets its real `RAILS_ENV` when the container starts (Kamal already did that).

### The idea that didn't survive

When we tried `--skip-push` promotion, Kamal refused. It stamps every image it builds with a label naming its
service (`service=smart_school_staging`). After pulling an image it checks that label, and it won't boot one
built for another service. Sharing one image name wouldn't help either: whichever environment pushed last would
overwrite the other's tag.

But once the build no longer depends on the environment, **production doesn't need to skip the build**. When
production builds a commit staging already built, every step finds its saved result. A label is image
*metadata*, not a layer, so it doesn't break the cache. Production's "build" now takes seconds, and its push is
mostly "already exists".

### A guard: production only runs what staging runs

"Promote what you tested" is a rule, and rules need enforcing. The `pre-deploy` hook now asks Kamal which commit
staging is running and stops a production deploy of anything else:

```sh
if [ "$KAMAL_DESTINATION" = "production" ] && [ "$ALLOW_UNSTAGED_PRODUCTION" != "1" ]; then
  case "$KAMAL_COMMAND" in
    deploy|redeploy)
      staging_version=$(kamal app version -d staging -q | grep -Eo '[0-9a-f]{40}' | head -1)
      [ "$staging_version" = "$KAMAL_VERSION" ] || { echo "Staging runs $staging_version"; exit 1; }
      ;;
  esac
fi
```

Two details matter:

- **Rollbacks always pass.** `kamal rollback` runs the same hook, and the one time you must not be blocked is
  when you're rolling back. Kamal passes `KAMAL_COMMAND` to hooks, which is how the hook tells the two apart.
- **There's an escape hatch** (`ALLOW_UNSTAGED_PRODUCTION=1`) for the day staging itself is down and production
  needs a fix.

We tested all five cases against the live deployment: same commit passes, different commit is blocked, rollback
passes, override passes, staging is unaffected.

## Problem 4: 27 PDF programs we never run

Receipts, ID cards, report cards and invoices are PDFs, made with wkhtmltopdf via the `wkhtmltopdf-binary` gem.
That gem ships a gzipped wkhtmltopdf for **every OS and CPU type it supports**: 28 files, about 16 MB each.
After all the work above, it was **436 MB of the 787 MB** of installed gems.

Our image runs exactly one of them: Debian on ARM. So the build keeps that file and deletes the rest, and it
fails loudly if the base image ever changes to an OS we haven't mapped:

```dockerfile
RUN set -e; . /etc/os-release; \
    case "${ID}_${VERSION_ID}" in \
      debian_12*|debian_13*) os=debian_12 ;; \
      *) echo "no wkhtmltopdf mapping for ${ID} ${VERSION_ID}"; exit 1 ;; \
    esac; \
    keep="wkhtmltopdf_${os}_$(dpkg --print-architecture).gz"; \
    dir=$(ls -d /usr/local/bundle/ruby/*/gems/wkhtmltopdf-binary-*/bin); \
    [ -f "$dir/$keep" ] || { echo "missing $keep"; exit 1; }; \
    find "$dir" -name 'wkhtmltopdf_*.gz' ! -name "$keep" -delete
```

The failure is the point. Without it, a future base-image upgrade would build fine and then quietly break every
fee receipt. With it, the build breaks, which someone notices immediately. The app's PDF setup and the
Dockerfile each have a comment pointing at the other, so the mapping is changed in both places.

We didn't trust "it's just deleting unused files". On staging, we cleared the extracted binary, let the app
unpack the one remaining file, and then downloaded a real fee receipt, a class ID-card sheet with QR codes and a
class timetable through the app. We rendered each page and checked it by eye, including the ₹ symbol and
Devanagari text.

Image: **2.84 GB → 1.96 GB.** Installed gems: **787 MB → 367 MB.**

## The scoreboard

| | Before | After |
|---|---|---|
| Staging deploy, code changed, dependencies unchanged | 11–15 min | **~3 min** (178 s measured) |
| Staging deploy, nothing in the image changed | 11–15 min | **65 s** |
| Production deploy (same commit as staging) | 10–14 min | **79 s** (push: 10 s) |
| A full release | ~25 min | **~2.5–4 min** |
| Image size | 3.11 GB | **1.96 GB** |

## What we decided not to do

- **Deploy staging and production in parallel.** Production deploys only what staging has run and been checked
  on; doing both at once would defeat the point of staging.
- **Run a self-hosted registry instead of a hosted one.** That was the plan's last phase, meant to save upload
  time. Production's push now takes 10 seconds, so there's little left to save. The hosted registry also keeps an
  off-site copy of every release, which is what you roll back from if a machine is ever lost.
- **Build on a laptop.** The images are ARM; cross-building was slower and ran out of memory.

## Why the design holds up

All of this rests on one rule: **one image per commit, nothing environment-specific inside it, promoted from
staging to production.** That rule doesn't depend on how the infrastructure is laid out:

- **Branches don't matter to Kamal.** It builds the checked-out commit and tags the image with its hash. Whether
  production follows `main`, a release branch or tags, the guard still checks "same commit as staging".
- **Hosts don't matter either.** Any host pulls the image from the registry; nothing in the image ties it to a
  machine or an environment. That's one more reason to keep a hosted registry.
- **The one thing that breaks it is a different CPU type.** An x86 host can't run ARM images. Keep every
  environment on the same architecture, or accept a multi-architecture build.

## Takeaways

- **In a Dockerfile, order is a cache key.** An environment variable set too early made every deploy reinstall
  every gem.
- **Prove "these builds are the same" with `diff`, not by reading config.** It took two minutes and settled the
  biggest design question.
- **Read how your deploy tool treats images before designing around it.** Kamal's service label ruled out the
  plan's headline idea, and a different route reached the same result.
- **When you delete things, make the build fail loudly if your assumption ever stops being true.**
