# Blueprint Protocol — Specification v2.4.0

**Status:** Draft  
**Published:** 2026-04-13  
**Author:** Abe Reimer

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

### 2.1 Discovery Surfaces

Recommended discovery surfaces (SHOULD, not MUST):

| Surface | Format |
|---------|--------|
| `llms.txt` | `Blueprint: https://yourapp.com/blueprint.txt` |
| HTML `<head>` | `<link rel="blueprint" href="/blueprint.txt" type="text/plain" />` |
| `robots.txt` | First line: `# Blueprint: https://yourapp.com/blueprint.txt` |

The three-surface pattern maximises likelihood that crawlers and agents find the
file. Only the root URL is strictly required for a valid deployment.

---

## 3. Document Structure

```
# BLUEPRINT header
## IDENTITY block
## SUMMARY block (optional — for discovery and recommendation)
## INDEX block (optional — for large apps with multiple sub-blueprints)
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

## 7. INDEX Block

The `INDEX` block is optional. It is designed for large apps where a single
`blueprint.txt` would become unwieldy. Instead of declaring all capabilities
inline, the root blueprint declares an index of sub-blueprints — one per
section, feature area, or bounded context of the app.

This follows the same pattern as an XML sitemap index: one root file that points
to sub-files, each covering a scoped portion of the whole.

```
## INDEX
blueprints:
  - url: https://yourapp.com/blueprints/auth.txt
    scope: <one line — what this sub-blueprint covers>
  - url: https://yourapp.com/blueprints/billing.txt
    scope: <one line — what this sub-blueprint covers>
  - url: https://yourapp.com/blueprints/dashboard.txt
    scope: <one line — what this sub-blueprint covers>
```

**Rules:**
- Each `url` MUST point to a valid blueprint file conforming to this spec.
- Each sub-blueprint MUST include its own `## IDENTITY` and `## AUTH` block
  (or an `## AUTH ref:` pointing back to the root).
- Each sub-blueprint MUST include its own `## ACCESS` block declaring the
  methods available for its capabilities.
- The root blueprint MUST NOT declare `## CAPABILITIES` blocks if `## INDEX`
  is present — capabilities live in the sub-blueprints only.
- The root blueprint SHOULD NOT declare `## ACCESS` when `## INDEX` is present
  — access is declared per sub-blueprint.
- Agents SHOULD read the root `## SUMMARY` to understand intent, then fetch
  only the sub-blueprint whose `scope` matches the requested task.

Example — a large project management app:

```
## INDEX
blueprints:
  - url: https://yourapp.com/blueprints/projects.txt
    scope: creating, editing, and archiving projects
  - url: https://yourapp.com/blueprints/tasks.txt
    scope: adding, assigning, and completing tasks
  - url: https://yourapp.com/blueprints/billing.txt
    scope: subscription management and invoices
  - url: https://yourapp.com/blueprints/account.txt
    scope: user profile, settings, and notifications
```

---

## 8. AUTH Block

Declare authentication once. Apps sharing an auth provider (e.g., a suite on shared Firebase Auth) should declare auth in a central blueprint and reference it.

### Inline auth declaration:

```
## AUTH
provider: <firebase | auth0 | custom | none>
method: <email | oauth | api-key | session>
```

Or **plural modes** when the UI supports multiple sign-in surfaces (comma-separated):

```
## AUTH
provider: firebase
methods: email-password, oauth-google
```

Agents MUST interpret `methods` as an unordered set of allowed sign-in surfaces.
Absent `methods`, derive behaviour from single `method` only.

Valid `method`/`methods` tokens: `none`, `email`, `email-password`, `oauth`, `oauth-google`, `oauth-github`, `oauth-microsoft`, `api-key`, `session`, `magic-link`.

### Reference to a shared auth spec:

```
## AUTH
ref: https://yourapp.com/blueprint.txt#auth
```

Using a reference means a change to shared auth requires updating one file, not every app blueprint in the suite.

Fragment resolution: `#auth` designates the textual span from the line matching
`^\s*##\s+AUTH\s*$` inclusive, until the next line matching `^\s*##\s+[A-Za-z]`
(exclusive) or EOF. Other fragments MUST be ignored unless a future minor version
assigns meaning.

---

## 9. MCP Block

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

## 10. ACCESS Block

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

### Tier evaluation rules

Agents MUST evaluate the ACCESS hierarchy in order and stop at the first available method:

1. If `preferred: mcp` is declared and an MCP server is reachable, use MCP. Do not attempt API or UI.
2. If `fallback: api` is declared and no MCP is available or reachable, use the API endpoint.
3. If `last-resort: ui` is declared and no programmatic interface is available, use UI automation.
4. If a declared method is missing from a specific capability block, skip to the next tier for that capability only.

**UI-only apps** (no API, no MCP):
```
## ACCESS
last-resort: ui
```

**API-only apps** (no MCP, no UI automation):
```
## ACCESS
preferred: api
```

Do not declare `preferred: mcp` or `fallback: api` as a placeholder. If the method does not exist, omit it.

---

## 11. CAPABILITIES Block

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

`<capability-id>` MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` (lowercase kebab-case).
IDs MUST be unique within the document.

---

## 12. UI Step Actions

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
VERIFY url contains "/path"
VERIFY selector_exists [data-agent-id="x"]
VERIFY selector_not_exists [data-agent-id="x"]
VERIFY file_type == ".zip"
VERIFY text_contains [data-agent-id="x"] "string"
VERIFY http_status == 200
```

`url contains` is preferred over `url ==` for SPAs where query strings or hash fragments may be appended to the path.

---

## 13. Variables

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
| `読書` | `(strip — empty result, see below)` |

App developers rendering dynamic `data-agent-id` values in HTML MUST apply the same normalization to ensure selectors match at runtime. Both sides of the contract must normalize identically or the selector will silently fail.

If normalisation produces an empty segment, agents MUST NOT attempt to construct
or match the selector. App authors MUST instead emit a stable opaque id
(e.g. UUID or numeric row id) in the HTML attribute and reference that literal
value in the blueprint step. Silent failure from an empty dynamic selector is
not acceptable.

---

## 14. Scope Values

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

## 15. Relationship to Other Standards

| Standard | Purpose | Blueprint's role |
|----------|---------|-----------------|
| `llms.txt` | Describes content for AI crawlers | Blueprint describes actions — `llms.txt` points to it |
| `robots.txt` | Crawler access control | Blueprint declares agent scope permissions |
| `sitemap.xml` | Page discovery | Blueprint declares capability discovery |
| MCP Tool Definitions | Structured tool calls for LLM agents | Blueprint capability blocks map directly to MCP tool schemas |
| OpenAPI | REST API documentation | Blueprint covers UI and non-REST interactions; links to OpenAPI for APIs |

---

## 16. Parsing and Error Handling

- Parsers SHOULD recover block-by-block; a malformed capability MUST NOT
  invalidate unrelated capabilities in the same document.
- Unknown `VERIFY` predicates SHOULD fail closed: abort the flow and surface
  the unrecognised condition to the user rather than silently continuing.
- On major version incompatibility (agent's supported major < document major),
  agents MUST warn the user before executing any steps.

---

## 17. Versioning

Blueprint follows [Semantic Versioning](https://semver.org/).

- **MAJOR** — breaking changes to required fields or action verb set
- **MINOR** — new optional fields or new verbs added
- **PATCH** — clarifications and corrections

Agents encountering a major version higher than their supported version MUST warn the user before executing any flow.

---

*Blueprint Protocol is an open standard. Use it, fork it, contribute examples via pull request.*
