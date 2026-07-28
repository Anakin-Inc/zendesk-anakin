# zendesk-anakin

A Zendesk Support app (built on the Zendesk Apps Framework, ZAF v2) that adds
[Anakin](https://anakin.io) — web scraping, AI-powered search, and
multi-stage agentic research — to the ticket sidebar.

## What's covered

A ticket-sidebar iframe app with three tabs, matching the same baseline
every other single-purpose Anakin integration built this session covers:

- **Scrape URL** — submits `POST /v1/url-scraper`, then polls
  `GET /v1/url-scraper/:jobId` until the job completes or fails. Shows the
  returned markdown.
- **AI Search** — calls `POST /v1/search`. Synchronous, no polling. Costs 3
  credits per call.
- **Agentic Search** — submits `POST /v1/agentic-search`, then polls
  `GET /v1/agentic-search/:jobId`. Shows the AI-generated summary plus, if a
  JSON Schema was supplied, the structured data. Costs 10 credits.

Once a result is showing, **Insert as internal note** / **Insert as public
reply** append it into the ticket's comment editor via the documented
`comment.type` / `comment.appendText` ZAF paths — the agent still has to hit
Zendesk's own Submit button, this only fills the box.

## Files

```
manifest.json            App metadata, ticket_sidebar location, the secure
                          apiKey parameter, domainWhitelist
assets/iframe.html        The entire app: form UI + JS, single file
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
   three tabs against live data.
5. For public Marketplace listing: gather brand assets (icon, screenshots)
   per Zendesk's "Create app brand assets" guide, then submit for review via
   "Submit your app" — a manual review process, not a CLI command.
