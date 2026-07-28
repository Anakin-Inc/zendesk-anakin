# Zendesk — submission instructions

Real, self-serve, CLI-buildable — Zendesk apps are built and validated with
`zcli` (the `@zendesk/zcli` npm package) against a real Zendesk account, then
either installed privately on that account or submitted to the Zendesk
Marketplace via a manual review step in the Zendesk admin UI (not a repo/PR
process, unlike the Dagster submission in this batch).

## What's here

```
manifest.json          App metadata, ticket_sidebar location, the secure
                        apiKey parameter, domainWhitelist: ["api.anakin.io"]
assets/iframe.html      The whole app — form UI + JS, ~400 lines, no build step
translations/en.json    name / short_description / long_description /
                        installation_instructions / parameter label+help —
                        the fields Zendesk's own deploying doc says a
                        listing requires
```

Structure and every API surface used (`location.support.ticket_sidebar`
with `url`/`flexible`, `parameters` with `secure: true`, `domainWhitelist`,
`client.request()` with the `{{setting.NAME}}` placeholder and `secure:
true`, `client.set('comment.type', ...)`, `client.invoke('comment.appendText',
...)`, the `zaf_sdk.min.js` CDN script tag, `zcli apps:new` /
`apps:server` / `apps:validate` / `apps:create` / `apps:package`) was
checked against Zendesk's real developer docs (fetched live — see
citations in README.md and below), a real manifest.json pulled from
`zendesk/demo_apps` (`v2/support/modal_sample_app/manifest.json`, fetched
via GitHub's raw content API), and `zendesk/zendesk_apps_support`'s own
manifest validation test spec — not assumed or reconstructed from memory.

## The one real design question in this task, answered from source

**How should an iframe app safely call a third-party API that needs a
secret key — proxy the value server-side, or read it into the browser and
call the API directly?** Zendesk's own "Making API requests from a Zendesk
app" doc answers this directly: declare the setting as `{"type": "text",
"secure": true}` in `manifest.json`, then reference it in a `client.request()`
call as a `{{setting.apiKey}}` placeholder with `secure: true` set on the
request options. The literal quote: "the settings for client-side HTTP
requests are visible in a browser's dev tools... only the placeholder, not
the value, is displayed in the browser's dev tools. The Zendesk proxy
server later inserts the setting's value outside the browser." So the
answer is unambiguous — the key is never read into the iframe's JS at all,
which is a stronger guarantee than "the JS can read it via
`client.metadata()`" (that call exists and does return `metadata.settings`,
but a secure setting's real value is deliberately excluded from it — that's
the point of `secure`, confirmed by the same doc, not an assumption this
submission is making).

This also resolves the CORS question the task raised as a possible blocker:
`client.request()` is a Zendesk-proxied call, not a browser `fetch()`/XHR —
per Zendesk's docs it "handles cross-origin restrictions through Zendesk's
proxy infrastructure," gated only by listing the external domain in
`manifest.json`'s `domainWhitelist` (done here: `["api.anakin.io"]`). Since
the actual HTTP call to `api.anakin.io` is issued by Zendesk's servers, not
the sandboxed iframe origin, api.anakin.io's CORS configuration for
browser-origin requests is irrelevant — there was nothing to check on
Anakin's side, and no proxy of our own to build. `assets/iframe.html` uses
`client.request()` for every call (submit, poll, search) for this reason,
never a raw `fetch()`.

## Verified, not assumed

- **`manifest.json` parses as valid JSON** — `python3 -c "import json;
  json.load(open('manifest.json'))"`, exit 0.
- **`translations/en.json` parses as valid JSON** — same check, exit 0.
- **`assets/iframe.html`'s embedded `<script>` is syntactically valid
  JavaScript** — extracted the script block and ran `node --check` on it
  directly, exit 0 (Node v25.2.1 in this sandbox).
- **`manifest.json`'s shape was checked against a real, live Zendesk sample
  app** — `zendesk/demo_apps`'s `v2/support/modal_sample_app/manifest.json`,
  fetched via GitHub's raw content API, matches this submission's
  `author`/`defaultLocale`/`location.support.ticket_sidebar`/`version`/
  `frameworkVersion` shape field-for-field.
- **The `parameters` schema (`type`, `secure`) was checked against
  Zendesk's own validator test spec**, not just prose docs —
  `zendesk/zendesk_apps_support`'s `spec/validations/manifest_spec.rb`
  fixtures confirm `{"name": "api_token", "type": "text", "secure": true}`
  is a valid parameter and that `password` is not a real `type` value (only
  `text`, `number`, `oauth`, `hidden`, and a few others are enumerated) —
  this directly corrected an assumption in the original task brief, which
  named `type: password` as the expected mechanism.
- **The `comment.text` / `comment.type` / `comment.appendText` ticket
  sidebar paths are real, documented ZAF Apps Support API paths** —
  confirmed against the "Ticket and New Ticket sidebar" API reference page,
  including the exact allowed `comment.type` values (`internalNote`,
  `publicReply`, plus the channel-specific ones) and that `appendText` is
  an `invoke`-only action distinct from the gettable/settable `comment.text`.
- **`npx @zendesk/zcli --version` installs and runs for real** —
  `@zendesk/zcli/1.1.4 darwin-arm64 node-v25.2.1`, a genuine `npx` install
  and execution in this sandbox, not assumed to exist.
- **`npx @zendesk/zcli apps:validate .` and `apps:package .` were both
  actually run against this directory** — both reach real, documented zcli
  commands (confirmed their `--help` output matches the command reference)
  and both ultimately fail with `Authorization failed. Set the following
  environment variables: ZENDESK_SUBDOMAIN, ZENDESK_EMAIL,
  ZENDESK_API_TOKEN. Or try logging in via zcli login -i` — a real
  credential requirement, not a bug in this submission. But before hitting
  that wall, the commands got further than "just an error": both runs left
  a real `tmp/app-<timestamp>.zip` behind (`tmp/` is now in `.gitignore`,
  the zips were deleted after inspection), and unzipping one showed zcli
  had already locally assembled the exact right structure —
  `manifest.json`, `assets/iframe.html`, `translations/en.json`, correct
  paths, nothing missing or extra — before making the network call that
  needs an authenticated account. That's real signal that the local file
  layout is correct at the level zcli itself checks; only the remote
  validate/upload call is blocked on credentials. There is no fully
  offline `zcli` command that both packages *and* reports validation
  errors without an account.
- **The CDN script tag URL was checked against Zendesk's own docs** —
  `https://static.zdassets.com/zendesk_app_framework_sdk/2.0/zaf_sdk.min.js`
  pinned to the 2.0 major (not the `.../2/...` floating-minor URL, which
  the docs note "may include breaking changes" — not appropriate for a
  submission meant to be stable).
- **Precedent**: Yext's "AI Search" app, live in the real Zendesk
  Marketplace, adds a third-party-API-backed AI search box to the ticket
  sidebar with an agent-supplied key — located via web search, confirming
  this app category is accepted and precedented. Its source isn't public;
  it informed the *shape* decision only (see README.md "Precedent"), nothing
  was copied.

## Not done

- **Never authenticated `zcli` against a real Zendesk account** — no
  Zendesk subdomain/email/API-token credentials exist in this environment.
  `apps:validate` and `apps:package` (Zendesk's own real, authoritative
  local-ish validators) could not actually be run to completion — only
  confirmed they're the right commands and that they fail on auth, not on
  the app's own content.
- **Never installed the app on a real Zendesk instance or opened it inside
  an actual ticket sidebar** — so the UI has not been eyeballed rendering
  in the real product, only reasoned about from the CSS and Zendesk's
  documented iframe sizing (`flexible: true`) behavior.
- **Never called Anakin's live API from this app** — no real Anakin API key
  was exercised end-to-end through the `client.request()` + `secure: true`
  path, so the actual runtime substitution of `{{setting.apiKey}}` — which
  only happens inside Zendesk's own proxy, not locally reproducible without
  a live install — is unverified beyond matching the documented mechanism
  exactly.
- **`api.anakin.io`'s CORS headers for direct browser-origin requests were
  not checked** — deliberately: per the design decision above, ZAF apps
  never need this checked, since `client.request()` doesn't originate as a
  browser-CORS-governed request in the first place. Flagging this
  explicitly rather than silently treating "didn't check" as "not needed"
  — if this API surface is ever called from a *non-ZAF* browser context
  (e.g. a plain web page), CORS would need checking separately; that's out
  of scope for this submission.
- **No app icon/brand assets** — `manifest.json` has no `iconLocations`,
  and there's no `assets/icon.png`/`assets/logo-small.png`. Fine for a
  private install and for this submission's purpose; required before a
  public Marketplace listing per Zendesk's "Create app brand assets" guide.

## Steps (needs the account owner)

1. `npx @zendesk/zcli login -i` (or export `ZENDESK_SUBDOMAIN`,
   `ZENDESK_EMAIL`, `ZENDESK_API_TOKEN`) against a real Zendesk account —
   Support Professional plan or above (private app upload requirement).
2. `npx @zendesk/zcli apps:validate .` — confirms the manifest/assets for
   real against Zendesk's servers, the one check this session couldn't run.
3. `npx @zendesk/zcli apps:create .` — uploads and privately installs the
   app.
4. In a real ticket's sidebar, install with a real Anakin API key from
   [anakin.io/dashboard](https://anakin.io/dashboard) and exercise all
   three tabs (Scrape URL, AI Search, Agentic Search) plus both insert
   buttons against live data — the one thing this session couldn't check.
5. Add `assets/icon.png` (128×128) and register it via `iconLocations` in
   `manifest.json` if pursuing a public Marketplace listing, then submit
   for review through the Zendesk admin UI per "Submit your app."
