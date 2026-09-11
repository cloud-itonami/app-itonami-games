# Operator quickstart

Get from a fresh clone to *"I have seen what this repo actually does"* in about
five minutes.

**Steps 1–3 were executed against tip `2720ae4` (`cloud-itonami/main`) on
2026-08-13/2026-09-07** (they are backend-only and unaffected by the frontend
migration below); **steps 4–5 were executed while landing the Svelte→cljs
frontend migration (ADR-2608260900) on 2026-09-07**, against the migration
branch built on top of that same tip. The output shown in each step is the
output observed. Two steps are expected to fail — that is stated where it
happens, and a failure there is the correct result, not a broken quickstart.
Step 6 was deliberately **not** run; it is marked.

Read [`../README.md`](../README.md) first if you have not. The short version:
this repo is an edge that forwards eight XRPC methods to an MCP router, the
methods are implemented elsewhere, nothing it declares currently resolves, and
as of 2026-09-07 the deployed surface is a static-assets-only cljs page with
**no** `/xrpc/*` handler (the previous SvelteKit BFF had one; it is preserved,
unwired, at `src/xrpc-proxy.ts`).

## 0. Prerequisites

Versions used for the runs recorded here. Nothing is pinned by the repo, so
newer versions are likely fine and older ones untested.

```
node      v26.3.0
npm       11.16.0
nbb       (on PATH)
clojure   (on PATH — shadow-cljs starts the cljs frontend's JVM/Clojure CLI
           side via this; see step 4)
```

`wrangler` is only needed for step 6, which was not run.

Set a shell variable for the app directory — every step after the first needs
it:

```bash
APP=appview/games-mcp-component
```

## 1. Is any of this live? (≈5s)

Run from the repo root:

```bash
kbb --backend sci docs/check-surface.cljk
```

Observed — **exit 1, and exit 1 is the expected result today**:

```
SCANNED	15 files	DECLARED-HOSTS	5
control	registry.npmjs.org	resolves

dispatcher.etzhayyim.com  NXDOMAIN  <- appview/games-mcp-component/src/app.ts
           etzhayyim.com  resolves  <- .../kotodama.jsonld, .../wrangler.jsonc
     games.etzhayyim.com  NXDOMAIN  <- .../kotodama.jsonld, .../src/app.ts, .../wrangler.jsonc
       mcp.etzhayyim.com  NXDOMAIN  <- .../src/xrpc-proxy.ts, .../wrangler.jsonc
  t9nt3w5u.etzhayyim.com  NXDOMAIN  <- .../wrangler.jsonc

4 of 5 declared hosts do not exist.
```

(Observed with a clean tree — no `node_modules/`, `.shadow-cljs/`, `out/`, or
`public/js/` present under `$APP/cljs`. Those are gitignored and, if present
from steps 4–5, add noise — build-cache copies of the same host strings — but
do not change the verdict.)

The script has three exit codes, and they were each demonstrated before this
document was written:

| exit | meaning |
|---|---|
| `0` | every declared host resolves — the README's status table is stale |
| `1` | at least one does not resolve — **today's state** |
| `3` | **could not answer**: the control host failed (no working DNS here), or zero hosts were extracted (wrong directory) |

Exit `3` exists so that "I could not measure" is never reachable from the same
exit code as "I measured, all fine".

## 2. Install and test the outer package (≈10s)

```bash
cd $APP
npm install --no-audit --no-fund
npm test
```

Observed — `added 65 packages`, then:

```
 RUN  v4.1.8

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

**Do not read this as coverage.** The single test is
`expect(true).toBe(true)`. It would still pass with every source file in the
repo deleted. See known gap 2 in the README. (This is the *backend* test
suite; the frontend has its own, real test suite — step 5.)

## 3. Watch the typecheck fail (≈3s) — expected

```bash
npm run typecheck        # still in $APP
```

Observed: **exit 1**, and instead of type errors it prints the `tsc` help text
(`--removeComments`, `--strict`, `--types`, …).

That is the whole finding: there is no `tsconfig.json` in `$APP`, so
`tsc --noEmit` receives no input files and falls back to printing usage.
Neither `src/app.ts` nor `src/xrpc-proxy.ts` has ever been typechecked by this
script (the latter also could not be — it imports `@sveltejs/kit`, a
dependency no longer installed anywhere in this repo now that `svelte/` is
gone; see its own header comment).

Confirm the cause directly:

```bash
ls tsconfig.json          # No such file or directory
```

`$APP/cljs/` has no `tsconfig.json` either — it is ClojureScript, not
TypeScript, and is not a target of this script at all (see step 4).

## 4. Build the artifact that actually deploys (≈20s)

`wrangler.jsonc` has **no `main`** — the deploy is Cloudflare Workers
static-assets-only, `assets.directory: "./cljs/public"`. This step builds
that directory's `js/app.js`.

```bash
cd $APP/cljs
npm install --no-audit --no-fund
```

**Inside the `com-junkawasaki` superproject, route the build through the
shared resource governor** — concurrent agent sessions on this machine must
not run two heavy builds at once:

```bash
node /path/to/com-junkawasaki/scripts/resource-guard.mjs run build -- amu compile --target wasm32-browser app
```

Standalone clones outside that superproject have no such script; use
`amu compile --target wasm32-browser app` directly.

The guard refuses rather than queues — **exit `2` means "the guard is busy",
not "the build failed"**; wait and retry.

Observed once the lock cleared — exit 0:

```
shadow-cljs - starting via "clojure"
[:app] Compiling ...
[:app] Build completed. (111 files, 110 compiled, 0 warnings, 14.60s)
```

Confirm the artifact wrangler expects now exists:

```bash
ls -la public/js/app.js     # ~3.3 MB (dev build; `npm run release` minifies)
```

## 5. Test the frontend, then probe it locally (≈30s)

```bash
node /path/to/com-junkawasaki/scripts/resource-guard.mjs run build -- amu compile --target wasm32-browser test
node out/tests.js
```

Observed:

```
Testing games.app-test

Ran 5 tests containing 14 assertions.
0 failures, 0 errors.
```

(`re-frame: Subscribe was called outside of a reactive context` lines printed
during the run are expected — the tests read subscriptions outside a Reagent
render, which re-frame warns about but still answers correctly. They are not
failures.)

Serve the built `public/` directory with a plain static file server (**not**
`wrangler dev` — see the note at the end of this step) and probe it:

```bash
cd public && python3 -m http.server 18734 --bind 127.0.0.1
```

In another shell:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18734/
# 200 — the static self-description page

curl -sI http://127.0.0.1:18734/ | grep -i content-type
# Content-type: text/html

curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18734/js/app.js
# 200

curl -s http://127.0.0.1:18734/ | grep -o 'id="app"[^<]*<'
# id="app">games-mcp-component loading…<
#     — the pre-hydration placeholder text; Reagent replaces it once app.js runs.
```

There is no `/xrpc/*` probe here (unlike the pre-migration quickstart) because
there is no handler for it in this deploy shape at all — see the README's
"Frontend migrated Svelte → cljs" section and known gap 5. A plain static file
server, not `wrangler dev`, is used deliberately: this step verifies the built
assets, not a simulated Worker runtime.

## 6. Deploy — NOT RUN

```bash
npx wrangler deploy        # from $APP
```

This was **not** executed, and you should not execute it to "check the
quickstart". `wrangler.jsonc`'s new shape (no `main`, `assets.directory:
./cljs/public`, `APP_FRAMEWORK: cljs-reagent-re-frame-jp-go-dds`) is
**UNVERIFIED** against a real `wrangler deploy`/`wrangler dev` — see README
known gap 6. Deploying would also publish `games.etzhayyim.com` and
`t9nt3w5u.etzhayyim.com`; the eight methods still resolve nowhere
(`mcp.etzhayyim.com` is NXDOMAIN — step 1), and now there is not even a proxy
route to fail against it.

The workspace also requires any deploy to run from a checkout that contains
`origin/main` (here, `cloud-itonami/main`), since deploys have no fast-forward
check and the last writer wins.

## What you now know

- The repo is an edge, not a service. The eight methods live behind
  `mcp.etzhayyim.com` (step 1) and are implemented nowhere in this tree.
- `src/app.ts` is dead code — untouched by the frontend migration, and never
  part of the deployed surface either before or after it.
- The deployed surface changed shape on 2026-09-07: SvelteKit Worker (script +
  assets) → static assets only (no script). The static self-description page
  still works (step 5); `/xrpc/*` currently has no handler at all, where
  before it had one that could not succeed anyway (step 1).
- Neither the typecheck (step 3) nor the backend test suite (step 2) would
  notice if `src/app.ts` broke. The frontend does have a real test suite now
  (step 5), covering the ported page's data and view logic.
- Nothing is live, and nothing can be until `mcp.etzhayyim.com` exists (step 1)
  **and** the proxy is re-wired into the deploy (known gap 5).
