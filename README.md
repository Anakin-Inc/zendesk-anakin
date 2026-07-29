# zendesk-anakin

A Zendesk Support app (built on the Zendesk Apps Framework, ZAF v2) that adds
[Anakin](https://anakin.io) — web scraping, mapping/crawling, AI-powered and
agentic search, Wire site actions, AI-visibility comparisons, and
monitor/session lookups — to the ticket sidebar.

## What's covered

A ticket-sidebar iframe app with ten tabs, one per Anakin REST endpoint (or
tightly-coupled pair of endpoints), covering 14 of Anakin's 21 API
capabilities:

- **Scrape URL** — submits `POST /v1/url-scraper`, then polls
  `GET /v1/url-scraper/:jobId` until the job completes or fails. Shows the
  returned markdown.
- **AI Search** — calls `POST /v1/search`. Synchronous, no polling. Costs 3
  credits per call.
- **Agentic Search** — submits `POST /v1/agentic-search`, then polls
  `GET /v1/agentic-search/:jobId`. Shows the AI-generated summary plus, if a
  JSON Schema was supplied, the structured data. Costs 10 credits.
- **Map** — submits `POST /v1/map`, then polls `GET /v1/map/:jobId`. Lists
  the internal/external URLs discovered under a site — useful for finding
  the right sub-page before scraping it.
- **Crawl** — submits `POST /v1/crawl`, then polls `GET /v1/crawl/:jobId`.
  Bulk-fetches markdown across several pages at once (e.g. a whole docs
  section) in one call.
- **Wire Find** — two lookups in one tab: `GET /v1/wire/resolve?q=&limit=`
  (find candidate actions from a natural-language intent) and
  `GET /v1/wire/catalog[/:slug]` (browse every supported site, or one site's
  full action list). Both feed an `action_id` into Wire Run.
- **Wire Run** — submits `POST /v1/wire/task` with an `action_id` (and
  optional `params`/`credential_id`); polls `GET /v1/wire/jobs/:jobId` when
  the response includes a `job_id`, otherwise the sync result is shown
  directly. Only exercises Wire's **read** actions — see "Scope" below.
- **AI Visibility** — submits `POST /v1/ai-visibility/search`, then polls
  `GET /v1/ai-visibility/search/:search_id` until the status leaves
  `"running"`. Shows the cross-engine synthesis plus each engine's raw
  result. A secondary button calls `GET /v1/ai-visibility/sources` to list
  queryable engine slugs.
- **Monitors** — `GET /v1/monitors` (list) plus, given a monitor ID,
  `GET /v1/monitors/:id/changes`. Read-only status/history lookups against
  monitors created elsewhere (e.g. the dashboard or another integration).
  A second section on the same tab exposes **monitor control**
  (`POST /v1/monitors/:id/pause`, `/resume`, `/run`) so an agent who spots a
  stale or noisy monitor while working a ticket can act on it immediately —
  restricted to those three reversible actions (never `DELETE
  /v1/monitors/:id`) and gated behind a checkbox that must be re-checked
  before every single use; see "Scope" below for why this one piece of
  `monitor_control` earned guardrails instead of exclusion.
- **Sessions** — `GET /v1/sessions?domain=`. Lists saved login sessions
  usable by Scrape/Crawl/Monitors/Wire; does not create or delete any.

Once a result is showing, **Insert as internal note** / **Insert as public
reply** append it into the ticket's comment editor via the documented
`comment.type` / `comment.appendText` ZAF paths — the agent still has to hit
Zendesk's own Submit button, this only fills the box.

## Scope: why 14 of 21 capabilities, not all of them

Anakin's API has 21 tool-level capabilities (per `anakin-mcp/src/tools/*.ts`).
The first pass on this app covered the 13 that are **lookups** — safe to
hand an agent working a live ticket with no extra review step — and excluded
8 that are **state-changing** or **credential-managing** under the blanket
reasoning "a support agent's sidebar is for lookups, not automation."

This revision re-examined all 8 individually against one question: *does a
concrete, in-app guardrail (restricting which actions are exposed, an
explicit re-checked confirmation step) actually neutralize the risk, or does
it only paper over a click?* One of the 8 — `monitor_control` — passed that
test in a restricted form and was added. The other 7 did not, each for a
distinct reason, not a repeat of the blanket one:

- **`monitor_control` (added, restricted)** — the tool bundles four actions
  behind one `action` parameter: `pause`, `resume`, `run_now`, `delete`. The
  first three only change *scheduling* on a monitor the agent is already
  looking at (having just called `monitor_list`/`monitor_changes` on this
  same tab) and are fully reversible — pausing/resuming toggles a flag,
  `run_now` just triggers one extra check ahead of schedule. `delete` is
  irreversible and was **left out of this UI's `action` dropdown entirely**
  — the app only ever calls `POST .../pause`, `/resume`, or `/run`, never
  `DELETE /v1/monitors/:id`, regardless of what the underlying capability
  supports. The three that remain are gated by a checkbox ("I understand
  this changes the monitor's schedule immediately...") that must be
  re-checked before *every* use — the Apply button is disabled by default,
  stays disabled until checked, and the checkbox is force-cleared after each
  action and on every tab switch, so a stray extra click can't silently
  re-fire it. This is a narrower guarantee than the MCP tool's own
  `destructiveHint` annotation (which covers all four actions); the
  narrowing is intentional and the tradeoff — real, useful, in-context
  monitor cleanup vs. leaving `delete` in the dashboard's hands — is a
  correct one for a ticket sidebar.
- **`wire_write_action`** — reconsidered and still excluded. Unlike
  `monitor_control`, this touches a **third-party site**, not just Anakin's
  own account, and most write actions require a `credential_id` — meaning a
  confirmation dialog would only gate the *click*, not the underlying
  concern of an agent using a shared, org-level Wire credential to take an
  irreversible action (cancel an order, submit a form, post content) on a
  live external site with no undo. `anakin-mcp/src/tools/wire.ts` itself
  documents a further wrinkle: `GET /wire/resolve`'s results don't reliably
  report an action's `type` (confirmed live by the sibling
  `intercom-anakin` submission in this batch), so safely gating write vs.
  read requires an extra `wire_catalog` lookup to trust the `type` field at
  all — real engineering this pass didn't do, and shipping a "confirm first"
  write surface without it would be exactly the kind of half-verified,
  destructive capability this task said not to fabricate.
- **`wire_identities`** — technically `readOnlyHint: true` in
  `anakin-mcp/src/tools/wire.ts` (it only lists saved credentials, it
  doesn't create them), so it was never the risky half of the excluded
  pair — `wire_login` is. It stays excluded for the original reason:
  with `wire_write_action` out of scope, there's still no in-app consumer
  for a `credential_id` it would return.
- **`wire_login`** — reconsidered and still excluded. This doesn't run a
  bounded action, it establishes a brand-new, persistent, encrypted session
  on a third-party site that outlives the ticket. A confirmation dialog
  addresses "did you mean to click this," not "is a support ticket the
  right moment to be creating durable third-party account sign-in state" —
  that's a deliberate, dashboard-level decision, not an in-the-moment one.
- **`wire_build`** — reconsidered and still excluded. It publishes a new,
  billed, AI-generated scraper action to the **whole account's** Wire
  catalog — a side effect that outlives and outscopes the ticket entirely,
  and the agent has no way to evaluate whether the generated scraper is
  even correct before "confirming" it. There's nothing concrete to confirm.
- **`monitor_create`** — reconsidered and still excluded, unlike its sibling
  `monitor_control`. Creating a monitor starts a **recurring, indefinitely
  billed** job with no natural owner once the ticket closes — and because
  this app's `monitor_control` UI deliberately omits `delete`, there isn't
  even an in-app way to unwind an accidental creation. A confirmation
  checkbox prevents a misclick; it doesn't give the new monitor a manager.
- **`session_delete`** — reconsidered and still excluded. It destroys a
  saved login session that other integrations/monitors may depend on, and
  neither `session_list` nor this app surfaces what else references a given
  session — so there's no way to show an agent an accurate "here's what
  breaks" before they confirm. A confirmation dialog can't compensate for
  information the app doesn't have.
- **`browser_task`** — reconsidered and still excluded. It's driven by a
  free-text natural-language prompt with no `readOnlyHint` at all (the one
  capability with zero safety annotation), can run up to ~5.5 minutes, and
  can do a materially different thing each time depending on how the prompt
  is interpreted. A confirmation dialog would be confirming a task
  *description*, not a concrete action — not a meaningful guardrail for an
  open-ended capability.

For a second opinion on the same question in a near-identical context, the
sibling `intercom-anakin` submission in this batch (also a support-agent
conversation-sidebar app, built independently this session) reached the same
conclusion for `wire_write_action`, `wire_login`/`wire_identities`,
`wire_build`, `monitor_create`, `session_delete`, and `browser_task` —
excluding all of them, and additionally never rendering a Wire write action
as tappable "anywhere" in its UI. `github-app-anakin`, a comment-triggered
bot in a similar single-actor-facing surface, independently reached the same
conclusion for the same reasons, plus flagged that even its read-only
`/anakin-wire-run` command doesn't verify an action's `type` before running
it — the same resolve-endpoint gap noted above. By contrast, `coda-anakin`
and `tray-anakin` — a low-code builder and an iPaaS workflow tool, not an
ambient always-on ticket sidebar — do expose `wire_write_action` and
`monitor_create`/`monitor_control` in full, because in those tools the
"confirmation" is structural: a human explicitly drags in a write-capable
connector at *design* time, days before it ever runs, which is a
fundamentally different guarantee than a checkbox next to a button in a
sidebar that's open because a ticket happens to be.

`wire_read_action` is included even though it *can* touch pages that require
auth: the tool itself is annotated `readOnlyHint: true` in
`anakin-mcp/src/tools/wire.ts` (it only extracts data), and the sidebar takes
an optional `credential_id` field rather than trying to reproduce Wire's
sign-in flow.

## Files

```
manifest.json            App metadata, ticket_sidebar location, the secure
                          apiKey parameter, domainWhitelist
assets/iframe.html        The entire app: form UI + JS for all ten tabs,
                          single file
translations/en.json      Required marketplace-listing copy (name,
                          short/long description, installation_instructions,
                          parameter label/help)
```

No build step — this is a plain HTML/CSS/JS file, no bundler, no
`package.json`. That matches every real Zendesk ZAF sample app checked
(`zendesk/demo_apps`) — ZAF apps are static assets loaded in an iframe, not
a Node project.

## Install (local dev)

```sh
npx @zendesk/zcli apps:server
```

Serves the app locally (`zcli apps:server` per the real ZCLI command
reference) so it can be pointed at from a Zendesk instance in development
mode for live testing inside an actual ticket.

To install for real, a Zendesk account is required — see SUBMIT.md.

## Auth: the API key never reaches the browser

`manifest.json` declares one installation parameter:

```json
{ "name": "apiKey", "type": "text", "required": true, "secure": true }
```

`type: "password"` is **not** a real ZAF parameter type — checked against
`zendesk_apps_support`'s own manifest validation spec (`zendesk/
zendesk_apps_support`, `spec/validations/manifest_spec.rb`), which enumerates
`text`, `number`, `oauth`, `hidden`, `checkbox`, `multiline`, and a few
others but not `password`. The real mechanism for a secret is `type: "text"`
plus `"secure": true`.

`secure: true` does more than hide the field on the settings screen: per
Zendesk's own "Making API requests from a Zendesk app" doc, a secure
setting's *value* is never sent to the app's client-side JavaScript at all.
It's referenced only as a `{{setting.apiKey}}` placeholder inside a
`client.request()` call's `headers`/`data`, and Zendesk's own request proxy
substitutes the real value server-side, outside the browser — "only the
placeholder, not the value, is displayed in the browser's dev tools." So
`assets/iframe.html` never calls `client.metadata()` to read the key (that
call exists, and does return `metadata.settings`, but a secure setting's
real value deliberately isn't in there — that's the point of `secure`, not
an implementation gap here).

## CORS: not a blocker, because it's the wrong question

`client.request()` isn't a browser `fetch()`/`XHR` — Zendesk's own docs
describe it as handling "cross-origin restrictions through Zendesk's proxy
infrastructure," and every real request to an external domain has to be
declared in `manifest.json`'s `domainWhitelist` first (`["api.anakin.io"]`
here). The request is issued by Zendesk's servers, not the sandboxed iframe
origin, so api.anakin.io's CORS headers (or lack of them) for
browser-origin calls are irrelevant — there's no scenario in a ZAF app where
that matters, secure setting or not. This is why `iframe.html` uses
`client.request()` throughout instead of `fetch()`.

## Precedent

Yext's "AI Search" app is already live in the Zendesk Marketplace, adding
an AI-search box to the ticket sidebar backed by a third-party API key the
agent supplies during install — confirming a search-over-third-party-API
sidebar app is an accepted, precedented app category, not a novel one.
Nothing was copied from it (its source isn't public); it only confirmed the
shape.

## Steps (needs the account owner)

1. `npx @zendesk/zcli login -i` (or set `ZENDESK_SUBDOMAIN` /
   `ZENDESK_EMAIL` / `ZENDESK_API_TOKEN`) against a real Zendesk account —
   needed for every remaining step, including local validation.
2. `npx @zendesk/zcli apps:validate .` — real manifest/asset validation
   against Zendesk's servers.
3. `npx @zendesk/zcli apps:create .` — uploads and installs the app
   privately on that account (Support Professional plan or above required).
4. Open a real ticket, add the app to the sidebar, paste a real Anakin API
   key (from [anakin.io/dashboard](https://anakin.io/dashboard) — 500
   credits, no card required) into the install settings, and exercise all
   ten tabs against live data — including the Monitors tab's pause/resume/
   run-now controls against a real (non-critical) monitor, confirming the
   checkbox-gating actually blocks the Apply button until checked.
5. For public Marketplace listing: gather brand assets (icon, screenshots)
   per Zendesk's "Create app brand assets" guide, then submit for review via
   "Submit your app" — a manual review process, not a CLI command.
