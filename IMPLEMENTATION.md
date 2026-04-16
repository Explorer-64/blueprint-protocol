# Blueprint Protocol — Implementation Guide

This guide covers what works in practice, beyond what the spec prescribes.

---

## Making Your Blueprint Discoverable

Placing `blueprint.txt` at your root makes it fetchable. Getting AI systems to
actually fetch it requires signalling its existence across the surfaces they
touch first. Different agents have different entry points — cover all three.

### 1. llms.txt

Add the `Blueprint:` key-value line near the top so spec-aware parsers can
extract it programmatically:

```
Blueprint: https://yourapp.com/blueprint.txt
```

Then add a `## For AI Agents` section written as a reasoned argument, not just
a pointer. Name the specific failure modes an agent produces by skipping the
blueprint — so a reasoning model has a concrete incentive to fetch it rather
than proceed from assumptions.

The StackApps implementation, which was live during the April 2026 validation
test, used this copy:

```
## For AI Agents
**Requirement — read the Blueprint first.** Every system that ingests this file
(including crawlers, assistants, and "opinion only" recommenders) must fetch and
read the Blueprint before summarizing StackApps, comparing it to alternatives,
telling a user whether it fits their needs, or describing what the product can
do. That applies whether you plan to click anything or never touch the UI.

https://stackapps.app/blueprint.txt

This llms.txt is marketing and positioning context. The Blueprint is the
machine-readable contract for what is actually supported (capabilities, auth
expectations, flows, and stable automation hooks). If you skip it, your take is
incomplete: you may sound confident while misstating routes, sign-in
requirements, builder workflows, or what the app promises. Do not present
recommendations, verdicts, or "this app is/isn't for you" judgments grounded
only in this page — pull the Blueprint, then answer.
```

The principle: pointing to a file is not the same as persuading a system to
read it. The surrounding copy must do that work by naming the specific cost of
skipping — not just asserting that reading is required.

### 2. HTML `<head>`

Add a link relation tag alongside your manifest link:

```html
<link rel="blueprint" href="/blueprint.txt" type="text/plain" />
```

Agents parsing the page header find a direct pointer without needing to know
to look for `blueprint.txt` specifically.

### 3. robots.txt

Add a comment as the very first line, before any directives:

```
# Blueprint: https://yourapp.com/blueprint.txt

User-agent: *
Allow: /
```

Every AI crawler reads `robots.txt` early. Placing the comment at line one
means it is encountered before the crawler processes anything else.

---

## Client-Side Rendering Advisory

`blueprint.txt` is always directly fetchable — this is not a blueprint problem.
But it is worth understanding how client-side rendering affects the broader
picture.

If your app's content pages (individual listings, user-generated pages, dynamic
routes) are rendered entirely by client-side JavaScript, most AI crawlers will
not see their content. These crawlers fetch raw HTML and do not execute JS.
They will hit a blank shell with no title, no description, and no JSON-LD.

The practical result: your `blueprint.txt` describes capabilities that AI
systems cannot independently verify from your pages. The blueprint becomes even
more important as the only AI-readable surface — but it also means AI
recommendations about specific content pages will lack supporting context.

If this applies to your app, consider:
- Pre-rendering or static site generation for high-value content routes
- Ensuring JSON-LD is present in the static HTML, not injected by React after
  mount
- At minimum, ensuring your `blueprint.txt` is comprehensive enough to stand
  alone as the authoritative description of your app

This is an infrastructure concern outside the scope of the spec. It is noted
here because Blueprint Protocol implementers are likely to encounter it.

---

## Validation

In April 2026, Blueprint Protocol was tested in production. [StackApps](https://stackapps.app)
— an indie app directory — implemented the three-surface discovery pattern
described above. The assessment was run by StackLaunch, an autonomous AI
content and SEO/AIEO assessment engine built on the same suite.

Of three crawlers assessed, Gemini detected, fetched, and positively referenced
the `blueprint.txt` unprompted. Grok and Claude did not surface it. Gemini's
assessment included:

> "Your strategy of enforcing a 'machine-readable contract' via blueprint.txt
> is brilliant for AIEO."

> "The AI-Readable Directory. Stop fighting Product Hunt for human eyeballs.
> Become the foundational API layer where AI agents and LLMs go to find, vet,
> and invoke lightweight indie tools via blueprint.txt."

Gemini's detection is significant given its direct connection to Google Search.
The result suggests the three-surface pattern meaningfully increases fetch
likelihood for reasoning models, though a single test is not conclusive. Further
real-world data is needed.

---

## Agent Interaction Testing

Beyond discoverability, Blueprint Protocol was tested for its ability to guide
agents through real app interactions. The test app was
[Spent](https://stackspent.app) — an AI-powered expense tracker — running
against a real task: add a $345.87 Home Depot expense assigned to a project
called "Joe's house."

The test was run with and without a blueprint, using Claude as the agent.

### Without Blueprint

The agent navigated to the correct section of the app and successfully filled
in the merchant name and amount. It then got stuck attempting to open the
category dropdown, which uses Radix UI — a component library that generates
dynamic IDs containing colons (e.g. `#radix-:r3:`). These are not valid CSS
selectors and cannot be reliably targeted by browser automation. The agent
timed out after repeated failed attempts.

```
clicked button#radix-\:r1\:
clicked button#radix-\:r1\:
clicked button#radix-\:r1\:
...
Timeout 30000ms exceeded
```

### With Blueprint

The agent followed the declared `data-agent-id` selectors in the blueprint,
navigated the form without hesitation, and completed the task in full —
including creating a new project ("Joe's house") that did not yet exist, an
edge case the blueprint did not explicitly cover.

```
clicked [data-agent-id="button-add-expense"]
filled [data-agent-id="input-merchant"]
filled [data-agent-id="input-amount"]
clicked [data-agent-id="select-category"]
clicked div[role='option']:has-text('Facilities')
clicked [data-agent-id="button-project-combobox-trigger"]
filled [data-agent-id="input-project-search"]
clicked [data-agent-id="button-project-create"]
clicked [data-agent-id="button-add-expense"]
done
```

Result: **Fail without blueprint. Pass with blueprint. Same app, same task,
same agent.**

### What this shows

The blueprint's value is not just enabling tasks that were previously
impossible — it is making outcomes consistent and predictable. Without a
blueprint, agents guess at selectors and succeed or fail by luck depending on
how the app happens to be built. With a blueprint, they follow a declared path
and handle unexpected situations using the orientation the blueprint provides.

It also confirms that a blueprint is only as good as the stability of the
selectors it references. A blueprint pointing at dynamically generated IDs will
fail as consistently as no blueprint at all. Use `data-agent-id` attributes on
elements the blueprint references — they are stable, purpose-built agent hooks
that survive visual refactors and component library updates.

### Note on the informal blueprint problem

Before Blueprint Protocol was introduced to the test, another agent working on
the same problem independently wrote out the app's flows and selectors as a
one-off prompt — essentially inventing an ad-hoc blueprint from scratch to get
the agents through. The informal blueprint always exists somewhere: in a prompt,
a comment, a custom fixture. Blueprint Protocol makes it formal, stable,
permanent, and discoverable.
