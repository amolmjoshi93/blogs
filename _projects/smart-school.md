---
title: Smart School
order: 1
kicker: Multi-tenant school management platform
status: Live
icon: https://amolmjoshi93.github.io/assets/icons/school.webp
live_url: https://managemysmartschool.co.in
live_label: managemysmartschool.co.in
tags: [Rails 8.1, Vue 3, PostgreSQL, Hotwire, Tailwind 4, Solid Queue, Kamal, Android]
description: "Smart School: a multi-tenant school management platform built with Rails 8.1 and Vue 3. Engineering notes on its performance work and design decisions."
---

One deployment serves many schools, each on its own subdomain with its own branding, roles, data and billing.
It covers admissions, attendance, timetabling, homework, exams and report cards, fee collection and parent
communication, with a lighter mode for solo tutors.

- A Rails JSON API with a Vue 3 single-page app on top, plus an Android app.
- Role-based access enforced on every action, endpoint and dashboard widget.
- Online fee payments, GST invoicing, and PDF receipts, ID cards and report cards.
- AI lesson plans and homework feedback that a teacher reviews before students see them.
- About 14,000 automated tests, run before every push.
