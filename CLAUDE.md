# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single static page (`index.html`) served via GitHub Pages. There is no build step, no
package.json, no framework, and no test suite — it's plain HTML/CSS/vanilla JS in one file.
It is an admin control panel for dispatching and monitoring GitHub Actions `workflow_dispatch`
runs in a *separate* repository, `Clinic-Software-Hub/clinicsoftwarehub`, which holds the
actual clinic software's runtime workflow YAMLs (`runtime-*.yml`). This repo does not contain
those workflows — only the UI that triggers and streams logs for them via the GitHub REST API.

## Development

There is no build/lint/test tooling. To work on this:
- Edit `index.html` directly.
- Preview by opening the file in a browser directly, or serving it locally, e.g. `python3 -m http.server` from the repo root.
- Deployment is just pushing to `main` — GitHub Pages serves `index.html` as-is.

## Architecture (all in `index.html`)

Everything client-side, no backend of its own. The page calls `api.github.com` directly from
the browser using a user-supplied fine-grained PAT (pasted into `#patInput`, kept only in
`sessionStorage`, never sent anywhere but GitHub's API).

- **`WORKFLOWS` array**: single source of truth for the cards rendered into `#workflowList`.
  Each entry maps to a `runtime-*.yml` workflow file in the `clinicsoftwarehub` repo, its
  required `workflow_dispatch` inputs, and which input key carries the environment name
  (`envKey` — note it's inconsistently `"enviroment"` (sic) vs `"environment_name"` depending
  on the workflow, so don't "fix" the typo without checking the actual workflow file first).
  A `type: "sequence"` entry (see "Restart Nginx + Dozzle") instead lists ordered `steps`
  dispatched one after another with a fixed `delayMs` between them, sharing one log tab.
- **Dispatch flow** (`buildCard` / `buildSequenceCard` click handlers):
  1. Open a blank tab synchronously inside the click handler (`window.open("", "_blank")`) so
     browsers don't block it as a popup — it gets a real URL/content only later.
  2. POST to `.../actions/workflows/{file}/dispatches`. A 204 means GitHub accepted it, but the
     response carries no run ID.
  3. `findNewRun` polls `.../workflows/{file}/runs?event=workflow_dispatch` (up to
     `POLL_MAX_ATTEMPTS` × `POLL_INTERVAL_MS`) matching the newest run created at/after the
     dispatch timestamp, to work around that missing run ID and around GitHub's dispatch-to-run
     latency (worse when another `runtime-*.yml` run holds the shared `global-deployment`
     concurrency group).
  4. Once found, `pollRunLogs`/`renderLogsInTab` streams the run's status and each job's raw
     logs (ANSI-stripped) into the previously opened tab, polling every `LOG_POLL_MS` until the
     run completes. The `jobs/{id}/logs` endpoint can redirect to a signed URL that doesn't
     always return CORS headers to arbitrary origins — if that fetch fails, the UI shows a
     fallback message pointing at "Open in GitHub Actions" rather than failing silently.
- **Tenant ID autocomplete** (`populateTenantIdList`): fetches the public, unauthenticated
  `https://clinicsoftwarehub.online/status.json` to fill a shared `<datalist id="tenantIdList">`
  used by every `tenant_id` input. If that fetch fails (network error, non-2xx, malformed JSON,
  or CORS since the VPS nginx doesn't send `Access-Control-Allow-Origin` for cross-origin Pages
  requests), it fails silently and the inputs just behave as plain free-text fields.
- **Persisted fields**: PAT, environment name, and branch inputs are persisted to
  `sessionStorage` via `bindPersisted` so they survive a page reload within the same tab session.
- **Status Dashboard card** is just a static link to `https://clinicsoftwarehub.online/status.html`;
  it has no dispatch logic.

When adding a new workflow card, add an entry to `WORKFLOWS` with the correct `file`, `envKey`
(check the actual workflow's declared inputs in the `clinicsoftwarehub` repo — don't assume),
and `inputs`; the render/dispatch/log-streaming logic is fully generic and needs no changes for
a normal (non-sequence) workflow.
