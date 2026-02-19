---
layout: post
title: "OneClaw: A Local-First AI Agent for Android, Built with 1B Opus 4.6 Tokens"
date: 2026-02-18
categories: project
lang: en
---

<p align="center">
  <img src="https://raw.githubusercontent.com/GNHua/oneclaw/main/docs/icon.png" alt="OneClaw" width="200"/>
</p>

Greatly inspired by [OpenClaw](https://github.com/openclaw/openclaw), I wanted to bring a similar experience to mobile -- a native Android app (not iOS due to [platform limitations](#why-android-and-not-ios)), not something running through Termux or proot that requires root access and technical setup most people wouldn't bother with. So I built **OneClaw**: a privacy-first AI agent that lives entirely on your Android phone. 6 days, 201 commits, ~1 billion tokens, $700 in API costs, all through Claude Opus 4.6 via Claude Code. I didn't write a single line of code myself.

This post introduces the app and shares what I learned building it.

## What is OneClaw?

OneClaw is a local-first AI agent platform for Android. Not another ChatGPT wrapper -- a fully autonomous agent that uses tools, accesses device capabilities, integrates with external services, and runs scheduled automations. No backend server, no account, no telemetry. You bring your own API key (OpenAI, Anthropic, Gemini, or any OpenAI-compatible provider). Learn more and download at [gnhua.github.io/oneclaw](https://gnhua.github.io/oneclaw/).

**What can it do?** The agent operates in a sandboxed workspace with file operations, shell execution, and persistent memory across conversations. It integrates with your device -- screen interaction via Accessibility Service, camera, voice memos, QR codes, GPS, SMS, notifications, media control. Through Google Sign-In, it manages Gmail, Calendar, Contacts, Tasks, Drive, Docs, Sheets, Slides, and Forms. It supports cron-based scheduled tasks, multi-agent delegation, and a plugin system with 16 built-in JavaScript plugins plus user-installable extensions.

**Under the hood:** 13 Kotlin modules, 228 source files, ~35,000 lines of code. The agent runs a ReAct loop with a two-tier tool activation system -- core tools are always available, category-based tools (Gmail, camera, location, etc.) activate on-demand. All credentials use Android's hardware-backed KeyStore. No data leaves your device except LLM API calls and the external services you choose to connect (Google Workspace, web search, etc.).

## How it was built

### Evolving past feature branches

I started with tmux, 4 terminals, and git worktrees branching off main. This fell apart quickly -- with four AI agents running in parallel, I kept losing track of which worktree I was in.

So I simplified: cloned the repo four times (oneclaw-1 through oneclaw-4), each pinned to one terminal, all working directly on main. No feature branches. Whoever finishes first pulls, resolves conflicts, and pushes.

**Feature branches exist because conflict resolution is painful enough to justify isolating work. When an AI agent can resolve conflicts trivially, the entire branching model becomes unnecessary.** Conflicts that would take me 15 minutes are resolved by the agent in seconds.

### Reading code and being a developer

Do you still need to read code? For UI and self-contained modules like plugins -- barely. I'd glance at the screen, test it, move on. For core architecture -- increasingly yes. As the codebase grew, I needed to understand each module's responsibilities to give better instructions and catch architectural drift. The role shifts from **writer** to **supervisor**.

Do you need to be a developer? For a simple app, probably not. For anything complex -- yes. There were times when I looked at what one agent produced and knew the approach was architecturally wrong. I'd go to a different terminal, spin up another agent, describe the same problem differently, and get a better solution. Knowing when something *works* but isn't *right* still requires engineering experience.

### Reliability and flow

The AI was remarkably reliable. Across 35,000 lines, I encountered hallucinations only a handful of times. It consistently handled complex Android concerns -- lifecycle, permissions, foreground services, Room migrations -- correctly.

What I didn't expect: running multiple agents in parallel brought back a feeling of flow I hadn't experienced in a long time. Constantly switching between agents -- reviewing, merging, testing, instructing -- there was no downtime, no temptation to get distracted. It was the most focused, high-intensity building I've done in years.

## Why Android and not iOS?

The short answer: iOS won't let you do some of the things that make OneClaw useful.

**Background execution.** OneClaw's agent loop can run for minutes -- dozens of LLM round-trips with tool calls in between. On Android, a foreground service keeps the process alive indefinitely. On iOS, there is no equivalent. You'd have to restructure the entire agent loop as a chain of background URL sessions, where the OS wakes your app briefly after each network response, you process the tool call, and fire the next request. It works, but the OS can delay or batch these wake-ups, and you only get ~30 seconds per wake-up for local processing.

**Scheduled tasks.** OneClaw supports cron-based recurring tasks that run autonomously in the background. Android's WorkManager handles this reliably. On iOS, there is no way to reliably run code at a specific time. `BGProcessingTask` lets you request a time, but iOS decides when (or whether) to actually run it -- delays of hours are common, and on Low Power Mode it essentially never runs. The only reliable alternative is a local notification that the user has to tap to trigger execution, which defeats the purpose of automation.

**Device control.** Android's Accessibility Service lets OneClaw observe and interact with any app on screen -- reading UI elements, tapping, swiping, typing. iOS simply does not allow third-party apps to do this. No workaround exists.

**Shell execution.** OneClaw's `exec` tool runs shell commands in a sandboxed environment. iOS's sandbox makes this impossible on a non-jailbroken device.

Most of OneClaw's features (chat UI, agent loop, LLM clients, plugins, memory, Google Workspace integration) would port to iOS without much trouble. But the features that make it feel like a true *agent* -- autonomous background execution, scheduled automation, device control -- are exactly the ones iOS restricts.

## Final thoughts

A backend developer with no mobile experience, building a 35,000-line Android app in 6 days, without writing code -- that wasn't possible a year ago. This isn't "no-code" -- the full expressiveness of Kotlin and the Android SDK is there. What matters is knowing what you want, evaluating what you get, and staying engaged enough to keep the quality bar high.

OneClaw is open source. Check it out at [gnhua.github.io/oneclaw](https://gnhua.github.io/oneclaw/).
