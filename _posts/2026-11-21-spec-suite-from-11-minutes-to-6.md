---
layout: post
title: "Our test suite went from 64 seconds to 11 minutes. Here's how we took 4 minutes back."
description: "Factory setup was 72% of the run. Sharing one pre-seeded school per worker, a bulk role seed and runtime-balanced workers took the full suite from 10:49 to 6:41, with no tests deleted."
image: /assets/covers/2026-11-21-spec-suite.png
image_alt: "Abstract cover: a grid of passing tests, six worker bars finishing together, and a stopwatch"
project: smart-school
tags: [performance, testing, rspec]
---

Smart School's test suite is the only gate between a change and a deploy. We stopped running RSpec on CI to
save Actions minutes, so every push waits for the local parallel run. In July that run took about **64
seconds**. By November it took about **11 minutes**.

This post is about how we found out why, and how we got it to **6 minutes 41 seconds** without deleting a single
test.

## The first surprise: it wasn't the number of tests

The obvious explanation was growth. It was only part of the story:

| | July 2026 | November 2026 |
|---|---|---|
| Spec files (excluding browser specs) | 352 | 1,230 |
| Examples | ~5,200 | ~13,900 |
| Full run, 6 workers | ~64 s | ~11 min |

The suite grew **2.7×**. The run time grew about **10×**. So the problem was mostly that **each test had become
slower**, not that there were more of them.

## Measure before guessing

We took a 60-file sample (854 examples) and ran it serially with profiling. It took 142 seconds, and **103 of
those seconds (72%) went to factory setup**: creating the records each test needs before it can run.

| Factory | Time per call | SQL queries per call |
|---|---|---|
| `teacher` | 127–210 ms | 149 |
| `admin` | 77–170 ms | 151 |
| `student` | 96–140 ms | 147 |
| `school` | ~14 ms | 15 |

Creating one teacher ran **149 SQL queries**. The reason was a chain of reasonable decisions:

1. Every user belongs to a school, so `create(:teacher)` without a school built a **new** school.
2. The first user in a new school seeds that school's **default roles and permissions** (seven roles, hundreds of
   permission grants). Real sign-ups need this, so the factory did it too.
3. The seeding job looked up **each permission grant with its own query** (~63 per school), found each role by
   name separately, and re-ran a "school setup prerequisites" check after **every** role it saved.

A test that created a teacher, a student and an admin therefore built three schools and seeded roles three
times, just to check one permission.

Two smaller costs sat on top:

- **Uneven workers.** Tests were split across the six workers by file size. The workers finished anywhere from
  8:33 to 10:56, and a parallel run is only as fast as its slowest worker.
- **Boot under load.** Each worker took ~22 s to load when all six started together (6 s alone).

## Step 1: make the seed cheaper (it helps real sign-ups too)

The role-seeding job now reads the permission catalogue **once per run** and indexes the school's roles once,
instead of a lookup per grant. That took a user factory from **149 to 71 queries**.

This wasn't a test-only fix. The same job runs when a real school signs up, so real sign-ups got faster too.

We also switched parallel_tests to record each file's runtime and **balance workers by recorded time** instead
of file size.

Result: **~11 min → 10:49.** Almost nothing. Each school's seed was cheaper, but tests still made a school for
every user. The fix had to go after the *number* of schools.

## Step 2: one school per test, not one per user

We changed the user factory so that, unless a test passes a `school:`, every user in that test **shares one
school**. A test that needs two different schools still asks for them explicitly
(`school: create(:school)`).

Before shipping it, we checked what relied on the old behaviour. Only **one** spec depended on two bare users
getting two different schools, and the specs that test isolation between schools already passed `school:`
explicitly.

| | Before | After |
|---|---|---|
| Creating a teacher/student/admin | 200–300 ms, ~80 SQL | ~23–25 ms, 17–25 SQL |
| Full run | 10:49 | **9:15** |

## Step 3: one school per worker, and a seed in two statements

Profiling again showed role seeding was **still 37%** of a sampled run. The source was 532 spec files that
declare their own `let(:school) { create(:school) }`, which step 2 couldn't reach.

Two changes:

**A school built once per worker.** Before the suite starts, each worker builds one school with its roles and
permissions already seeded and commits it, outside the per-test transactions. Each test runs inside a
transaction that rolls back whatever it did to that school, so tests still can't see each other's changes. The
only fallout was three specs that asserted an absolute count (`School.count == 1`); they now assert a change
(`change(School, :count).by(1)`) or scope by school.

**A seed that's two statements, not eighty.** All roles go in with one `insert_all`. All of their permission
grants go in with **one** `INSERT … SELECT unnest(...)`. An earlier version used `insert_all` for the ~900 grant
rows too, but casting 900 rows in Ruby alone cost ~60 ms; letting PostgreSQL expand arrays was far cheaper.

| Role seed per school | Before | After |
|---|---|---|
| Time | ~150 ms | **~34 ms** |
| SQL queries | ~80 | **26** |

Real sign-ups got this too.

We also turned on **YJIT** in the test environment, since production already runs it. It saved only about 1.5%
on the full run, but it costs nothing.

Result: **9:15 → 6:59**, and the six workers now finish **within 12 seconds of each other**.

## Step 4: set up once per file where it's safe

For specs that only *read* their setup (index pages, "this role gets a 403" checks), there's no reason to rebuild
the same records for every example. [test-prof](https://test-prof.evilmartians.io/)'s `let_it_be` and
`before_all` build them once per file.

- One permission-search spec: **20.7 s → 6.8 s**.
- A shared context used by 19 files (a school plus eleven people and their reporting lines) is now built once
  per file, with `refind: true`, so an example that edits a record never leaks the edit into the next one.
  Those files: **157 s → 101 s** combined.

Full run: **6:51 → 6:41**.

Then we stopped converting files one by one, because the profile said to. Across the whole suite, factories are
still about half of every worker's time, but **spread thin**: the worst file spends 12 seconds in them, and the
top 30 files together about 220 s out of ~1,170 s. Converting more is diminishing returns. We'll take files as
they get flagged (next section) instead of sweeping.

## Step 5: keep it fast

A faster suite slowly becomes a slow one again. So `bin/spec_budget` runs after every suite:

- A **new** spec file fails the check if it takes more than **10 s** (plus 25% headroom).
- Files that are already slow are on a list with a **ceiling**. The list is a ratchet: it can only go down, and
  new files can't be added to it.

The headroom came from experience. Two green runs each flagged a *different* pair of files at 10.0–10.7 s,
because parallel timings move about ±10% from run to run. A hard 10-second line would have failed pushes at
random, and a check that fails at random gets ignored.

The suite's wall time is also printed after every pre-push run, so a regression shows up the day it lands.

## What we didn't do

- **Run more workers.** Six is our agreed ceiling for the machine the suite runs on.
- **Put RSpec back on CI.** That trade was made on purpose.
- **Skip or delete slow tests that catch real bugs.** The fix was cheaper setup, not less coverage.
- **Delete the legacy-screen tests yet.** About 121 request-spec files cover classic server-rendered screens
  that most users no longer reach. We inventoried them: only ~24 s of wall time sits in the ones that have a
  modern API equivalent. Deleting them is a small win and a product decision, so it waits for sign-off, file by
  file.

## The scoreboard

| Stage | Full run |
|---|---|
| Start (November) | ~11 min |
| Cheaper seed, balanced workers | 10:49 |
| One school per test | 9:15 |
| One school per worker, bulk seed, YJIT | 6:59 |
| Shared setup once per file | **6:41** |

About 14,340 examples, all green, nothing deleted. Real school sign-ups got faster along the way.

## What we'd tell our past selves

- **Profile before optimising.** Our guess ("too many tests") was wrong by a factor of four.
- **Look at the factories first.** A factory that quietly builds a whole tenant is the most expensive line in
  most Rails suites.
- **A fix that only helps tests is half a fix.** The best ones here made production faster as well.
- **Put a budget on it,** with enough headroom that it only fails when something is actually wrong.
