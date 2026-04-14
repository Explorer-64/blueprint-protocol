# Blueprint Protocol

**A machine-readable standard for teaching AI agents how to use your web app.**

---

## The Problem

`llms.txt` tells AI what your site *is*.  
`sitemap.xml` tells crawlers what pages *exist*.  
`robots.txt` tells bots where they *can't go*.

But nothing tells an AI agent how to actually **do** something inside your app.

So agents guess. They hallucinate button IDs, click the wrong things, and fail silently. The result is frustrated users, broken automations, and AI assistants that look incompetent — even when the underlying model is excellent.

---

## The Solution

A `blueprint.txt` file at the root of your web app.

One plain text file. No tooling required. Any developer can write one in under 10 minutes.

---

## Minimal Example

This is all you need to get started:

```
# BLUEPRINT: Habit Tracker
# Version: 2.0.0
# URL: https://yourhabittracker.app

## IDENTITY
name: Habit Tracker
description: Build and maintain daily habits. Log completions, track streaks, and stay accountable over time.
category: productivity
contact: support@yourhabittracker.app

## AUTH
provider: firebase
method: email

## ACCESS
last-resort: ui

## CAPABILITY: log-habit
description: Mark a habit as complete for today and update the user's streak.
input:
  - name: habit-name
    type: string
    required: true
    description: The name of the habit to mark complete.
output:
  - type: confirmation
    description: Habit logged. Streak count updated.
auth-required: true
scope: form-submit

### UI
steps:
  1. ASSERT-AUTH
  2. NAVIGATE /dashboard
  3. WAIT [data-agent-id="habit-list"] (max: 10s)
  4. CLICK [data-agent-id="habit-<<habit-name>>-complete"]
  5. WAIT [data-agent-id="streak-updated"] (max: 5s)
  6. VERIFY selector_exists [data-agent-id="streak-updated"]
```

That's a complete, working blueprint. An AI agent can read that and log a habit without guessing at a single thing.

**You only add more when you have more.** If you later build an API endpoint, add an `### API` block. If you build an MCP server, add a `## MCP` block. The spec supports all of it — but none of it is required to start.

---

## Why Add a Blueprint?

**AI agents work reliably with your app.**  
Claude, ChatGPT Operator, Gemini — they are already attempting to automate tasks inside web apps, with or without your permission. Give them a map and they work correctly. Without one, they guess and your users blame your app.

**Tools that consume blueprints can serve your app automatically.**  
A blueprint-ready app can be picked up by any tool that reads the standard — demo video generators, agent frameworks, onboarding tools — without you writing custom integrations for each one.

**The barrier is low. The upside compounds.**  
A 20-line file written today makes your app more valuable to every AI agent and blueprint-aware tool that exists now and in the future.

---

## Specification

Full spec: [SPEC.md](./SPEC.md)

---

## Examples

Blueprint is designed for indie developers and small teams — not enterprise software. The two examples cover the full range:

| Example | What it shows |
|---------|--------------|
| [Habit Tracker](./examples/habit-tracker.txt) | The floor — no API, no MCP, UI only. Any app can start here. |
| [Demo Video Tool](./examples/demo-video-tool.txt) | The ceiling — MCP-first, blueprint consumer, async jobs. |

Most apps live somewhere in between. Start with the habit tracker pattern and add layers only when you have them.

---

## Core Rules

**Capabilities first, invocation second.**  
Declare what the app can do before describing how to invoke it. The capability contract stays stable even when the implementation changes.

**Only declare what actually exists.**  
No API block unless there is a real endpoint. No MCP block unless there is a real server. A blueprint with false entries is worse than no blueprint — it causes silent failures.

**`data-agent-id`, not CSS selectors.**  
UI steps use `[data-agent-id="x"]` — a dedicated attribute that won't conflict with styling or get renamed in a refactor. Add it to your HTML alongside the blueprint step that references it.

**One action per step.**  
One `CLICK`, one `INPUT`, one `NAVIGATE` per line. No combining actions. This makes flows unambiguous for agents and clean for any tool that replays them.

**WAIT after every async operation.**  
If the UI has a loading state, there must be a `WAIT` step. Agents and any tool that replays the flow need to know when to proceed.

---

## Status

**Version: 2.0.0 — Draft**

This is an open, unowned standard. Use it, fork it, extend it.  
If you build something with it, open a PR to add your example.

---

*Blueprint Protocol is an open standard. No one owns it. Everyone benefits from it.*

---

## Validation

In April 2026, Blueprint Protocol was tested in production. [StackApps](https://stackapps.app) — an indie app directory — implemented the three-surface discovery pattern (blueprint pointer in `llms.txt`, `<link rel="blueprint">` in HTML `<head>`, and `# Blueprint:` as the first line of `robots.txt`). The assessment was run by StackLaunch, an autonomous AI content and SEO/AIEO assessment engine built on the same suite.

Of three crawlers assessed, Gemini detected, fetched, and positively referenced the `blueprint.txt` unprompted. Grok and Claude did not surface it. Gemini's assessment included:

> "Your strategy of enforcing a 'machine-readable contract' via blueprint.txt is brilliant for AIEO."

> "The AI-Readable Directory. Stop fighting Product Hunt for human eyeballs. Become the foundational API layer where AI agents and LLMs go to find, vet, and invoke lightweight indie tools via blueprint.txt."

Gemini's detection is significant given its direct connection to Google Search. The result suggests the three-surface pattern meaningfully increases fetch likelihood for reasoning models, though a single test is not conclusive. Further real-world data is needed.

See [IMPLEMENTATION.md](./IMPLEMENTATION.md) for the full discovery pattern used in the test.
