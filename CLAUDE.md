# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`discourse-webhook-github-issue` is a small Koa (v2) webhook receiver. It listens for Discourse "post created" webhooks, scans the post's rendered HTML for links to GitHub pull requests (matching a specific `owner/repo`), and if a matching PR doesn't already have a comment linking back to that Discourse post, creates one via the GitHub API.

The entire application logic lives in `index.js` (~140 lines) — there is no framework scaffolding, no database, and no test suite. `views/`, `public/`, and the screenshots in `docs/` are unused leftovers from the Heroku "node-js-getting-started" template that this repo was bootstrapped from; `index.js` never requires or serves them.

## Commands

- Install dependencies: `npm install`
- Run the server: `npm start` (runs `node index.js`)
- No lint, build, or test scripts are defined in `package.json` — there is nothing to run for those.

## Configuration (environment variables)

All configuration is read from env vars at startup in `index.js` (lines 12-18), with no runtime validation — a missing var silently falls back to an empty string/default rather than erroring:

- `URL` — webhook path to listen on (default `/discourse-webhooks`)
- `PORT` — port to listen on (default `4000`)
- `SECRET_KEY` — must match the webhook secret configured in Discourse; used to verify the HMAC signature
- `DISCOURSE_URL` — base URL of the Discourse instance (no trailing slash), used to build the link back to the post
- `GITHUB_USERNAME` — GitHub owner/org to match PR links against and to comment as
- `GITHUB_REPO` — GitHub repo (under `GITHUB_USERNAME`) to match PR links against
- `GITHUB_ACCESS_TOKEN` — personal access token used to authenticate GitHub API calls

For local development, copy these into a `.env` file (gitignored) or export them in the shell — there's no dotenv loader wired up in `index.js`, so a `.env` file only helps if something like `foreman`/Heroku local sources it (see `Procfile`).

## Request flow (the part worth understanding end-to-end)

The single route handler (`router.post(watchUrl, ...)` in `index.js`) does the following, in order:

1. **Filter by event type**: only `x-discourse-event-type: post` headers are processed; anything else short-circuits with `'No interests.'`.
2. **Verify HMAC signature**: computes `sha256=` HMAC of the raw request body using `SECRET_KEY` and compares it against the `x-discourse-event-signature` header. On mismatch it responds with a debug body (signature, computed hash, raw body) rather than a 4xx — there's no auth failure status code.
3. **Extract PR links**: loads `body.post.cooked` (the rendered HTML of the Discourse post) with `cheerio`, and collects every `<a href>` matching `githubPullRequestRegexp` (built from `GITHUB_USERNAME`/`GITHUB_REPO`) into a `Set` (dedupes repeated links to the same PR).
4. **Build the Discourse post path**: `/t/{topic_slug}/{topic_id}` optionally suffixed with `/{post_number}` when the post isn't the first in the topic.
5. **For each matched PR** (in parallel, via `Promise.all`): fetch existing comments (paginating with `github.hasNextPage`/`getNextPage`), check whether any existing comment body already contains the post path, and only if not found, create a new comment of the form `` `@username` mentioned this pull request. See {DISCOURSE_URL}{postPath}. ``

### Things to watch for when modifying this flow

- The raw-body capture happens in Koa middleware *before* `koa-bodyparser` runs (the app's `.use()` chain at the bottom of `index.js`), because the HMAC must be computed over the exact raw bytes, not the parsed JSON.
- The pagination loop in step 5 (`while (github.hasNextPage(json))`) fires `getNextPage` calls without awaiting them before checking the loop condition again — this is pre-existing async iteration logic, be careful about ordering assumptions if touching it.
- `ctx.body` is reassigned per-PR inside the `forEach`/`.then()` chain, so with multiple matched PRs only the last-resolved one's status message is reflected in the response body; the GitHub comment side effects still happen for all of them.
- The `github` npm client (v2, from the `github` package — the old REST client, not `@octokit/rest`) is instantiated fresh per-request.

## Deployment

Designed to run on Heroku (`Procfile`, `app.json`) but is a plain Node/Koa app that can run anywhere `node` is available with the env vars above set.
