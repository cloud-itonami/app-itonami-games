# games

**A thin edge for a game-platform API that has no back end here.** It accepts
eight XRPC methods — game titles, scores, leaderboards, play sessions,
achievements — validates nothing, stores nothing, and (when wired) forwards
each call to an MCP router on another host. The repo holds the *edge and the
contract*: a static self-description page, a BPMN file naming the eight
processes, and the identity metadata that declares the actor. **The eight
methods themselves are not implemented anywhere in this repo.**

It is `kind :app` (`README.edn`), extracted verbatim from `etzhayyim/root` at
`60-apps/etzhayyim-project-games` (`migration.edn`, source revision
`7c815d8a21`, 17 files / 71,434 bytes).

The name says the subject, not the role. Read it as *the games edge*, not as a
game or a game engine — nothing here renders, simulates, or plays anything.

## Frontend migrated Svelte → cljs (2026-09-07, ADR-2608260900)

The public self-description page was a SvelteKit route
(`appview/games-mcp-component/svelte/src/routes/+page.svelte`), built into a
Cloudflare Worker at `svelte/.svelte-kit/cloudflare/_worker.js`. It is now a
reagent + re-frame view built from `jp-go-dds` (デジタル庁デザインシステム)
hiccup, at `appview/games-mcp-component/cljs/src/games/app.cljk`, per this
workspace's repo-wide single-page-app + cljs-first UI rules
(ADR-2608080100, ADR-2608260900). The `svelte/` directory is deleted.

**This changed what is deployed, not just how it is built.** The old
`_worker.js` was one script that served both routes: `GET /` (the static page)
and `POST /xrpc/[...path]` (a real proxy to `AGENTGATEWAY_MCP_ROUTER_URL`).
`wrangler.jsonc` now has no `main` at all — `assets.directory` points at
`./cljs/public`, so the deploy is Cloudflare Workers **static assets only**,
with no Worker script. **`/xrpc/*` currently has no handler in this repo's
deploy config.** The proxy route's source is preserved, unmodified, at
`appview/games-mcp-component/src/xrpc-proxy.ts` (moved verbatim from
`svelte/src/routes/xrpc/[...path]/+server.ts`, which imported
`@sveltejs/kit` types that no longer resolve anywhere in this repo since
`svelte/` — the only place that dependency was declared — is gone) but it is
**not wired to anything**. Reviving `/xrpc/*` (as a Worker script that also
calls `env.ASSETS.fetch()` for everything else, or some other shape) is an
unresolved product decision — see [Known gaps](#known-gaps) item 5.

No `wrangler deploy` was run as part of this migration — see
[Known gaps](#known-gaps) item 6.

## Status: not live — measured 2026-09-07

**Nothing this repo declares is reachable on the internet today,** and neither
is either of the two upstreams it would forward to. This is a measurement taken
at the tip this README landed on, not an inference from the code:

| host | declared in | resolves? |
|---|---|---|
| `etzhayyim.com` | zone for both routes | **yes** |
| `games.etzhayyim.com` | `wrangler.jsonc` route, `kotodama.jsonld`, `src/app.ts` | **NXDOMAIN** |
| `t9nt3w5u.etzhayyim.com` | `wrangler.jsonc` route | **NXDOMAIN** |
| `mcp.etzhayyim.com` | `AGENTGATEWAY_MCP_ROUTER_URL` — the proxy's would-be upstream, now unwired | **NXDOMAIN** |
| `dispatcher.etzhayyim.com` | `src/app.ts` upstream (that file is not deployed) | **NXDOMAIN** |

The parent zone resolves and a control lookup against an unrelated host
succeeded, so these are genuine absences and not a local DNS fault.

**Re-measure before trusting this table** — it is dated, and dated tables rot:

```bash
nbb docs/check-surface.cljk      # exit 1 today; exit 0 would mean this table is stale
```

The consequences are concrete:

- **The app is not deployed.** Neither route host exists.
- **The upstream is absent**, so `/xrpc/*` could not succeed even if it were
  wired — and today it is not wired at all (see above).

## What is actually in here

Four pieces. Only the second one is deployed.

### 1. `appview/games-mcp-component/src/app.ts` — a Worker that is **not deployed**

A handler that answers `/health` and `/_app/meta`, and proxies
`/xrpc/com.etzhayyim.apps.games.*` (GET or POST) to `DISPATCHER_URL`, attaching
an `x-internal-secret` header.

**Nothing references this file.** `wrangler.jsonc` has no `main` (the frontend
migration removed the only value it ever had, `svelte/.svelte-kit/…`); the
deploy is static-assets-only, built from `cljs/`. A repo-wide search for
`app.ts` outside `node_modules/` and build output returns no hits. This file
is untouched by the frontend migration — see
[Do NOT touch backend TypeScript](#known-gaps) note in item 5.

Two things follow, both true both before and after this migration:

- **`/health` and `/_app/meta` do not exist in the deployable surface.** Both
  would 404 (no Worker script at all now, not just a route that omits them).
  Any monitor pointed at them is measuring the absence of a route, not the
  health of a service.
- **The two edges never agreed on the upstream.** This file targets
  `dispatcher.etzhayyim.com` and speaks plain XRPC; the (now unwired) proxy
  targeted `mcp.etzhayyim.com` and spoke MCP `tools/call`. They were never two
  paths to the same place.

It is also unchecked: `npm run typecheck` cannot run (see
[Known gaps](#known-gaps)).

### 2. `appview/games-mcp-component/cljs/` — the static info page that **is** deployed

One view, reagent + re-frame, rendered from `jp-go-dds` hiccup — a faithful
port of the deleted SvelteKit page's four sections (top / facts / public
routes / runtime bindings / source), field for field, with two stale literals
corrected (`routeCount`/`routes` and `vars` were hardcoded empty in the Svelte
source; they now read the same values `wrangler.jsonc` actually declares — 2
routes, 8 vars). See the docstring in `cljs/src/games/app.cljs` for the full
provenance note.

`GET /` is the only route this surface answers. **There is no `/xrpc/*`
handler in the deployed config** (see the migration note above) — this is a
change in behaviour from the deleted SvelteKit BFF, which did export a real
`POST /xrpc/[...path]` proxy.

Build/test (from `cljs/`):

```bash
npm install
npm run build   # shadow-cljs compile app  -> public/js/app.js
npm test        # shadow-cljs compile test && node out/tests.js
```

Observed on this migration's landing commit: build `111 files, 110 compiled,
0 warnings`; test `Ran 5 tests containing 14 assertions. 0 failures, 0
errors.` (the `re-frame: Subscribe was called outside of a reactive context`
lines the test run prints are expected — `cljs.test` assertions read
subscriptions outside a Reagent render, which re-frame warns about but still
answers correctly; they are not failures).

### 3. `bpmn/games.bpmn` — the eight processes

Eight `bpmn:process` definitions, each a start → single `serviceTask` → end,
with a `zeebe:taskDefinition` naming the method. The set matches the eight
capabilities in `wrangler.jsonc` and `kotodama.jsonld` exactly:

```
createTitle  listTitles  recordScore  listScores
getLeaderboard  createSession  listSessions  getAchievements
```

These are process *declarations*. No worker in this repo implements a
`taskDefinition`, and no engine here executes them.

### 4. Identity — `kotodama.jsonld`, `README.edn`, `migration.edn`, `NOTICE`

Declares `did:web:games.etzhayyim.com`, an autonomous bot actor operated by
`amanomibashira`, classification `public`, protocols `xrpc` + `mcp`, and a
`subscribeRepos` trigger on the `…games.title` and `…games.score` collections.
Licensed Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1.

## Where the logic actually lives

Not here, and — as of the measurement above — not anywhere reachable:

| concern | where the config says it lives |
|---|---|
| the eight methods | behind `mcp.etzhayyim.com` (MCP router → pod-side LangServer) — **if** the proxy is ever re-wired |
| process execution | a Zeebe-compatible engine consuming `bpmn/games.bpmn` |
| persistence | none declared — no D1, KV, R2 or DO binding in `wrangler.jsonc` |

## Known gaps

Each of these was observed, not inferred. None is fixed by this README.

1. **`npm run typecheck` cannot run.** There is no `tsconfig.json` in
   `appview/games-mcp-component/`, so `tsc --noEmit` gets no input files, prints
   its help text, and **exits 1**. It has never checked `src/app.ts`, and it
   does not check `src/xrpc-proxy.ts` either (nor could it usefully — that
   file imports `@sveltejs/kit`, a dependency no longer installed anywhere in
   this repo now that `svelte/` is gone; see the migration note above and the
   file's own header comment). `cljs/` is ClojureScript, not TypeScript, and is
   checked by its own build/test commands (item 2 in
   [What is actually in here](#what-is-actually-in-here)), not by this script.
2. **The backend test suite is a placeholder.** `test/games.test.ts` asserts
   `expect(true).toBe(true)`. It passes, and it would keep passing if every
   source file in the repo were deleted. This is unrelated to the frontend's
   own test suite (`cljs/test/games/app_test.cljs`), which is real and passing
   — see item 2 above.
3. **The org and the identity disagree.** The repo is `cloud-itonami/games`;
   `migration.edn` names its destination `etzhayyim/com-etzhayyim-app-games`,
   `README.edn` names it `com-etzhayyim-app-games`, and every host and DID is
   under `etzhayyim.com`. `MIGRATION-TODO.md` records the codemod as pending.
4. **`src/app.ts` is dead** — see §1. Wiring it, deleting it, or reconciling the
   two upstreams is an owner decision; this README only records that the choice
   has not been made. The frontend migration did not touch this file.
5. **`/xrpc/*` has no handler in the deployed config, as of this migration.**
   The previously-deployed SvelteKit BFF had one (a real, if unreachable-today,
   proxy to `AGENTGATEWAY_MCP_ROUTER_URL`); the cljs static-assets-only
   replacement does not. The proxy's source is preserved verbatim at
   `src/xrpc-proxy.ts` for exactly this reason, but re-wiring it — as a Worker
   script alongside `env.ASSETS.fetch()`, or some other shape — is an
   unresolved product decision, not made by this migration (per the migration
   task's own instructions: do not try to make it run).
6. **The wrangler config change is UNVERIFIED.** No `wrangler deploy` or
   `wrangler dev` was run against the new `wrangler.jsonc` (no `main`,
   `assets.directory: ./cljs/public`, `APP_FRAMEWORK: cljs-reagent-re-frame-jp-go-dds`).
   Confirming it deploys cleanly is left to whoever next runs step 6 of
   `docs/operator-quickstart.md`.

## Verify any of the above

Start with [`docs/operator-quickstart.md`](docs/operator-quickstart.md). Every
command in it was executed against the pre-migration tip `2720ae4`
(`cloud-itonami/main`), and the output shown there is the output observed —
except where marked as the new cljs build/test output captured while landing
this migration.
