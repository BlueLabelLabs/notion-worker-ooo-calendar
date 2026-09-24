# notion-worker-ooo-calendar

Turns Notion out-of-office requests into Microsoft 365 calendar events, with
holiday syncs, Slack notifications, and a reconciliation sweep.

**`README.md` is current and the most thorough of the five (451 lines)** — how it
works, the three databases, ownership rules, Slack behavior, setup, known edge
cases, platform gotchas. Read it first. This file carries the cross-cutting
conventions that live outside any one repo.

## Brand rule

Always write **"BlueLabel"** — never "Blue Label", "Blue Label Labs", or "BLL".

## Notion Workers platform constraints (workspace-wide)

These bite every new worker in workspace `b006331e-8616-4685-aaaf-0cc33dc9b6ef`.
Learned across the Kantata, Read.ai, and DPPT workers.

1. **`worker.automation` is DISABLED for this workspace.** Triggers must be a
   `worker.webhook` called by a Notion in-app "Send webhook" automation.
2. **A managed `worker.database`'s non-title properties are read-only to the API**
   (400 "read-only"). Any database the worker writes columns to must be **native**.
   Managed DBs are useful only as empty scheduling anchors, because `worker.sync`
   is the only scheduler and requires one.
3. **"Send webhook" payloads carry only a page reference with an empty
   `properties` object.** Always `pages.retrieve` the row.
4. **Notion automations have no "page deleted" trigger**, so deletion handling
   always needs a scheduled sweep.
5. **As of 2026-07-16 the page CONTENT of worker-managed (synced) rows is
   read-only to integrations.** Property writes on those rows still work. A
   worker that needs to write blocks onto a row has to use a native companion
   database instead — this is why `notion-worker-dppt` writes its parsed tables
   to a "DPPT Details" DB rather than onto Kantata Engagement pages
   (`src/lib/dpptDetails.ts` in that repo).

Also: a scheduled `worker.sync` needs `NOTION_API_TOKEN` set — webhooks get
`context.notion` injected, syncs do not. And `ntn workers exec` never initializes
the SDK pacer runtime (`Pacer "..." not found`), so pacing must degrade gracefully.

## House conventions across the five workers

Reference implementations, richest first: `notion-worker-kantata`,
`notion-worker-meetings`, `notion-worker-dppt`.

ESM + NodeNext (`.js` import extensions on `.ts` sources), TypeScript strict with
`noUncheckedIndexedAccess`, Node 22 via `.nvmrc`, `@notionhq/workers` ^0.4.0.
One `src/worker.ts` holds the single `Worker` instance plus OAuth and pacer
handles; `src/index.ts` imports every capability module for its registration
side-effect, then `export default worker`. Capabilities split across `src/tools/`,
`src/syncs/`, `src/webhooks/`; shared code in `src/lib/`.

Every repo's `src/lib/` carries: a normalized `<Api>ApiError` in `errors.ts` with a
`kind` union and a `retryable` flag; `retry.ts` with exponential backoff and full
jitter honoring `Retry-After`, retrying only network/429/5xx; lazy env getters in
`env.ts` with a `requireEnv()` that names the missing key; one API client every
call funnels through; and a best-effort `slack.ts` that logs and never throws.

Secrets via `ntn workers env set`, documented in `.env.example`, `.env` gitignored.
Tests are `node:test` + `node:assert/strict` run through `tsx --test`, colocated as
`*.test.ts`, exercising pure functions with injected dependency bundles (see
`lib/router.test.ts` in the meetings worker). Test files are excluded from tsconfig
`include`. No vitest, no jest, no linter.

READMEs carry a "Platform notes / gotchas (learned the hard way)" section. Read it
before starting a new worker.

## Notion MCP rewrites public page links

When writing markdown links through `notion-update-page`, Notion normalizes any
URL it recognizes as an internal page: it **strips the query string** and rewrites
a public `bluelabellabs.notion.site/<id>` host to `app.notion.com/p/<id>`.
Re-applying the correct URL does not stick — the normalizer wins every time, and
the write reports success either way.

This matters because a Notion form's public link
(`bluelabellabs.notion.site/<id>?pvs=105`) becomes an `app.notion.com` link that
demands a login, defeating the point for Flex team members without a Notion
account. It broke the Flex request form link on the OOO process page.

**Never hand-write a link to a public notion.site URL through the MCP tool.**
Either put the URL in an inline code span (backticks survive verbatim, since code
spans are not auto-linked) and ask the user to convert it in the Notion UI, or have
the user paste it directly — the Notion editor preserves notion.site links when a
human pastes them. Always re-fetch the page after writing links and verify the href.

## `ntn` CLI auth is keychain-bound

`NOTION_API_TOKEN` does **not** override the keychain for `ntn workers` commands
in v0.23.9, despite what `ntn login --help` claims. Verified: with a throwaway
`HOME` and a valid token from `ntn auth token`, both `ntn doctor` and
`ntn workers ls` fail with "Failed to fetch token from keychain".
`NOTION_WORKSPACE_ID` *is* honored, which proves env vars reach the process.

The headless path is **`NOTION_KEYRING=0`**, which reads
`~/.config/notion/auth.json`. Generate it once with `NOTION_KEYRING=0 ntn login`.
The file is a flat `{"<workspace-uuid>": "<token>"}` map with no expiry, refresh
token, or machine binding, so it transplants to another host as-is.

Also: `ntn update` refuses to self-update an npm-installed copy. Use
`npm install -g ntn@latest`.

## Deploying

```bash
npm run deploy      # local, interactive
npm run deploy:ci   # cloud sessions — adds --yes
```

`ntn workers deploy` prompts for confirmation because this worker declares a
managed `worker.database`, and a cloud session cannot answer an interactive
prompt — it fails with "Deploying this worker with linked databases requires
confirmation". Use `deploy:ci` there.

`--yes` skips a guard against **releasing** a managed database, which happens when
new code stops declaring one that is currently linked. Before using it, confirm the
`worker.database(...)` declarations are unchanged — do not assume it from the diff
looking small.
