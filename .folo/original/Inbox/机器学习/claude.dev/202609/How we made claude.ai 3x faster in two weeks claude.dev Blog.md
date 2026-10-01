---
title: "How we made claude.ai 3x faster in two weeks / claude.dev Blog"
source: "claude.dev"
category: "Inbox"
group: "机器学习"
author: "Raymond Wang"
url: "https://claude.dev/blog/how-we-made-claude-ai-faster/"
published: 2026-09-23T08:00:00+08:00
saved: 2026-10-01T09:53:35+08:00
folo_key: "url::https://claude.dev/blog/how-we-made-claude-ai-faster"
tags:
  - "folo"
  - "Inbox"
  - "claude.dev"
---

# How we made claude.ai 3x faster in two weeks / claude.dev Blog

> [!info] claude.dev · Inbox · 2026-09-23 08:00 · [原文](https://claude.dev/blog/how-we-made-claude-ai-faster/)

## THE BRIEF

Before the sprint, we created a Slack channel with the following [standing instructions](https://claude.com/docs/claude-tag/users/getting-started#give-claude-standing-instructions):

> @Claude Your job is to facilitate all things related to the performance of the claude.ai website and desktop app. Your responsibilities include monitoring deploys for performance regressions, assessing the accuracy and comprehensiveness of existing telemetry, maintaining well-curated observability dashboards, proactively implementing solutions for observed issues and low-hanging fruit, proposing performance project opportunities, and communicating with your human teammates. \[…\]
> 
> The ultimate goal for this channel is for you to become as autonomous as possible, but today we know that isn’t yet possible.

We asked Claude to analyze usage data through the Datadog MCP server. It identified the four highest-impact user journeys: launching the app, starting a conversation, loading an existing conversation, and sending a message. Between web and desktop, and across our products, those journeys came to thirteen distinct measurements. To establish baselines, we added instrumentation until they were directly comparable: each started with a user interaction, ended once the result was rendered, and disambiguated client and server work.

We kicked off the sprint with a list of about twenty hand-picked projects, each targeting a specific journey. Claude estimated the impact of each project in milliseconds, and we aggregated those estimates to set our targets for the sprint. Some of the projects were fairly large, but we thought we could probably achieve most of them within two weeks.

We hit twelve of the thirteen targets by day three.

The planned projects landed early. For faster launches, we baked a static composer into the HTML so users can type during React initialization, and precompiled a V8 code cache so the desktop shell’s main process doesn’t recompile from scratch. For faster navigations, we kept the composer mounted between conversations, prefetched sessions when the user hovered over them, and cut sidebar re-renders by 90%.

We had also left room for Claude to identify opportunities and propose new workstreams. Those workstreams quickly ramped into full projects of their own, which far exceeded our initial targets. So we set new targets, then looked for more things to measure:

> @Claude we’ve ended up funding nearly every project in the original projects list and more. let’s do a refresh \[…\] what have we not explored, what can we hill climb on, where is the most opportunity at this point? \[…\] i am open to WACKY ideas
