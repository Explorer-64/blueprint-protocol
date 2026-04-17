# Blueprint Protocol — Specification v2.1.0

**Status:** Draft  
**Published:** 2026-04-13  
**Author:** Blueprint Protocol Contributors

---

## 1. Overview

A `blueprint.txt` file placed at the root of a web application (e.g., `https://yourapp.com/blueprint.txt`) tells AI agents what the app can do and how to invoke it — in priority order.

Blueprint answers three questions:

1. **What can this app do?** → `CAPABILITIES`
2. **How should an agent interact with it?** → `ACCESS` hierarchy
3. **What does it need first?** → `AUTH`

An agent reads the capability list to understand intent, then uses the access hierarchy to pick the best available method: MCP first, API second, UI as a last resort.

---

## 2. File Location and Discovery

Place the file at:

```
https://yourapp.com/blueprint.txt
```

Reference it from your `llms.txt` so AI crawlers already reading that file can follow the pointer:

```
Blueprint: https://yourapp.com/blueprint.txt
```

---

## 3. Document Structure

```
# BLUEPRINT header
## IDENTITY block
## SUMMARY block (optional — for discovery and recommendation)
## AUTH block (or reference)
## MCP block (if MCP server exists)
## ACCESS block
## CAPABILITIES block (one per capability)
```

---

## 4. Header

```
# BLUEPRINT: <App Name>
# Version: <semver>
# URL: <canonical app URL>
# Updated: <YYYY-MM-DD>
```

All four fields are required.

---

## 5. IDENTITY Block

```
## IDENTITY
name: <human-readable app name>
description: <one sentence — what does this do for the user>
category: <value>
contact: <support email or URL>
```

Category is a single value from this list: `productivity`, `finance`, `design`, `marketing`, `communication`, `developer-tools`, `ecommerce`, `media`, `legal`, `health`

Do not use slash-separated subcategories. One value only.

---

## 6. SUMMARY Block

The `SUMMARY` block is optional. It provides a concise overview of the app for
agents that need to understand what an app does without parsing every capability
in detail.

Crawlers, recommendation engines, and discovery tools read this block and stop.
Agent runtimes executing tasks read the full `CAPABILITIES` blocks. For large
apps with many capabilities, `SUMMARY` is the difference between a blueprint
that answers a discovery agent immediately and one that requires parsing hundreds
of lines before forming a useful response.

```
## SUMMARY
tagline: <one sentence — what this app does for the user>
audience: <who this app is built for>
capabilities:
  - <capability-id>: <one line — what it does>
  - <capability-id>: <one line — what it does>
  - <capability-id>: <one line — what it does>
```

List only the top 3–7 capabilities a user would reach for first. This is not an
exhaustive list — the full `CAPABILITIES` blocks cover everything. `SUMMARY` is
a curated front door for agents that want to understand before they act.

Example:

```
## SUMMARY
tagline: Snap a receipt and your expense is logged. No forms, no deadlines.
audience: Freelancers, solopreneurs, and neurodivergent users who struggle with traditional expense apps.
capabilities:
  - snap-receipt: Photograph a receipt and AI extracts and saves the expense automatically
  - manual-entry: Add an expense by filling a short form
  - export-csv: Download all expenses as a CSV for tax or accounting use
```

---

## 7. AUTH Block

Declare authentication once. Apps sharing an auth provider (e.g., a suite on shared Firebase Auth) should declare auth in a central blueprint and reference it.

### Inline auth declaration:

```
## AUTH
provider: <firebase | auth0 | custom | none>
method: <email | oauth | api-key | session>
```

### Reference to a shared auth spec:

```
## AUTH
ref: https://yourapp.com/blueprint.txt#auth
```

Using a reference means a change to shared auth requires updating one file, not every app blueprint in the suite.

---

## 8. MCP Block

If an MCP server exists for this app, declare it here so agents know how to
connect before attempting to call any tool. This block is optional — omit it
if no MCP server exists.

```
## MCP
server: <server-name>
install: <full install command the user or agent runs once>
auth: <env var name and what it contains>
```

Example:

```
## MCP
server: imagcon-mcp
install: claude mcp add imagcon -- uvx imagcon-mcp --api-key <<api-key>>
auth: IMAGCON_API_KEY — user's Imagcon API key from account settings
```

An agent that reads this block knows:
- The server name to reference when calling tools
- The exact command to install it
- Which credential is required and where to get it

Individual capability blocks then reference the tool name only:

```
### MCP
tool: <tool-name>
```

---

## 9. ACCESS Block

The `ACCESS` block declares the preferred hierarchy for agent interaction. Agents MUST attempt methods in order and stop at the first one available.

```
## ACCESS
preferred: mcp
fallback: api
last-resort: ui
```

| Value | Meaning |
|-------|---------|
| `mcp` | MCP tool definition available — use it |
| `api` | REST or GraphQL endpoint available |
| `ui` | Browser UI automation — only if no programmatic interface exists |

Only list methods that actually exist. An app with no backend API omits `fallback: api`. An app with no MCP server omits `preferred: mcp`. Do not declare a method that isn't implemented.

If a capability only supports one method, only that method appears in its definition. An agent encountering a missing method skips to the next available fallback.

---

## 10. CAPABILITIES Block

Each capability is declared independently. The capability describes **what** the app can do; the invocation blocks describe **how** to do it via each access method.

```
## CAPABILITY: <capability-id>
description: <what this does for the user>
input:
  - name: <param-name>
    type: <string | number | file | boolean>
    required: <true | false>
    description: <what this param is>
output:
  - type: <file | json | redirect | confirmation>
    description: <what the agent gets back>
auth-required: <true | false>
scope: <read-only | form-submit | file-download | account-modify | financial-transaction | destructive>

### MCP
tool: <mcp-tool-name>

### API
method: <GET | POST | PUT | DELETE>
endpoint: <path>
body:
  <param>: <<variable>>
response:
  <field>: <type>

### UI
steps:
  1. NAVIGATE <path>
  2. INPUT [data-agent-id="<id>"] <<variable>>
  3. CLICK [data-agent-id="<id>"]
  4. WAIT [data-agent-id="<id>"] (max: <N>s)
  5. VERIFY <condition>
```

Only include the invocation blocks that actually exist. A capability with only an API has no MCP or UI block.

---

## 11. UI Step Actions

UI steps are a last resort. When UI steps are required, use `data-agent-id` attributes — not `id`, `class`, or CSS selectors.

`data-agent-id` is a dedicated agent contract attribute. It:
- Does not conflict with styling or JavaScript hooks
- Signals to developers that removing it breaks the blueprint
- Remains stable across visual refactors

```html
<button data-agent-id="generate-button">Generate</button>
```

Referencing it in a step:
```
CLICK [data-agent-id="generate-button"]
```

### Available Step Actions

| Action | Syntax | Notes |
|--------|--------|-------|
| Navigate | `NAVIGATE <path>` | Waits for DOM ready |
| Input | `INPUT [data-agent-id="x"] <<var>>` | Fills a form field |
| Click | `CLICK [data-agent-id="x"]` | Clicks an element |
| Scroll | `SCROLL [data-agent-id="x"]` | Scrolls element into view before interacting |
| Wait | `WAIT [data-agent-id="x"] (max: Ns)` | Waits for element in DOM |
| Wait fixed | `WAIT <N>s` | Fixed delay — use sparingly |
| Select | `SELECT [data-agent-id="x"] "<option>"` | Dropdown selection |
| Upload | `UPLOAD [data-agent-id="x"] <<file-var>>` | File input |
| Assert auth | `ASSERT-AUTH` | Agent must be authenticated before this step |
| Verify | `VERIFY <condition>` | Flow fails if condition is false |

### VERIFY Conditions

```
VERIFY url == "/path"
VERIFY selector_exists [data-agent-id="x"]
VERIFY selector_not_exists [data-agent-id="x"]
VERIFY file_type == ".zip"
VERIFY text_contains [data-agent-id="x"] "string"
VERIFY http_status == 200
```

---

## 12. Variables

Variables are written `<<variable-name>>` and resolved at runtime from user context, prior conversation, or by prompting the user.

Standard variable names:

| Variable | Meaning |
|----------|---------|
| `<<user-email>>` | User's email address |
| `<<user-password>>` | User's password (never logged or stored by agent) |
| `<<app-name>>` | Name of the app or project |
| `<<app-description>>` | Short description of the app |
| `<<file-path>>` | A local file the user provides |
| `<<api-key>>` | An API key from the user |

### Variable Normalization in Selectors

Variables may be interpolated inside `data-agent-id` values to target dynamic elements:

```
CLICK [data-agent-id="habit-<<habit-name>>-complete"]
```

When a variable is used inside a selector, implementers MUST normalize the resolved value before constructing the attribute string. The normalization rule is:

1. Convert to lowercase
2. Replace spaces with hyphens
3. Strip all characters except alphanumeric and hyphens

Examples:

| Raw value | Normalized |
|-----------|-----------|
| `Drink Water` | `drink-water` |
| `Drink water (8oz)` | `drink-water-8oz` |
| `Morning Run 5km` | `morning-run-5km` |
| `読書` | `(strip — result is empty, do not use dynamic selectors for non-latin text)` |

App developers rendering dynamic `data-agent-id` values in HTML MUST apply the same normalization to ensure selectors match at runtime. Both sides of the contract must normalize identically or the selector will silently fail.

---

## 13. Scope Values

Scope declares the highest-risk operation a capability performs. Agents MUST NOT exceed declared scope. Agents MUST prompt the user for confirmation before executing `financial-transaction` or `destructive` scope.

| Scope | Permits |
|-------|---------|
| `read-only` | Navigation and observation only |
| `form-submit` | Fill and submit forms |
| `file-download` | Trigger a file download |
| `account-modify` | Change account settings or profile |
| `financial-transaction` | Payment, billing, or subscription |
| `destructive` | Delete or permanently modify data |

---

## 14. Relationship to Other Standards

| Standard | Purpose | Blueprint's role |
|----------|---------|-----------------|
| `llms.txt` | Describes content for AI crawlers | Blueprint describes actions — `llms.txt` points to it |
| `robots.txt` | Crawler access control | Blueprint declares agent scope permissions |
| `sitemap.xml` | Page discovery | Blueprint declares capability discovery |
| MCP Tool Definitions | Structured tool calls for LLM agents | Blueprint capability blocks map directly to MCP tool schemas |
| OpenAPI | REST API documentation | Blueprint covers UI and non-REST interactions; links to OpenAPI for APIs |

---

## 15. Versioning

Blueprint follows [Semantic Versioning](https://semver.org/).

- **MAJOR** — breaking changes to required fields or action verb set
- **MINOR** — new optional fields or new verbs added
- **PATCH** — clarifications and corrections

Agents encountering a major version higher than their supported version MUST warn the user before executing any flow.

---

*Blueprint Protocol is an open standard. Use it, fork it, contribute examples via pull request.*
