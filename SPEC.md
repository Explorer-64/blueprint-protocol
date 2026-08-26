# Blueprint Protocol — Specification v3.3.0

**Status:** Draft  
**Published:** 2026-07-10  
**Author:** Abe Reimer

---

## 1. Overview

A `blueprint.txt` file placed at the root of a web application (e.g., `https://yourapp.com/blueprint.txt`) tells AI agents what the app can do and how to invoke it.

Blueprint is structured for agents first. The information an agent needs to make a decision is at the top. The information humans and crawlers need is below it.

An agent evaluating a shortlist of candidate apps reads the header to check for MCP support, reads the CAPABILITIES index to match against the task, fetches only the relevant capability file, and executes. It never needs to read the rest.

Blueprint answers three questions in order of agent priority:

1. **Does this app have MCP?** → Header flag `[MCP]`
2. **Does it do what I need?** → `CAPABILITIES` index
3. **How do I use it?** → Individual capability file

---

## 2. File Location and Discovery

`blueprint.txt` is a machine-readable operational contract for agents — not
page content, not marketing copy. It belongs in `/.well-known/` per RFC 8615,
which designates that path for site-wide machine-readable contracts.

**Primary location (SHOULD):**

```
https://yourapp.com/.well-known/blueprint.txt
```

**Root fallback (MUST also be checked by agents):**

```
https://yourapp.com/blueprint.txt
```

Agents MUST check the `.well-known/` path first, then fall back to root.
Publishers SHOULD serve from `.well-known/` and MAY also serve from root for
backwards compatibility.

A blueprint MUST have exactly one canonical file. Any additional discovery
location — the root fallback, or any other path — MUST be an alias to that
canonical file (a server rewrite, redirect, or symlink), never a separate
physical copy. A copy silently diverges the moment either file is updated
without the other, and there's no error signal when it happens — both URLs
keep returning 200, one of them is just wrong.

**Why `.well-known/` and not root:**
`llms.txt`, `robots.txt`, and `sitemap.xml` are root-level files from an earlier
era of web conventions — they describe content for crawlers. Blueprint Protocol
is different: it is an operational contract that tells agents how to invoke an
app. That distinction maps directly to the RFC 8615 intent. `.well-known/` says
"this is infrastructure for machines operating on this domain."

**For indie developers and quick deployments:**
The root fallback means you can start with `blueprint.txt` at root today and
move it to `.well-known/` when ready. Nothing breaks either way.

Reference it from your `llms.txt` so AI crawlers already reading that file can
follow the pointer:

```
Blueprint: https://yourapp.com/.well-known/blueprint.txt
```

### 2.1 Discovery Surfaces

Recommended discovery surfaces (SHOULD, not MUST):

| Surface | Format |
|---------|--------|
| `llms.txt` | `Blueprint: https://yourapp.com/.well-known/blueprint.txt` |
| HTML `<head>` | `<link rel="blueprint" href="/.well-known/blueprint.txt" type="text/plain" />` |
| `robots.txt` | First line: `# Blueprint: https://yourapp.com/.well-known/blueprint.txt` |

The three-surface pattern maximises likelihood that crawlers and agents find the
file. The `.well-known/` path with root fallback is the only required deployment.

---

## 3. Document Structure

```
# BLUEPRINT header (includes [MCP] flag if server exists)
## CAPABILITIES block — agent reads this first
   Format A: inline declarations (small apps)
   Format B: index with linked capability files (larger apps)
## IDENTITY block
## SUMMARY block (optional — for humans and crawlers)
## AUTH block (or reference)
## MCP block (full server declaration, with optional TRANSPORT and REQUIRED-SECRETS sub-blocks)
## ACCESS block
## TIMING block (optional — real-world wait maximums)
```

Blocks are ordered by agent priority. An agent that finds a capability match
in the CAPABILITIES block can fetch the relevant file and stop — it never needs
to read IDENTITY, SUMMARY, AUTH, or MCP from the root blueprint. Those blocks
exist for human developers, crawlers, and agents that need to authenticate or
install the MCP server.

---

## 4. Header

```
# BLUEPRINT: <App Name>
# Version: <semver>
# URL: <canonical app URL>
# Updated: <YYYY-MM-DD>
```

All four fields are required.

If the app has an MCP server, append `[MCP]` to the app name on the first line:

```
# BLUEPRINT: Imagcon [MCP]
# Version: 3.0.0
# URL: https://imagcon.app
# Updated: 2026-05-15
```

The `[MCP]` flag lets an agent confirm MCP availability from line one — before
reading any other block. Apps without an MCP server omit the flag.

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

## 8. MCP Block

If an MCP server exists for this app, declare it here so agents and tools know
how to connect before attempting to call any tool. This block is optional —
omit it if no MCP server exists.

```
## MCP
server: <server-name>
preferred-transport: <stdio | streamable_http | sse>
install: <full install command the user or agent runs once>
auth: <env var name and what it contains>
```

`preferred-transport` declares which transport the server is optimised for.
Agents and tools SHOULD attempt this transport first.

### 8.1 TRANSPORT Sub-blocks

For each supported transport, declare a `### TRANSPORT (type)` sub-block with
the machine-readable launch or connection parameters. This enables tools to
auto-launch or auto-connect without requiring the user to look up documentation.

**stdio transport:**

```
### TRANSPORT (stdio)
command: <runtime — e.g. uv, uvx, npx, node, python3>
args: ["<arg1>", "<arg2>", ...]
```

**streamable_http transport:**

```
### TRANSPORT (streamable_http)
url: <full URL of the MCP endpoint>
auth: bearer ${<SECRET_NAME>}
```

**sse transport:**

```
### TRANSPORT (sse)
url: <full URL of the SSE endpoint>
auth: bearer ${<SECRET_NAME>}
```

Only declare transports that actually exist. `${VAR}` denotes a secret that
must be resolved at runtime — declare all required secrets in
`### REQUIRED-SECRETS`.

### 8.2 REQUIRED-SECRETS Sub-block

Declare every secret the server needs to operate. Tools use this to prompt
users for credentials before attempting to launch or connect.

```
### REQUIRED-SECRETS
- <SECRET_NAME>:
    description: <what this secret is>
    obtain-at: <URL where the user can get it>
    format: <prefix or pattern — e.g. ic_live_*>
```

### 8.3 CLIENT-CONFIGS Sub-block (optional)

When an MCP server needs different setup instructions per client (Claude Code,
Cursor, VS Code, Windsurf, Zed, and others), don't inline every client's
config into the `## MCP` block — that's the fastest way to bloat a
multi-client server's blueprint. Index them the same way `## CAPABILITIES`
indexes tools: one line per client, pointing to a dedicated file.

```
### CLIENT-CONFIGS
<client-name>: <URL to a file with that client's full setup>
```

Each linked file contains the setup instructions for exactly one client —
nothing else. An agent automating setup, or a human following instructions
for their own IDE, fetches only the file that applies to them.

Linked client files SHOULD open with a `## CLIENT: <name>` header and end
with a `secret-storage:` line describing where that client keeps the
credential (a config file, an OS keychain, a CLI argument). This gives every
file the same shape regardless of which client it documents.

A client file MUST document every transport the server supports **that this
client supports** — not only the server's `preferred-transport`. A client file
that omits a working transport sends its reader to another client's file to
find the config that actually applies to them, which defeats the point of
splitting by client. Where a client supports more than one, lead with the one
that has the fewest prerequisites: a hosted HTTP endpoint needs a URL and a
key, while stdio needs a runtime installed at a compatible version.

Where a client reads config from more than one location, name the specific one
to use and say why. Project-level config files are typically committed to
version control; a file that tells a user to paste a live credential into one
is telling them to commit a secret.

Example (Imagcon, 10 client files, one line each in the index):

```
### CLIENT-CONFIGS
claude-code-cli: https://imagcon.app/blueprints/mcp-clients/claude-code-cli.txt
cursor: https://imagcon.app/blueprints/mcp-clients/cursor.txt
vscode-copilot: https://imagcon.app/blueprints/mcp-clients/vscode-copilot.txt
```

This is the same index-of-links idiom `## CAPABILITIES` establishes, applied
to a second case: content that varies per-reader and shouldn't be loaded by
every reader.

### 8.4 Full example

The examples in this section (and the CLIENT-CONFIGS example in §8.3) are
illustrative snapshots — they will drift from the live file over time, the
same way any frozen copy does.

Live reference: https://imagcon.app/blueprint.txt

```
## MCP
server: imagcon-mcp
preferred-transport: stdio
install: claude mcp add imagcon -- uvx imagcon-mcp --api-key <<api-key>>
auth: IMAGCON_API_KEY — user's Imagcon API key from account settings

### TRANSPORT (stdio)
command: uv
args: ["run", "imagcon-mcp", "--api-key", "${IMAGCON_API_KEY}"]

### TRANSPORT (streamable_http)
url: https://mcp.imagcon.app
auth: bearer ${IMAGCON_API_KEY}

### REQUIRED-SECRETS
- IMAGCON_API_KEY:
    description: Your Imagcon API key for MCP and API access
    obtain-at: https://imagcon.app/api-keys
    format: ic_live_*
```

A tool reading this block can:
- Pick a transport (stdio or streamable_http)
- Know exactly how to launch or connect
- Identify which secrets are needed and where to get them
- Prompt the user for only the secrets that are missing

Individual capability blocks reference the tool name only:

```
### MCP
tool: <tool-name>
```

### 8.5 Schema Precedence

Where a capability names a live machine-readable interface — an MCP tool, an
OpenAPI operation — **that interface is normative and the blueprint is not.**
The blueprint's job is discovery and selection: helping an agent find the app,
decide the capability matches the task, and understand what it will get back.
Once the agent is connected and the schema is in front of it, the schema wins
on every parameter name, type, and required flag.

Agents MUST resolve any disagreement in favour of the live schema, regardless
of read order. Without a stated rule, an agent trusts whichever it read last,
and which one that is depends on the runtime rather than on either document.

The corollary for publishers is the more important half:

- Declare the tool name and the parameters an agent needs in order to *decide*.
- Do NOT restate the full schema. A duplicated schema is a schema that drifts,
  and nothing fails loudly when it does — both documents keep serving 200.
- Prefer a pointer over a copy for anything the live interface already
  publishes in machine-readable form.

This rule is what makes it safe for a capability file to stay short. A
capability block that names a tool and stops is not underspecified — it is
correctly deferring.

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

## 10. CAPABILITIES Block

Two formats are valid. Choose based on app size. Both produce the same outcome
for agents — the difference is whether capabilities live inline or in separate
fetchable files.

---

### Format A — Inline (small apps)

All capabilities declared directly in `blueprint.txt`. Suitable for apps with
fewer than ~10 capabilities. This is the 10-minute version.

Each capability is declared independently. The capability describes **what** the
app can do; the invocation blocks describe **how** to do it.

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
    retrieval: <inline | url-public | url-signed | url-authenticated>
    description: <what the agent gets back>
next-step: <what the caller must do with the output to finish the task>
auth-required: <true | false>
scope: <read-only | form-submit | file-download | account-modify | financial-transaction | destructive>
permissions:
  - <resource>: <read | write | delete>

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

Only include the invocation blocks that actually exist. A capability with only
an API has no MCP or UI block.

#### Parameter Naming

`name:` under `input:` is the **literal wire name** of the parameter, exactly as
the target interface accepts it — the MCP tool's schema key, the JSON body
field, the query string key. Copy it verbatim. Never normalize it, never
convert it to kebab-case.

Kebab-case governs exactly two things: `<capability-id>` values and
`<<variable>>` names. It does not govern parameter names. Applying it to them
produces a blueprint that reads correctly and fails on every call:

```
input:
  - name: image-key       # WRONG — the tool's schema key is image_key
  - name: image_key       # correct
```

This failure is invisible at authoring time. The document looks well-formed, no
parser can catch it, and the agent that trusted the blueprint gets an
unknown-parameter error from the interface instead. Where a capability targets a
live schema (§8.5), implementers SHOULD validate `input.name` values against
that schema in CI — it is a mechanical check, and it is the only thing that
catches this class of drift.

#### Output Retrieval

`retrieval` declares how the caller actually gets the output, which is not the
same question as what the output is. It is optional; omit it when the output is
returned directly.

| Value | Meaning |
|-------|---------|
| `inline` | The content is in the response body — no further request |
| `url-public` | A URL requiring no credential |
| `url-signed` | A pre-authorised URL carrying its own expiry — fetch with a plain GET, no header |
| `url-authenticated` | A URL requiring the caller to attach a credential on a second request |

`url-authenticated` is a declaration that the capability cannot be completed in
one hop. That is worth stating plainly because it is a common place for agent
flows to stall: many runtimes are sandboxed against putting a live credential
into an outbound request they construct themselves, so an agent can hold a valid
key, complete the call, receive a URL, and still be unable to retrieve the
result. Declaring it lets an agent detect that before it starts rather than
two-thirds of the way through.

Publishers SHOULD prefer `url-signed` over `url-authenticated` for generated
artifacts. A short-lived signed URL is retrievable by any client that can make a
GET, and it does not put a long-lived credential into a second request.

Where `retrieval` is `url-signed`, the output SHOULD also carry the field naming
its expiry so an agent can tell a stale URL from a broken one.

#### next-step

`next-step` declares what the caller must do with the output for the task to
actually be finished. It is optional, and should be omitted where receiving the
output *is* the completion.

It exists because `output` answers "what do I get back" and stops there, which
leaves a gap for any capability whose result is an input to work the agent still
has to do. A capability can return exactly the right archive and still leave the
task failed — files extracted to the wrong root, names changed on the way in, a
required snippet never inserted. Nothing in the capability declaration would
have caught it, because from the app's side the call succeeded.

```
next-step: Extract the archive into the project's static asset root. Do not
  rename files — the manifest references them by path. Insert the returned
  html_head value into the document <head>.
```

Two rules make it useful rather than decorative:

- **State it as an instruction to the caller, not a description of the output.**
  "Returns a ZIP of icons" is an `output` description. "Extract into the static
  asset root without renaming" is a next-step.
- **Where the app can supply a piece of what the caller would otherwise have to
  derive, supply it and say so.** If the app knows the exact markup, the exact
  config values, or the exact paths, emitting them turns an inference the agent
  might get wrong into a value it can paste. An agent reconstructing something
  the app already knows is a failure of the contract, not of the agent.

---

### Format B — Index (larger apps)

The root `blueprint.txt` declares a capability index. Each entry points to a
standalone capability file and declares the intended actor.

```
## CAPABILITIES
<capability-id>: <url> | <actor> | <description>
<capability-id>: <url> | <actor> | <description>
```

The third field is an optional one-line description. Publishers SHOULD include
it. Parsers MUST treat a two-field line as valid and the description as absent.

Example:

```
## CAPABILITIES
generate-icon-set: https://imagcon.app/blueprints/generate-icon-set.txt | mcp | Generate a full PWA icon set from a text description
generate-splash-screens: https://imagcon.app/blueprints/generate-splash-screens.txt | mcp | Generate iOS and Android splash screens from a text description
edit-image: https://imagcon.app/blueprints/edit-image.txt | human-only | Crop, draw on, and refine an image in the browser editor
check-credits: https://imagcon.app/blueprints/check-credits.txt | mcp | Read the account's remaining generation credits
purchase-credits: https://imagcon.app/blueprints/purchase-credits.txt | human-only | Buy additional credits via checkout
browse-inspiration: https://imagcon.app/blueprints/browse-inspiration.txt | ui | Browse the public gallery of generated images
```

#### Why the description field matters

Without it, the index gives an agent a capability id and an actor tag — enough
to rule out the wrong *actor*, but not enough to pick the right *capability*.
Faced with several plausible ids, the only way to find out which one matches the
task is to fetch them and read them, so fetching most of the index becomes the
rational strategy. The index stops being a routing table and becomes a directory
listing.

The cost shows up as an apparent verbosity problem — an agent reports that the
blueprint was too long, when what actually happened is that it was made to load
eight files to use one. One line per entry is what lets step 3 of the agent
behaviour below fetch exactly one file.

Keep descriptions short and task-phrased — what an agent would be trying to do,
in the words it would use. This is the text a discovery agent matches against.

#### Actor Values

| Actor | Meaning |
|-------|---------|
| `mcp` | Agent invokes via MCP tool — fetch the capability file for tool name and parameters |
| `api` | Agent invokes directly via HTTP — fetch the capability file for method, endpoint, and parameters |
| `ui` | Agent can automate via UI steps — fetch the capability file for step-by-step flow |
| `human-only` | Agent MUST NOT attempt this capability — intended for human users only |
| `x402` | Agent invokes via HTTP but must satisfy an x402 payment challenge — requires a funded wallet and on-chain signing capability, prerequisites a plain `api` caller does not have |

**Adding new actor values:** A new actor value is justified only when it names
a distinct class of caller with different prerequisites — not a transport or
implementation variant of an existing class. A payment-gated endpoint that
requires a funded wallet and signing capability (`x402`) is a different caller
class than a plain HTTP client (`api`). A tool that only runs via local stdio
versus one also available over hosted HTTP is still the same caller class
(`mcp`) — that distinction belongs in a prose note under the `## MCP` block,
not in the actor tag.

#### Rules for Format B

- Root blueprint MUST NOT contain inline `## CAPABILITY:` blocks when using
  Format B
- Each `url` MUST point to a publicly accessible plain text capability file
- Each capability file MUST contain exactly one `## CAPABILITY:` block using
  Format A syntax
- Capability files do NOT repeat IDENTITY, AUTH, or MCP blocks — those are
  declared once in the root blueprint and apply to all capabilities
- Agents MUST NOT fetch or attempt `human-only` capability files during task
  execution
- `human-only` capability files MAY exist for human developer reference but
  agents should treat the actor declaration as a hard stop
- `<capability-id>` MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` (lowercase kebab-case)
- IDs MUST be unique across the index
- The description field, when present, MUST be a single line. Parsers MUST split
  each entry on the first two `|` characters only and treat the entire remainder
  of the line as the description

#### Agent behaviour with Format B

1. Read root blueprint — identify SUMMARY, AUTH, MCP, and CAPABILITIES index
2. Scan index for capabilities matching the task — filter out `human-only` entries
3. Fetch only the capability file(s) relevant to the task
4. Execute using the fetched capability's invocation blocks

An agent handling a PWA icon generation task fetches `generate-icon-set.txt`
only. It never loads `edit-image.txt` or `purchase-credits.txt`.

---

`<capability-id>` MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` (lowercase kebab-case).
IDs MUST be unique within the document.

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

### Disambiguating Repeated Elements

Many UIs render the same interactive role multiple times — a resize button on
every card in a gallery, a delete button on every row in a table.
`data-agent-id` alone can't distinguish between them if the same role name
repeats across the list.

The RECOMMENDED pattern is a second attribute, `data-agent-key`, carrying the
instance identifier. `data-agent-id` stays a stable, static role name;
`data-agent-key` carries the dynamic value. Steps combine both in a compound
selector:

```html
<button data-agent-id="resize-button" data-agent-key="img_abc123">Resize</button>
```

```
CLICK [data-agent-id="resize-button"][data-agent-key="<<image-key>>"]
```

This keeps `data-agent-id` free of dynamic content entirely, which avoids the
selector normalization problem described in §12 — an attribute *value* has no
CSS-safety constraint, so nothing needs to be transliterated or stripped to
make it safe.

Agents obtain the key value either from a prior tool/API response (e.g. an
`image_key` returned by an earlier capability call) or by reading the
`data-agent-key` attribute directly off the relevant element in the DOM — for
example, reading it from a `data-agent-id="gallery-image-card"` element before
clicking that card's action button. If no key is available and the task
doesn't specify one, agents SHOULD default to the first matching element and
note the assumption.

Any step action MAY use a compound selector combining `data-agent-id` and
`data-agent-key` wherever a single `[data-agent-id="x"]` selector is shown
below.

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
| Complete | `COMPLETE <description>` | Hand control to an external flow the agent cannot step through (e.g. Stripe checkout, Google OAuth popup, native OS dialog). The description tells the agent what the user must do. Flow resumes at the next step once the external interaction is finished. |

### VERIFY Conditions

```
VERIFY url == "/path"
VERIFY url contains "/path"
VERIFY selector_exists [data-agent-id="x"]
VERIFY selector_not_exists [data-agent-id="x"]
VERIFY file_type == ".zip"
VERIFY text_contains [data-agent-id="x"] "string"
VERIFY value starts_with "prefix"
VERIFY attribute_changed [data-agent-id="x"] "src"
VERIFY http_status == 200
```

`url contains` is preferred over `url ==` for SPAs where query strings or hash fragments may be appended to the path.

`value starts_with` checks that the text content or value of the most recently interacted element begins with the given string — useful for verifying generated keys or tokens.

`attribute_changed` checks that a named attribute on an element has a different value than it did when the step began — useful for confirming an image or resource has been replaced.

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

### Variable Normalization in Selectors (legacy pattern)

For repeated elements, prefer `data-agent-key` (§11) over interpolating a
variable directly into `data-agent-id` — it avoids the entire normalization
problem this section exists to solve. This pattern remains valid for existing
implementations and for cases where the id itself is inherently unique and
meaningful on its own, but new blueprints SHOULD use the compound-selector
pattern instead. It may be deprecated in a future major version.

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

## 13. TIMING Block

The `## TIMING` block is optional. It declares real-world observed wait maximums
for operations that have variable execution times — AI generation, file processing,
external API calls. Agents use these values to set appropriate `WAIT` maximums
rather than guessing or using fixed values.

```
## TIMING
# Format: <operation-label>: <observed-range> — use max: <N>s
ai-image-generation: 15–45s — use max: 60s
ai-image-refinement: 20–90s — use max: 120s
file-processing: 5–15s — use max: 30s
file-upload: 2–5s — use max: 15s
```

Values MUST reflect real-world observations, not aspirational targets. Include
the observed range as a comment so implementers understand the variability.
Agents MUST use the declared maximum in `WAIT` steps for the corresponding
operation type.

Place `## TIMING` after `## ACCESS` and before `## CAPABILITIES`.

---

## 14. Scope Values

Scope declares the highest-risk operation a capability performs. Agents MUST NOT exceed declared scope. Agents MUST prompt the user for confirmation before executing `financial-transaction` or `destructive` scope.

| Scope | Permits |
|-------|---------|
| `read-only` | Navigation and observation only |
| `form-submit` | Fill and submit forms |
| `file-download` | Trigger a file download |
| `edit` | Modify existing content — crop, draw, annotate, refine. Does not submit a form or download a file. |
| `account-modify` | Change account settings or profile |
| `financial-transaction` | Payment, billing, subscription, or any in-app purchase |
| `destructive` | Delete or permanently modify data |

Use `financial-transaction` for any capability that initiates a payment or purchase — including in-app credit purchases. Do not invent scope values outside this list.

---

## 15. Permission Values

The optional `permissions` field declares what data resources a capability accesses and at what level. `scope` declares how risky the operation is — `permissions` declares what data it touches. Both together give an agent everything it needs to make an informed decision before connecting.

```
permissions:
  - <resource>: <read | write | delete>
```

| Resource | Meaning |
|----------|---------|
| `profile` | User profile data — name, email, avatar |
| `files` | User files or documents |
| `billing` | Payment methods, invoices, subscription data |
| `account` | Account settings and preferences |
| `contacts` | User contacts or address book |
| `calendar` | Calendar events |
| `data` | App-specific user data (general) |

Access levels: `read`, `write`, `delete`

Example:

```
## CAPABILITY: export-expenses
auth-required: true
scope: file-download
permissions:
  - data: read
  - files: write
```

Agents MUST surface declared permissions to the user before executing any capability with `write`, `delete`, or `billing` permissions.

`permissions` is optional. Omit it when the capability accesses no user data beyond what auth already covers.

Do not invent resource tokens outside this list.

---

## 16. Relationship to Other Standards

| Standard | Purpose | Blueprint's role |
|----------|---------|-----------------|
| `llms.txt` | Describes content for AI crawlers | Blueprint describes actions — `llms.txt` points to it |
| `robots.txt` | Crawler access control | Blueprint declares agent scope permissions |
| `sitemap.xml` | Page discovery | Blueprint declares capability discovery |
| MCP Tool Definitions | Structured tool calls for LLM agents | Blueprint names the tool and defers to its live schema (§8.5) |
| OpenAPI | REST API documentation | Blueprint covers UI and non-REST interactions; links to OpenAPI for APIs |

---

## 17. Parsing and Error Handling

- Parsers SHOULD recover block-by-block; a malformed capability MUST NOT
  invalidate unrelated capabilities in the same document.
- Unknown `VERIFY` predicates SHOULD fail closed: abort the flow and surface
  the unrecognised condition to the user rather than silently continuing.
- On major version incompatibility (agent's supported major < document major),
  agents MUST warn the user before executing any steps.

---

## 18. Versioning

Blueprint follows [Semantic Versioning](https://semver.org/).

- **MAJOR** — breaking changes to required fields or action verb set
- **MINOR** — new optional fields or new verbs added
- **PATCH** — clarifications and corrections

Agents encountering a major version higher than their supported version MUST warn the user before executing any flow.

---

*Blueprint Protocol is an open standard. Use it, fork it, contribute examples via pull request.*
