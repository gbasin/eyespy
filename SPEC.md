# Eyespy: Open-Source Visual Regression Testing Platform

## Context

Frontend teams ship visual regressions because their testing tools are inadequate. Meticulous AI solved this with: record real user sessions, replay deterministically, diff screenshots, use V8 coverage to select a minimal test suite. But it's closed-source, cloud-only, and expensive.

No open-source alternative provides the full pipeline. Lost Pixel, Argos CI, and Pixeleye do visual regression but require manually-written tests. Nobody has recording + coverage-based test selection + deterministic replay + visual diffing in open source.

**License**: MIT. Maximum adoption.

---

## Decisions (All Rounds)

- **Name**: Eyespy (`@eyespy/*` scope on npm)
- Full Meticulous parity (recording, replay, auto-test-generation, test pruning)
- "Good enough" determinism (Playwright anti-flake stack + pixel tolerance, NOT a Chromium fork)
- Record/replay focused (no AI exploration agent)
- Auto-generate tests from recorded sessions using coverage-based selection
- Both mocked network (from recordings) AND live backend modes
- **CLI-first for V1** — no server/DB/dashboard. Server + dashboard + GitHub integration in Phase 2.
- TypeScript + Playwright stack
- Both script tag recording AND Playwright recorder — **same session format, different capture methods**
- SDK posts to configurable endpoint; **always-on receiver** (`eyespy record --listen`) during recording
- **Baselines**: Manifest-only in git (`baselines.json` with SHA-256 hashes). Actual PNGs in local content-addressable blob store (gitignored). Optional remote sync (S3/R2) for team sharing.
- **Screenshots**: Tiered capture — always after navigations + end-of-session, with stabilization after significant interactions (clicks/submits), skip keystrokes/hovers/scrolls. Two-consecutive-identical-screenshots stabilization loop.
- Smart retry: 1x replay, re-confirm only diffs (not always-3x) — **default ON**
- Full dedup + retention for storage from V1
- **Phased delivery**: Phase 0 (PoC) → Phase 1a (MVP) → Phase 1b (completion) → Phase 2 (server)
- WS + SSE support in **Phase 1b** (not 1a MVP)
- Single-team for V1 (no multi-tenancy)
- GitHub OAuth + RBAC when server ships (Phase 2)
- **Clock**: Full `clock.install()` on Playwright >= 1.48. Date + setTimeout + setInterval all mocked. Custom `addInitScript` patches for `document.timeline.currentTime` and `MessageChannel` timing.
- **Network mock mode**: Mock recorded fetch/XHR requests (any origin), abort unrecorded fetch/XHR, allow document/script/stylesheet/image/font/media loads through to real server. ResourceType-based routing, not origin-based.
- **Auto-healing replay**: Basic fallback cascade in 1a, advanced healing overlay + confidence scoring in 1b
- **Response bodies**: Blob store has no size limit. Recorders have configurable capture limits (`maxBodyBytes`: SDK default 256KB, Playwright recorder default 1MB). Bodies larger than limit are truncated with `bodyTruncated: true` flag. Strip sensitive headers (Auth, Cookie, Set-Cookie). No body redaction (breaks replay).
- **Shared package**: Types + shared logic (selector matching, session validation, network normalization)
- **Recording UX**: Minimal floating overlay ("Stop Recording" button + event counter, z-index max). Ctrl+C in terminal also stops. Overlay excluded from screenshots and selector generation.
- Scope: Standard SPAs + common widgets. Out of scope for V1: cross-origin iframes, WebGL, PWAs/Service Workers

---

## Architecture (CLI-First)

```
Phase 1: CLI Tool (no server)

  Your App (dev/staging)
     │
     ├── Recording SDK (script tag) ──→ POST to configurable endpoint
     │   Records clicks + network         (or local file via CLI receiver)
     │
     └── OR: Playwright-based recording via CLI
              npx eyespy record --url http://localhost:3000

  Recorded Sessions (.eyespy/sessions/*.json)
     │
     ▼
  npx eyespy replay --url http://localhost:3000
     │
     ├── Playwright + anti-flake stack
     ├── Network mocking from recorded data
     ├── V8 coverage collection
     ├── Screenshot capture
     └── Smart retry (re-confirm diffs only)
     │
     ▼
  npx eyespy diff
     │
     ├── pixelmatch with tolerance
     ├── Content-addressable dedup (SHA-256)
     └── Three-panel HTML report
     │
     ▼
  npx eyespy generate
     │
     ├── Greedy set cover on coverage bitmaps
     ├── Dedup identical coverage sessions
     └── Outputs minimal test suite config

  Baselines: .eyespy/baselines.json (git-tracked manifest, SHA-256 hashes)
  Blobs:     .eyespy/blobs/        (gitignored, content-addressable PNGs + bodies)
  Sessions:  .eyespy/sessions/     (gitignored, recorded session data)
  Artifacts: .eyespy/runs/         (gitignored, replay run outputs)
```

```
Phase 2: Server + Dashboard (Docker)

  ┌──────────────────────────────────────────────────┐
  │                 SERVER (Docker)                    │
  │  ┌──────────┐  ┌────────────┐  ┌──────────────┐ │
  │  │ Fastify  │  │ Workers    │  │ Dashboard    │ │
  │  │ REST API │  │ (BullMQ)   │  │ (React SPA)  │ │
  │  └────┬─────┘  └─────┬──────┘  └──────────────┘ │
  │       └──────┬────────┘                           │
  │       ┌──────▼──────┐   ┌──────────────┐         │
  │       │ PostgreSQL  │   │ Blob Storage │         │
  │       └─────────────┘   └──────────────┘         │
  └──────────────────────────────────────────────────┘
         ▲              ▲
    ┌────┘        ┌─────┘
    │             │
  ┌─┴────┐  ┌────┴─────────┐
  │ CLI  │  │ GitHub       │
  │      │  │ Webhooks     │
  └──────┘  └──────────────┘
```

---

## Component Details

### 1. Recording SDK (Browser, < 18KB gzipped)

Lightweight JS injected via script tag. Records clickstream + network, NOT DOM mutations.

**Interaction events**: click, dblclick, input, change, scroll, keydown, keyup, navigate, resize, focus/blur, pointerdown/pointermove/pointerup, touchstart/touchmove/touchend, dragstart/dragover/drop/dragend, paste, copy, cut
- Capture modifier keys (metaKey, ctrlKey, shiftKey, altKey) on all keyboard events
- Record `inputType` on input events (for contentEditable/rich text editors)
- Detect and log browser autofill events

**Network capture**: Direct monkey-patch interception (OpenReplay/Sentry pattern) — replace `window.fetch` and `XMLHttpRequest.prototype.open/send` directly (NOT ES6 Proxy — Proxy adds overhead and breaks `instanceof`). Intercept `WebSocket` constructor via ES6 Proxy (correct for constructors) and `EventSource` constructor similarly.
- SDK MUST load before application bundles (first `<script>` in `<head>`)
- Runtime detection: warn when `window.fetch` is not the proxy (library cached original reference)
- Record full request (URL, method, headers, body) and response (status, headers, body)
- WebSocket: record frame data (send + receive) with timestamps
- SSE: record EventSource URL + received events
- `Response.clone()` to avoid consuming streams

**Known out-of-scope**: Service Workers bypass interception. Cross-origin iframes invisible. Subresource loads (`<img>`, `<link>`, `<script>`) not captured.

**Selector strategy**: `data-testid` > `id` > `role+aria-label` > structural CSS path. Store heuristic fingerprints (innerText snippet + boundingRect) alongside selector. Validate with `querySelector`. Skip virtualized list items (detect `data-index` or `style="transform:..."` patterns).

**Transport**: Uses `_originalFetch` (the saved pre-patch reference) for its own HTTP calls — MUST NOT use `window.fetch` which would be intercepted by the SDK's own monkey-patch, causing infinite recursion. Retry + exponential backoff. Reserve `sendBeacon` for `pagehide` event only (last-chance flush). Batch 50 events OR 5s. Max buffer 5MB — circuit breaker after N consecutive failures.

**Compression**: CompressionStream API (zero-cost) with fflate fallback for Safari < 16.4.

**Privacy & Safety**:
- Password fields (`type=password`) masked by default
- `data-no-record` attribute
- `Authorization`, `Cookie`, `Set-Cookie` headers stripped by default
- Configurable POST body deny-list for sensitive paths (`/login`, `/checkout`, etc.)
- **Production kill switch**: `allowedHosts` allowlist, default sampling 0% for unknown origins
- **Mutation throttle** (safety kill switch, NOT mutation recording): The SDK does NOT record DOM mutations. This is a MutationObserver-based safety detector that measures DOM mutation frequency. If > 10,000 mutations in a 10-second window (Sentry pattern), the SDK stops recording entirely and emits a terminal event `{ type: 'recording-stopped', reason: 'mutation-throttle' }`. Prevents runaway pages/extensions from consuming unbounded memory.

**Framework notes**:
- Angular/zone.js: Document initialization order. SDK must load before zone.js patches.
- Next.js SSR: Don't touch DOM before hydration completes. Wait for `__NEXT_DATA__`.
- React Strict Mode: Deduplicate double-fired effects in development.

### 2. Replay Engine (Playwright)

Replays recorded sessions in headless Chromium with deterministic environment.

**Determinism stack**:
- **Seeded PRNG injection** (table-stakes): `addInitScript()` that replaces `Math.random`, `crypto.randomUUID`, and `crypto.getRandomValues` with seeded deterministic versions. Without this, nothing works — every SPA generates client-side UUIDs.
- **Full Clock API** (`page.clock.install()`): Requires Playwright >= 1.48. Mocks `Date`, `setTimeout`, `setInterval`, `requestAnimationFrame`, `performance.now`, `requestIdleCallback`, `Intl.DateTimeFormat` (default date). Use `clock.pauseAt()` to freeze time for screenshots, `clock.runFor()` to advance timers deterministically between interactions, `clock.resume()` as escape hatch for edge cases. This gives full timer determinism — `setTimeout(() => X, 1000)` fires instantly when we advance the clock.
- **Clock gap patches** (via `addInitScript`): Playwright's clock does NOT mock `document.timeline.currentTime` (breaks Framer Motion, open issue #38951) or `MessageChannel` timing (used by React 18+ scheduler). We patch both:
  - `document.timeline`: Override `currentTime` getter to return mocked `performance.now()`
  - `MessageChannel`: Wrap `postMessage` to route through mocked `setTimeout(fn, 0)` for deterministic delivery order
  - `performance.mark()` / `performance.measure()`: Stubbed by clock (return no-ops). Document that Performance Observer / Navigation Timing APIs return empty results.
- **Playwright `animations: 'disabled'`**: Option on `page.screenshot()` (NOT browser/context level). Passed on every screenshot call: `page.screenshot({ animations: 'disabled' })`. Handles CSS animations + CSS transitions + WAAPI. Protocol-level, not CSS injection. Finite animations fast-forwarded to completion. Infinite animations cancelled. No conflict with clock mocking (different layers).
- Additional anti-flake CSS: `caret-color: transparent; scroll-behavior: auto !important`
- **Browser flags**: **Do NOT use `--deterministic-mode`** — it enables `--enable-begin-frame-control` which requires manual CDP `HeadlessExperimental.beginFrame` commands. Playwright's `page.screenshot()` does not issue these, causing it to hang indefinitely ([Playwright #31829](https://github.com/microsoft/playwright/issues/31829)). Playwright's maintainer: "If you touch `args`, you are on your own!" Instead, set the individual compositor flags that ARE safe with Playwright:
  - `--run-all-compositor-stages-before-draw` — forces full compositor pipeline before draw
  - `--disable-threaded-animation` — deterministic animation timing on compositor thread
  - `--disable-threaded-scrolling` — deterministic scroll on compositor thread
  - `--disable-checker-imaging` — no progressive image decode placeholders
  - `--disable-image-animation-resync` — prevents GIF/APNG resync to system clock
  - `--disable-new-content-rendering-timeout` — no timeout for new content rendering
  - **Explicitly DO NOT set** `--enable-begin-frame-control` — this is what causes the hang
  - Additionally: `--font-render-hinting=none`, `--disable-gpu`, `--force-color-profile=srgb`, `--hide-scrollbars`, `--disable-font-subpixel-positioning`, `--disable-lcd-text`, `--disable-skia-runtime-opts`, `--disable-partial-raster`
- Fixed viewport: 1280x720, deviceScaleFactor: 1
- **Headless mode**: Use `headless: true` (Playwright's default, which uses the new headless mode since Playwright 1.33). The old headless mode (`headless: 'shell'`) uses a separate headless shell binary with different rendering behavior. The new headless mode uses the same Chromium binary as headed mode — critical for screenshot consistency. Recording uses headed mode (`headless: false`) so the user can interact.
- **Fresh BrowserContext per session** (NOT a new browser process — reuse the same `Browser` instance for all sessions within a run, create a new `BrowserContext` for each): Each replay session starts with clean state — no localStorage, IndexedDB, cookies, or Service Workers from previous sessions. `browserContext` created with `storageState: undefined`, `serviceWorkers: 'block'`. Smart retry also creates fresh contexts (same browser).
- **Font loading gate**: Wait for `document.fonts.ready` AND `document.fonts.status === 'loaded'`. Font loading works correctly with paused clock because: `@font-face` triggers network requests → fulfilled by route handlers (Node.js, real time) → browser font renderer processes data (rendering pipeline, not JS timers) → `document.fonts.ready` resolves. The clock mock only affects JS-level timer APIs, not network completion or rendering.
- **Docker mandatory**: Mandate official Playwright Docker image for CI. Document that local replays may differ from CI.

**Network mocking**: Uses a **two-layer routing architecture** to separate egress enforcement from mock fulfillment:

**Layer 1 — Global egress policy** (`browserContext.route('**/*')`): A single catch-all route that ONLY enforces allow/deny decisions. It never fulfills or modifies responses (avoiding catch-all flakiness #22338). If a request's origin is not in the computed `allowedOrigins` set, abort with `blockedbyclient`. Otherwise `route.fallback()` (passes to Layer 2 or the network). This ensures ALL resource types (fetch, xhr, document, image, etc.) are subject to egress control.

**Layer 2 — Mock fulfillment** (per-origin `browserContext.route()`): Narrow origin-correct routes (e.g., `browserContext.route('https://api.stripe.com/**')`) that handle mock mode logic. These routes check `request.resourceType()` — only `fetch`/`xhr` are mocked from recorded FIFO queues; other resource types call `route.continue()`. This is where ResourceType-based routing lives.

- **Mock mode**: Layer 2 fulfills recorded fetch/XHR responses when matched, aborts unrecorded fetch/XHR (`route.abort('connectionrefused')`), and continues other resource types.
- **Live mode**: Layer 2 is not registered. Layer 1 still enforces egress policy. By default, live mode denies requests to origins not in `replay.allowedOrigins` (prevents malicious PRs from exfiltrating CI secrets or scanning internal networks). Set `replay.allowLiveExternalEgress: true` to explicitly allow unrestricted egress.
- **Origin remapping** (handles recording at `localhost:3000`, replaying at `ci:3000`): Config `replay.originMap` maps recorded origins to replay origins. Applied uniformly to: (a) route interception patterns, (b) FIFO queue keys, (c) session event URL matching. **Auto-infer**: if `originMap` is empty, the replay engine computes `{ recordedSessionOrigin → targetUrl origin }` automatically. For multi-origin apps (e.g., separate API server), provide explicit mappings: `{ "http://localhost:4000": "http://ci-api:4000" }`. All URL normalization passes through `remapUrl()` which applies the origin map before any matching.
- **Allowed origins** (SSRF protection in CI): Config `replay.allowedOrigins` restricts which origins **any request** (fetch, xhr, document, image, font, etc.) can reach. Enforced by the Layer 1 global route. Non-allowed origins aborted with `route.abort('blockedbyclient')`.
  - **Playwright-recorded sessions**: The recorder auto-discovers all origins observed during recording and persists them in `session.observedOrigins[]`. At replay time, the effective allowlist is: `[targetUrl origin]` + `originMap` values + `session.observedOrigins` + explicit `replay.allowedOrigins`. This handles CDN fonts/scripts/images automatically.
  - **SDK-recorded sessions**: The SDK only captures fetch/XHR, not subresource loads (images, fonts, scripts). Allowlist defaults to `[targetUrl origin]` + `originMap` values + explicit `replay.allowedOrigins`. Teams with CDN-heavy apps must manually add CDN origins to `replay.allowedOrigins`.
- **Two-tier FIFO matching**: For GET/DELETE/HEAD: FIFO per `METHOD:normalizedURL`. For POST/PUT/PATCH: first try body-fingerprinted queue (`METHOD:URL:bodyFingerprint`), fall back to URL-only FIFO. Body fingerprint is a truncated SHA-256 of the request body content (from BodyRef inline data or blob). This handles concurrent requests to the same endpoint (GraphQL, batch APIs) where arrival order may differ between recording and replay.
- **WebSocket/SSE (Phase 1b)**: WS mocking via `page.routeWebSocket()` (Playwright 1.48+), SSE mocking via `addInitScript()` EventSource replacement. **Phase 1a behavior**: WS and SSE connections are blocked (`route.abort()` for WS upgrade requests, EventSource constructor replaced with no-op) and logged as `W_WS_BLOCKED` / `W_SSE_BLOCKED`. This prevents nondeterminism from unmocked real-time connections.
- **Per-origin policy overrides**: Config `replay.originPolicies` allows per-origin control: `{ "https://api.example.com": "mock", "https://cdn.example.com": "live", "https://untrusted.example.com": "blocked" }`. When set, overrides the global `mode` for that origin. This enables mixed mock/live configurations — e.g., mock API calls while passing CDN through live, or explicitly blocking untrusted origins. Origins not in the map use the global `replay.mode`.
- **CORS preflights**: If OPTIONS requests are intercepted by Playwright, auto-fulfill with `204` and dynamic `Access-Control-Allow-*` headers based on the request. If the Playwright version does not surface preflights, fail fast with `E_CORS_PREFLIGHT_UNSUPPORTED` when a preflight-dependent request fails in mock mode, and recommend switching that origin to live mode via `originPolicies`.
- Never use `waitForLoadState('networkidle')` — hangs on persistent connections.
- Use `browserContext.route()` (survives full-page navigations) not `page.route()`.

**Auto-healing replay** (selector resilience):
When a CSS selector fails during replay, the engine tries a fallback cascade before giving up:
1. Retry primary selector with 1.5s timeout (handles timing, not structural breakage)
2. `data-testid` attribute match (confidence: 0.95)
3. ARIA `role` + `aria-label` combination (confidence: 0.90)
4. `role` + visible text content (confidence: 0.85)
5. Text content + tag name, strict single-match (confidence: 0.75)
6. Stable attributes (`name`, `type`, `placeholder`, `href`) fingerprint (confidence: 0.70)
7. Parent context + text (confidence: 0.65)
8. Fuzzy structural path walk (confidence: 0.50)
9. Visual proximity (bounding box + text similarity, confidence: 0.40 — floor)

**Phase 1a**: Strategies 1-5 (basic healing) + skip-and-warn if all fail. If >30% of interactions fail, abort session as unreplayable.
**Phase 1b**: Full cascade + healing overlay (don't mutate recordings), confidence scoring, interaction dependency graph for cascade-skipping, explicit "promote healing" action, auto-promotion policy (>= 0.90 confidence across 3 consecutive runs).

**Network mock miss handling**: When a fetch/XHR request during replay doesn't match any recorded response, behavior depends on `replay.unmatchedFetchXhrPolicy`:
- `"error"` (recommended for CI): abort the request, emit `E_ROUTE_MISS_MOCK`, mark session as `error` (exit code 2). Strict — any unrecorded API call is treated as a test failure.
- `"warn"` (default): abort the request (`route.abort('connectionrefused')`), emit `W_ROUTE_MISS_MOCK`, continue replaying. Session can still pass/diff.
- `"passThrough"`: allow the request to hit the real network (subject to egress allowlist), emit `W_ROUTE_MISS_PASSTHROUGH`. Useful during migration from live to mock mode.
- In live mode (regardless of policy): pass through to real backend.
- After replay: report all mock misses in the run summary
- **Fuzzy URL matching** (Phase 1b): Ignore configurable query params (e.g., `_t`, `timestamp`, `nonce`) when matching FIFO queues. Match on path pattern when exact URL fails.

**Navigation timeout recovery**: When a navigation exceeds 30s timeout:
- Capture a diagnostic screenshot of the current page state
- Log all pending/in-flight network requests
- Mark the session as "replay-error" (exit code 2, not exit code 1)
- Skip remaining interactions in this session, continue to next session
- Do NOT attempt to screenshot or diff a half-loaded page

**Screenshot timing** (tiered strategy):
- **Always capture**: After navigations (URL changes, route transitions), after end of session (final state)
- **Capture with stabilization**: After clicks/submits that trigger significant DOM changes. Detection heuristic: MutationObserver within 500ms of the event must see **structural mutations** (childList additions/removals of non-trivial nodes, i.e., elements with children or text content > 10 chars) OR **navigation/route changes**. Attribute-only mutations (class/style changes from focus, hover, `:active`) are excluded — these are cosmetic, not state changes. Form submissions (`<form>` submit event) always trigger capture regardless of DOM mutations.
- **Skip**: Individual keystrokes mid-input (wait for blur/submit), mouse moves, hovers, scroll events (unless they trigger lazy loading)
- **Stabilization loop** (Playwright/Chromatic pattern): After dispatching event and waiting for DOM settle, take screenshot A. Wait 100ms. Take screenshot B. If A === B (pixel-identical), use B. If different, repeat (max 5 attempts or 3s timeout). This absorbs remaining compositor/rendering timing variance.
- **Clock lifecycle invariant (normative)**: The clock has exactly two states during replay:
  1. **Running** — during `clock.runFor(delta)` event advancement AND during DOM settle detection (Node-orchestrated polling uses real wall time; browser timers fire normally).
  2. **Paused** — only immediately around screenshot capture: `clock.pauseAt()` before stabilization loop, take screenshot(s), then `clock.resume()` before the next interaction.
  This invariant prevents hangs from browser-timer-freeze during settle while ensuring frozen time for screenshots.
- Configurable via `screenshotStrategy` in config: `"tiered"` (default), `"every"` (every event), `"manual"` (only explicit `screenshot-marker` events)

**Screenshot key stability** (prevents baseline churn from skipped interactions):
Screenshot keys are derived from **event `seq`** (the event's monotonic sequence number), NOT sequential capture count or array index. This means if an interaction is skipped by auto-healing, later screenshot keys don't shift. `seq` is canonical — it never changes even if events are re-ordered or gaps exist.

Key format: `{type}@e{seq}` where:
- `nav@e{seq}` — post-navigation capture (after a `navigate` marker with that `seq`)
- `cap@e{seq}` — capture after an interaction event (click/submit/etc.) with that `seq`
- `final` — end-of-session capture (always last)

Example: If events are `[navigate(seq=0), click(seq=1), input(seq=2), click(seq=3), navigate(seq=4)]` and the tiered strategy captures after navigate(0), click(1), click(3), navigate(4), and end-of-session, the keys are: `nav@e0`, `cap@e1`, `cap@e3`, `nav@e4`, `final`. If click(1) is skipped by auto-healing, the remaining keys (`cap@e3`, `nav@e4`, `final`) are unchanged — no renumbering.

If a capture point fails (page crash, timeout), the key still exists in the run output as `{ status: "error", error: "..." }` — later keys do not shift.

**Smart retry**: Replay once. Determine diff candidates using the **same pixel threshold** as final pass/fail (not hash comparison alone — a hash can differ while the pixel diff is under threshold due to compression/metadata variance). For each candidate, replay 2 more times to confirm. Majority vote (2-of-3 must exceed pixel threshold for it to count as a confirmed diff). Only pays 3x cost for actual diffs (~5-20% of screenshots). **Default ON, Phase 1a.** Coverage collection is OFF during retry attempts — we already have the coverage bitmap from the first replay. Retry attempts exist solely to confirm/reject visual diffs.

**Fresh context per retry** (critical implementation detail): FIFO network queues are consumed by `.shift()` during replay — after the first replay, all queues are empty. Each retry attempt MUST:
1. Create a fresh `BrowserContext` (no shared state from previous attempt)
2. Rebuild FIFO queues from the original session data (`buildNetworkQueues(session)`)
3. Re-register all routes on the new context
4. Re-inject PRNG seed and clock (deterministic — same seed produces same sequence)

This means the retry function signature is: `replaySession(session, seed) → screenshots[]`. The orchestrator calls it up to 3 times for sessions with diffs, each call is fully independent. The orchestrator only compares specific screenshots that diffed, not the entire session.

**Coverage collection**: `page.coverage.startJSCoverage({ resetOnNavigation: false })`. Extract V8 block coverage bitmap (note: V8 "block coverage" ranges approximate but are not identical to source-level branches — they represent executed byte ranges within functions. We call these "coverage bitmaps" throughout; user-facing "branch coverage %" is an approximation). Version-stamp with `sourceContentHash` — SHA-256 of the concatenated `entry.source` strings from coverage output. This hashes what V8 actually executed, not build artifacts. Modern bundlers (webpack, vite, esbuild) are NOT deterministic between builds — same source can produce different output due to chunk hashing, timestamps, parallel processing artifacts. Hashing V8's script source at runtime eliminates this problem entirely. Only invalidate bitmaps when the source hash changes. `resetOnNavigation: false` accumulates coverage across SPA navigations within a session.

**Coverage limitation (mocked network)**: V8 coverage tracks which branches execute. With network mocking, responses come from FIFO queues, not real servers. If the app has different code paths for different response shapes/timings/errors, coverage won't capture all production paths. This is inherent — coverage reflects the recorded session's behavior, not all possible behaviors. Document this to users.

**DOM settle detection**: **Node.js orchestrated polling** — NOT a single `page.evaluate()` with timers (which would hang when clock is paused). Install a MutationObserver that sets a boolean flag on DOM changes. Poll from Node.js every 50ms using `page.waitForTimeout()` (real wall-clock time), checking the flag via instant `page.evaluate()` calls. After 300ms of no DOM changes, check `document.fonts.ready` and `img.complete` on visible images. All timer-based waits happen in Node.js, not in the browser where they'd be frozen by `clock.pauseAt()`.

**Why `page.waitForTimeout()` is clock-safe**: `page.waitForTimeout(ms)` creates a Node.js-side `setTimeout` on the Playwright controller process. Playwright's `clock.install()` only mocks browser-side timer APIs via CDP. Node.js timers are unaffected — `page.waitForTimeout(50)` always waits 50ms of real wall-clock time regardless of clock state. Similarly, `page.evaluate()` with synchronous code (read a boolean, return it) completes instantly even when clock is paused — it only hangs if the evaluated code itself calls `setTimeout`/`requestAnimationFrame`.

**Error handling during replay** (not just navigation timeout):
- `page.evaluate()` throws: Catch, log the error + event context, skip interaction (count as failed). Usually means the page crashed or a script error occurred.
- `route.fulfill()` fails: Catch, log the unmatched request, the route handler continues. Next matching request may also fail — if > 5 consecutive route errors, log diagnostic and switch to `route.abort()` for remaining requests.
- Browser crash (Chromium OOM, GPU process crash): `page.screenshot()` will throw. Catch at the session level, mark as `replay-error` (exit code 2), continue to next session.
- `locator.click()` / `.fill()` timeout (default 30s): Treated as interaction failure, count toward the 30% abort threshold.
- Unexpected popup/dialog: Register `page.on('dialog')` handler that auto-dismisses with `dialog.dismiss()`. Log the dialog message. Don't let a `window.confirm()` block replay.

**Resource constraints**: Chrome uses 3-20GB RAM per instance. Default concurrency: `Math.min(2, Math.floor(os.totalmem() / 5GB))`. Hard timeout: 120s per session replay, 30s per navigation.

### 3. Test Generator (Coverage-Based Selection)

**Algorithm**:
1. For each recorded session: replay with V8 coverage ON, extract V8 block coverage bitmap
2. Version-stamp bitmap with `(gitSha, sourceContentHash)`
3. Deduplicate: sessions with identical bitmaps → keep one (same flow, different data)
4. Greedy set cover: iteratively select session covering most uncovered branches
5. Stop when target coverage reached (default 100% of reachable branches)
6. Result: minimal set of K sessions (K << N total) that maximize coverage

**Staleness management**:
- Bitmaps stamped with `(gitSha, sourceContentHash)`. **Staleness is determined by `sourceContentHash`**, NOT git SHA — git SHA changes every commit even when the actual JavaScript bundle is identical (comment changes, test-only changes, README updates). `sourceContentHash` only changes when the code V8 actually executes changes.
- On replay against new code: if `sourceContentHash` differs, re-collect coverage and update bitmap (self-healing). If same hash, bitmap is still valid even with a different git SHA.
- If >50% of sessions are stale and unreplayable, fall back to "replay all"
- Coverage-guided session skipping (TurboSnap equivalent): only replay sessions whose bitmaps overlap with files changed in the current diff. 80%+ time savings.

**File storage** (CLI-first):
- `.eyespy/coverage/` — bitmap files per session, keyed by session ID
- `.eyespy/suites/` — generated suite config (list of session IDs + coverage stats)

### 4. Visual Diff Engine

- pixelmatch with configurable tolerance (`threshold`, `maxDiffPixels`, `maxDiffPixelRatio`)
- Anti-aliasing detection ON by default (`ignoreAntialiasing: true` in config — AA pixels excluded from diff counts)
- **Dimension check**: If baseline and current screenshots differ in dimensions, report as `E_DIMENSION_MISMATCH` (exit code 2, NOT exit code 1 — this is a tooling/config error, not a visual diff). Likely means viewport changed or scrollbar appeared. Don't attempt resize — the mismatch itself is the signal.
- Region masking: rectangles or CSS selectors for dynamic content (timestamps, ads, avatars) **(Phase 1b)**
- Content-hash short-circuit: SHA-256 hash screenshot before diffing. If hash matches baseline, skip pixelmatch entirely.
- Three-panel output: expected | actual | diff (red pixels)
- **HTML report**: Standalone HTML file with base64-inlined images for <= 30 screenshots. **Folder-based report** (images as separate files + index.html) for > 30 screenshots. Four viewer modes: side-by-side, slider (CSS `clip-path`), blend (opacity crossfade), toggle (click to switch). **All session-derived strings (URLs, session IDs, screenshot keys, error messages) MUST be HTML-escaped before rendering** — report templates must never inline unsanitized content to prevent XSS when reports are opened in browsers.

### 5. CLI (`@eyespy/cli`)

```bash
# Setup
eyespy init                                       # Create .eyespy/ dir, config.json, update .gitignore

# Recording
eyespy record --url http://localhost:3000        # Playwright-based recording
eyespy record --listen 8479                      # Start receiver for SDK recordings (Phase 1b)

# Replay + Diff
eyespy replay --url http://localhost:3000        # Replay all sessions, capture screenshots
eyespy diff                                       # Compare against baselines
eyespy approve                                    # Accept current screenshots as new baselines
eyespy approve --session <id>                     # Approve specific session

# Test Generation (Phase 1b)
eyespy generate                                   # Auto-generate minimal test suite from sessions
eyespy generate --target-coverage 80              # Target 80% branch coverage

# CI (all-in-one)
eyespy ci --url http://localhost:3000             # Replay + diff + exit code
eyespy ci --url http://localhost:3000 --confirm   # Smart retry for diffs

# Baselines
eyespy push-baselines                             # Upload baseline blobs to remote storage
eyespy pull-baselines                             # Download baseline blobs from remote storage

# Utilities
eyespy status                                     # Suite health, coverage stats, staleness
eyespy prune                                      # Remove stale sessions and expired artifacts
eyespy prune --older-than 30d                     # Custom retention
eyespy sanitize                                   # Strip response bodies for sharing sessions (Phase 1b)
```

**`eyespy init`**: Creates `.eyespy/` directory with default `config.json` and empty `baselines.json` (both git-tracked). Adds `.eyespy/sessions/`, `.eyespy/blobs/`, `.eyespy/runs/`, `.eyespy/coverage/` to `.gitignore`. Auto-runs on first `eyespy record` if not initialized.

**`eyespy approve` workflow**:
1. Read the latest run's screenshots from `.eyespy/runs/latest/screenshots/`
2. For each screenshot: compute SHA-256 hash, store in blob store (if not already present)
3. Update `baselines.json`: set each `session-id/screenshot-name → { hash, width, height }`
4. Write updated `baselines.json` to disk (git-tracked — user commits it)
5. With `--session <id>`: approve only that session's screenshots (partial approval)
6. Print summary: `"Approved N screenshots for M sessions. Run 'git add .eyespy/baselines.json && git commit' to persist."`
7. **Guard**: By default, refuse to approve on non-Docker environments (screenshots may differ from CI). Override with `--force`. Check via `process.env.container === 'docker'` or presence of `/.dockerenv`.

JSON output when piped (`| jq`), human-friendly TTY output otherwise. Exit code 0 = all pass, 1 = diffs detected, 2 = replay failures. **Mixed outcome precedence**: error (2) > diff (1) > pass (0). If any session has a replay error AND another session has a visual diff, the overall exit code is 2.

**Machine-readable summary** (`eyespy ci` always writes `.eyespy/runs/latest/summary.json`):
```json
{
  "version": 1,
  "runId": "abc123",
  "timestamp": "2025-01-15T10:30:00Z",
  "playwrightVersion": "1.50.0",
  "exitCode": 1,
  "sessions": [
    { "id": "session-abc", "status": "pass", "screenshots": 5, "diffCount": 0, "durationMs": 3200, "errors": [], "warnings": [] },
    { "id": "session-def", "status": "diff", "screenshots": 4, "diffCount": 2, "durationMs": 4100, "confirmedDiffs": ["cap@e5", "final"], "errors": [], "warnings": [{ "code": "W_BODY_TRUNCATED", "message": "2 responses truncated at maxBodyBytes", "count": 2 }] },
    { "id": "session-ghi", "status": "error", "durationMs": 30000, "errors": [{ "code": "E_NAV_TIMEOUT", "message": "Navigation timeout after 30000ms", "details": { "url": "http://localhost:3000/checkout" } }], "warnings": [] }
  ],
  "totals": { "sessions": 3, "passed": 1, "diffs": 1, "errors": 1, "screenshots": 9, "diffScreenshots": 2, "durationMs": 37300 }
}
```
Used by CI integrations (PR comments, Slack notifications, dashboards) without parsing CLI text output.

**Logging contract**:
- **stdout**: Reserved for JSON output when `--json` or piped (`!process.stdout.isTTY`). Human-formatted output otherwise.
- **stderr**: All diagnostic logs, progress bars (cli-progress), spinners (ora). Suppressed when `--json` or piped.
- **Correlation IDs**: Every log line includes `runId`. Per-session logs include `sessionId`. Network events include `requestId`.
- **`--verbose`**: Enables debug-level logs — route matching decisions, healing attempts + confidence scores, stabilization loop iteration count, clock lifecycle transitions, FIFO queue hits/misses.
- **Structured logging**: Internal log format is `{ level, timestamp, runId, sessionId?, message, data? }`. Written as formatted text to stderr in TTY mode.

### 6. Storage (Manifest + Content-Addressable Blobs)

```
.eyespy/
├── config.json                    # Project config — git-tracked
├── baselines.json                 # Baseline manifest — git-tracked (small JSON, ~5KB)
│                                  # Maps session-id/screenshot-name → SHA-256 hash
├── suites/                        # Git-tracked — generated test suite configs
│   └── default.json
├── sessions/                      # Gitignored — recorded session data
│   └── <session-id>.json          # Interactions + network (bodies stored separately)
├── blobs/                         # Gitignored — content-addressable store
│   └── <sha256-prefix>/<sha256>   # Baseline PNGs, network bodies, screenshots, diffs
├── coverage/                      # Gitignored — coverage bitmaps
│   └── <session-id>.bitmap
└── runs/                          # Gitignored — replay run artifacts
    └── <run-id>/
        ├── screenshots/           # References into blobs/
        ├── diffs/
        └── report.html
```

**Key design**: Only `config.json`, `baselines.json`, and `suites/` are git-tracked. All binary artifacts (PNGs, network bodies) live in `blobs/` (gitignored). This prevents git repo bloat — the manifest file changes by a few bytes per approval, not megabytes of PNGs.

**Canonical digest format**: `sha256:<lowercase-hex-64>` in all manifests, APIs, and summary.json. BlobStore filesystem paths use raw hex only (no `sha256:` prefix): `.eyespy/blobs/<hex[0..1]>/<hex[2..3]>/<hex>`. `BlobStore.put()` returns raw hex; callers add `sha256:` prefix when writing to manifests. On read, strip `sha256:` prefix to derive filesystem path. This separation keeps paths filesystem-safe while manifests are future-proof for algorithm changes.

**Baseline manifest format**:
```json
{
  "version": 1,
  "renderer": {
    "playwrightVersion": "1.50.0",
    "chromiumVersion": "133.0.6943.16",
    "dockerImage": "mcr.microsoft.com/playwright:v1.50.0-noble",
    "viewport": { "width": 1280, "height": 720, "deviceScaleFactor": 1 },
    "chromiumArgsHash": "sha256:a1b2c3d4...",
    "screenshotOptions": { "type": "png", "animations": "disabled" }
  },
  "baselines": {
    "session-abc123": {
      "nav@e0": { "digest": "sha256:e3b0c44298fc1c14...", "width": 1280, "height": 720 },
      "cap@e5": { "digest": "sha256:a1f2b3c4d5e6f7a8...", "width": 1280, "height": 720 },
      "final": { "digest": "sha256:d4e5f6a7b8c9d0e1...", "width": 1280, "height": 720 }
    }
  }
}
```

**Team sharing**: By default, blobs are local. For teams, configure a remote storage backend:
```bash
eyespy push-baselines              # Upload baseline blobs to remote
eyespy pull-baselines              # Download baseline blobs from remote
```
Pluggable storage interface: `StorageBackend { upload(hash, data), download(hash), has(hash) }`. Built-in: local filesystem (default), S3-compatible (AWS S3, Cloudflare R2, MinIO, Backblaze B2). Designed following reg-suit's plugin architecture. **Blob integrity**: On download from remote storage, always hash the received bytes and verify the digest matches the requested key before writing to the local blob store. Reject and retry (up to 2x) on mismatch.

**CI cold start (missing blobs)**: When `baselines.json` exists (git-tracked) but referenced blob files are missing (new developer clone, fresh CI), auto-detect and handle gracefully:
1. On `eyespy ci` / `eyespy diff`: Check if all hashes in `baselines.json` exist in local blob store
2. **If remote storage is configured**: Auto-pull missing blobs before diffing (`eyespy ci` implicitly runs `pull-baselines` for any missing hashes). Print `"Pulling N missing baseline blobs from remote..."`. This is the expected CI flow — no manual step needed.
3. If no remote configured and blobs missing: behavior depends on `replay.missingBaselinePolicy`:
   - `"warn"` (default for local dev): Print warning `"⚠ N baseline blobs missing."`, treat as "new", exit 0.
   - `"fail"`: Exit 2 (`E_BASELINE_MISSING`) — prevents CI from silently passing with missing baselines.
   - Recommended CI config: `"replay": { "missingBaselinePolicy": "fail" }`.
4. **Renderer mismatch detection**: On `eyespy ci` / `eyespy diff`, compare the current Playwright version against `baselines.json`'s `renderer.playwrightVersion`. If they differ, behavior depends on `replay.rendererMismatchPolicy`:
   - `"warn"` (default): Emit `W_RENDERER_MISMATCH` warning, continue diffing. Diffs are likely spurious but still reported.
   - `"fail"`: Exit 2 — forces explicit baseline approval before CI passes with a new renderer.
   - Recommended CI config: `"replay": { "rendererMismatchPolicy": "fail" }` (prevents false diffs from masking real regressions).
5. First `eyespy approve` after a fresh clone populates the blob store from current screenshots

**Dedup**: All binary artifacts stored by SHA-256 hash. Same screenshot across runs stored once. **Expected savings: 60-80%.**

**Retention**: `eyespy prune` command. Default: sessions older than 30 days, run artifacts older than 7 days. Baselines kept as long as referenced in `baselines.json`.

### 7. Config Schema

```json
{
  "targetUrl": "http://localhost:3000",
  "diff": {
    "threshold": 0.1,
    "maxDiffPixelRatio": 0.005,
    "ignoreAntialiasing": true
  },
  "replay": {
    "screenshotStrategy": "tiered",
    "quiescenceMs": 300,
    "sessionTimeoutMs": 120000,
    "navigationTimeoutMs": 30000,
    "concurrency": "auto",
    "smartRetry": true,
    "mode": "mock",
    "viewport": { "width": 1280, "height": 720 },
    "seed": "default",
    "localDiffThreshold": 0.02,
    "originMap": {},
    "allowedOrigins": [],
    "originPolicies": {},
    "allowLiveExternalEgress": false,
    "missingBaselinePolicy": "warn",
    "rendererMismatchPolicy": "warn",
    "unmatchedFetchXhrPolicy": "warn"
  },
  "storage": {
    "backend": "local",
    "remote": {
      "type": "s3",
      "bucket": "",
      "region": "",
      "prefix": "eyespy-baselines/"
    }
  },
  "recording": {
    "receiverPort": 8479,
    "maxBodyBytes": 262144,
    "bodyCapture": "truncate",
    "excludePaths": [],
    "sensitiveHeaders": ["Authorization", "Cookie", "Set-Cookie"]
  },
  "report": {
    "format": "html",
    "inlineThreshold": 30
  }
}
```

### 8. Receiver API (Phase 1b: `eyespy record --listen`)

Local HTTP server for ingesting SDK recordings. **Not a production service** — runs during recording sessions only.

**Safe defaults**:
- Binds to `127.0.0.1` (localhost only) unless `--host 0.0.0.0` explicitly provided
- Requires `X-Eyespy-Token` header unless `--no-auth` explicitly provided (discouraged)
- Token: auto-generated on startup and printed to terminal, or provided via `--token` / `EYESPY_TOKEN` env var
- CORS behavior:
  - If `--cors-origin` provided: allow exactly that origin.
  - If not provided and `.eyespy/config.json` exists: default to allowing the `targetUrl` origin (port-sensitive). This makes the SDK recording happy path work out-of-box.
  - Otherwise: CORS disabled (explicit opt-in required).
- MUST implement `OPTIONS` preflight for SDK use:
  - `Access-Control-Allow-Methods: GET, POST, OPTIONS`
  - `Access-Control-Allow-Headers: Content-Type, Content-Encoding, Idempotency-Key, X-Eyespy-Token`
  - `Access-Control-Max-Age: 600`
- Rate limit: 100 req/s per session (prevent runaway SDK)

**Endpoints**:

`GET /health` — Health check
- Response 200: `{ "ok": true, "version": "0.1.0" }`

`POST /v1/sessions` — Create session container
- Request: `{ "sessionId": "uuid", "startedAt": "ISO-8601", "url": "string", "viewport": { "width": 1280, "height": 720 }, "userAgent": "string", "sdk": { "version": "string" } }`
- Response 201: `{ "sessionId": "uuid" }`
- Errors: `400 E_SCHEMA`, `409 E_SESSION_EXISTS`

`POST /v1/sessions/{sessionId}/events` — Ingest event batch (at-least-once delivery)
- Headers: `Content-Type: application/json`, optional `Content-Encoding: gzip`, **`Idempotency-Key: <uuid>` (required)** — receiver deduplicates by storing last 100 keys per session for 24h. Duplicate key → return same 202 without re-appending.
- Request: `{ "events": [RecordedEvent...], "sequence": { "start": N, "end": M } }` — within a batch: `seq` must be contiguous (`events[i].seq = sequence.start + i`) and `t_ms` non-decreasing; receiver rejects otherwise (`400 E_EVENTS_OUT_OF_ORDER`). Cross-batch `seq` gaps are allowed during ingestion; `finish` validates global contiguity (`0..maxSeq` with no gaps) and rejects with `409 E_SESSION_INCOMPLETE` if gaps remain.
- Response 202: `{ "accepted": true, "eventCount": N }`
- Receiver: validates schema, appends to `.eyespy/sessions/.incoming/{sessionId}/`, extracts large bodies to blob store
- Errors: `400 E_SCHEMA`, `400 E_EVENTS_OUT_OF_ORDER`, `413 E_BATCH_TOO_LARGE` (> 5MB), `429 E_RATE_LIMIT`

`POST /v1/sessions/{sessionId}/finish` — Finalize session
- Request: `{ "endedAt": "ISO-8601" }`
- Response 200: `{ "sessionId": "uuid", "path": ".eyespy/sessions/{id}.json" }`
- Errors: `404 E_SESSION_NOT_FOUND`, `409 E_SESSION_NOT_OPEN`, `409 E_SESSION_INCOMPLETE`

**Error response format** (all non-2xx):
```json
{ "error": { "code": "E_SCHEMA", "message": "Human-readable description", "details": {} } }
```

### 9. Error Code Registry

Machine-readable error/warning codes used in `summary.json`, receiver responses, and CLI output.

**Errors** (cause exit code 2):
| Code | Context | Description |
|------|---------|-------------|
| `E_CFG_PARSE` | CLI | Config file is not valid JSON |
| `E_CFG_SCHEMA` | CLI | Config file fails schema validation |
| `E_SESSION_SCHEMA` | Replay | Session file fails schema validation |
| `E_SESSION_VERSION` | Replay | Session formatVersion is unsupported |
| `E_BLOB_MISSING` | Diff | Referenced blob hash not found in store or remote |
| `E_NAV_TIMEOUT` | Replay | Navigation exceeded timeout (default 30s) |
| `E_SESSION_TIMEOUT` | Replay | Session replay exceeded timeout (default 120s) |
| `E_PW_CRASH` | Replay | Browser/page crashed during replay |
| `E_DIMENSION_MISMATCH` | Diff | Screenshot dimensions differ from baseline |
| `E_ROUTE_MISS_MOCK` | Replay | Unrecorded fetch/XHR in mock mode when `unmatchedFetchXhrPolicy=error` |
| `E_SESSION_UNREPLAYABLE` | Replay | >30% of interactions failed auto-healing |
| `E_SCHEMA` | Receiver | Invalid request body |
| `E_EVENTS_OUT_OF_ORDER` | Receiver | Event batch has non-contiguous `seq` values within batch or `t_ms` not non-decreasing |
| `E_SESSION_INCOMPLETE` | Receiver | `/finish` validation: merged event stream has `seq` gaps (not contiguous `0..maxSeq`) |
| `E_SESSION_EXISTS` | Receiver | Session ID already exists |
| `E_SESSION_NOT_FOUND` | Receiver | Session ID not found |
| `E_BATCH_TOO_LARGE` | Receiver | Event batch exceeds 5MB |
| `E_RATE_LIMIT` | Receiver | Too many requests |
| `E_BASELINE_MISSING` | Diff | Baseline blob missing and `missingBaselinePolicy` is `"fail"` |
| `E_RENDERER_MISMATCH` | Diff | Playwright version differs from baseline renderer and `rendererMismatchPolicy` is `"fail"` |
| `E_CORS_PREFLIGHT_UNSUPPORTED` | Replay | CORS preflight request not interceptable; recommend switching origin to live mode |

**Warnings** (logged, don't affect exit code):
| Code | Context | Description |
|------|---------|-------------|
| `W_BODY_TRUNCATED` | Replay | Response body was truncated during recording; FIFO matching may degrade |
| `W_HEALING_USED` | Replay | Selector healed via fallback strategy (includes confidence) |
| `W_ROUTE_MISS_MOCK` | Replay | Unrecorded fetch/XHR in mock mode when `unmatchedFetchXhrPolicy=warn` |
| `W_ROUTE_MISS_PASSTHROUGH` | Replay | Unrecorded fetch/XHR passed through when `unmatchedFetchXhrPolicy=passThrough` |
| `W_ROUTE_MISS_LIVE` | Replay | Unrecorded request passed through in live mode |
| `W_BASELINE_MISSING` | Diff | No baseline exists for this screenshot (treated as "new") |
| `W_RENDERER_MISMATCH` | Diff | Playwright version differs from baseline renderer |
| `W_STALE_COVERAGE` | Generate | Coverage bitmap sourceContentHash doesn't match current build |
| `W_WS_BLOCKED` | Replay | WebSocket connection blocked in Phase 1a (not yet supported) |
| `W_SSE_BLOCKED` | Replay | EventSource connection blocked in Phase 1a (not yet supported) |

---

## Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Recording SDK | **TypeScript → ES2020 via esbuild** | IIFE bundle, < 18KB gz. ES2020 covers Chrome 80+/Firefox 80+/Safari 14+. ES5 adds 15-30% bloat and Proxy can't be polyfilled anyway. |
| Replay engine | **Playwright >= 1.48** | Clock API (with #31924 fix), routeWebSocket, animations:disabled |
| CLI | **Commander.js** | Lightweight, standard |
| Image diff | **pixelmatch + sharp** | Industry standard |
| Coverage | **Playwright `page.coverage` API** (V8 Profiler) | V8 block coverage (byte-range precision, approximates branch coverage), Chromium-only (Firefox/WebKit don't support JS coverage) |
| Monorepo | **pnpm workspaces + Turborepo** | Fast builds |
| Lint/Format | **Biome** | Single tool |
| Test runner | **Vitest** | Fast, Vite-native |
| Compression | **CompressionStream / fflate** | Zero-cost with fallback |

Phase 2 additions: Fastify, Drizzle, PostgreSQL 16, BullMQ, Redis, React 19, Vite

---

## Package Structure

```
eyespy/
├── packages/
│   ├── recorder-sdk/         # Browser recording SDK (< 18KB gzipped)
│   │   └── src/
│   │       ├── interactions/  # click, input, scroll, navigation, touch, drag, clipboard
│   │       ├── network/       # fetch/XHR proxy, WebSocket, EventSource interceptors
│   │       ├── selectors/     # multi-strategy selector generator + fingerprints
│   │       ├── transport/     # batching + fetch + beacon + retry + circuit breaker
│   │       ├── privacy/       # masking, redaction, allowlist, kill switch
│   │       └── session/       # lifecycle, metadata, mutation throttle
│   │
│   ├── replay-engine/         # Playwright-based replay + coverage
│   │   └── src/
│   │       ├── browser/       # launcher, anti-flake flags, PRNG seeding, scoped clock
│   │       ├── network/       # targeted route mocking, WS, SSE, CORS handling
│   │       ├── interactions/  # click, input, scroll, navigation, touch executor
│   │       ├── coverage/      # V8 Profiler, bitmap conversion, version stamping
│   │       ├── settle/        # Node.js orchestrated polling + font loading + img.complete
│   │       └── screenshot/    # capture, smart retry, majority vote
│   │
│   ├── diff-engine/           # pixelmatch + masking + content-hash + HTML report
│   │
│   ├── test-generator/        # Coverage-based set cover + staleness + session skipping
│   │   └── src/
│   │       ├── set-cover.ts
│   │       ├── deduplicator.ts
│   │       ├── staleness.ts
│   │       └── session-skipper.ts  # TurboSnap equivalent
│   │
│   ├── cli/                   # Commander.js CLI
│   │   └── src/commands/      # record, replay, diff, generate, approve, ci, status, prune, push/pull-baselines
│   │
│   └── shared/                # Types + utilities
│       └── src/
│           ├── types/         # session, replay, coverage, diff, suite
│           ├── storage/       # content-addressable blob store
│           └── prng/          # seeded PRNG (xoshiro256**)
│
├── turbo.json
├── pnpm-workspace.yaml
└── biome.json
```

---

## Implementation Reference

Concrete implementation patterns, data structures, and lessons learned from researching OpenReplay, Sentry, rrweb, Highlight.io, PostHog, Playwright internals, Lost Pixel, Argos CI, Chromatic, v8-to-istanbul, c8, and pixelmatch/sharp.

### Recording SDK: Network Interception

**Fetch interception** — Direct monkey-patch, NOT ES6 Proxy. Every production recording tool (OpenReplay, Sentry, Highlight.io) replaces `window.fetch` directly. ES6 Proxy adds overhead and breaks `instanceof` checks.

```typescript
// Max request/response body size to capture (bytes). Configurable via config.recording.maxBodyBytes.
// SDK default: 256KB. Playwright recorder default: 1MB.
// Bodies larger than this are truncated (bodyTruncated: true in event), which means
// body-fingerprinted FIFO matching may degrade for large POST bodies.
// 256KB covers GraphQL queries (typically < 10KB) and most REST API payloads.
// File uploads (multi-MB) are intentionally truncated — too large for session JSON.
const MAX_BODY_SIZE = config.recording?.maxBodyBytes ?? 256 * 1024; // 256KB default

// Store original before any framework loads
const _originalFetch = window.fetch.bind(window);
const _originalXHROpen = XMLHttpRequest.prototype.open;
const _originalXHRSend = XMLHttpRequest.prototype.send;

// SDK timestamps are relative to session start (ms offset, NOT epoch).
// The SDK session manager sets this when recording begins.
// IMPORTANT: All `Date.now()` calls below should use `sdkTs()` in production code.
// Shown as `Date.now()` in snippets for brevity — the SDK implementation replaces all with sdkTs().
let _sessionStartTime = Date.now();
function sdkTs() { return Date.now() - _sessionStartTime; }

window.fetch = function(input: RequestInfo | URL, init?: RequestInit): Promise<Response> {
  const url = typeof input === 'string' ? input : input instanceof URL ? input.toString() : input.url;
  const method = init?.method || 'GET';
  const requestId = nextRequestId();
  const startTime = performance.now();

  // Record request
  recordNetworkEvent({
    type: 'request', requestId, url, method,
    headers: sanitizeHeaders(init?.headers),
    body: shouldCaptureBody(url) ? truncate(init?.body, MAX_BODY_SIZE) : undefined,
    t_ms: sdkTs(),
  });

  return _originalFetch(input, init).then(
    (response) => {
      const cloned = response.clone(); // CRITICAL: clone before app consumes
      cloned.text().then(bodyText => {
        recordNetworkEvent({
          type: 'response', requestId, url, method,
          status: response.status,
          headers: sanitizeHeaders(response.headers),
          body: truncate(bodyText, MAX_BODY_SIZE),
          durationMs: performance.now() - startTime,
        });
      }).catch(() => {}); // Don't break app on recording failure
      return response;
    },
    (error) => {
      recordNetworkEvent({
        type: 'response', requestId, url, method,
        error: error.message, durationMs: performance.now() - startTime,
      });
      throw error; // Re-throw — never swallow app errors
    }
  );
};
```

**XHR interception** — Prototype patching (Sentry pattern). Patch `open` to capture URL/method, patch `send` to capture request body, listen to `loadend` for response.

```typescript
XMLHttpRequest.prototype.open = function(method: string, url: string, ...args: any[]) {
  this.__eyespy = { method, url, requestId: nextRequestId(), startTime: performance.now() };
  return _originalXHROpen.apply(this, [method, url, ...args]);
};

XMLHttpRequest.prototype.send = function(body?: any) {
  const meta = this.__eyespy;
  if (meta) {
    recordNetworkEvent({ type: 'request', ...meta, body: truncate(body, MAX_BODY_SIZE) });
    this.addEventListener('loadend', () => {
      recordNetworkEvent({
        type: 'response', requestId: meta.requestId,
        status: this.status, body: truncate(this.responseText, MAX_BODY_SIZE),
        durationMs: performance.now() - meta.startTime,
      });
    });
  }
  return _originalXHRSend.apply(this, [body]);
};
```

**WebSocket interception** — ES6 Proxy on constructor (Highlight.io pattern). Proxy IS correct here because we need to intercept construction.

```typescript
const _OriginalWebSocket = window.WebSocket;
window.WebSocket = new Proxy(_OriginalWebSocket, {
  construct(target, args) {
    const [url, protocols] = args;
    const ws = new target(url, protocols);
    const wsId = nextWsId();

    recordNetworkEvent({ type: 'ws-open', wsId, url, t_ms: sdkTs() });

    ws.addEventListener('message', (e: MessageEvent) => {
      recordNetworkEvent({
        type: 'ws-receive', wsId, url,
        data: typeof e.data === 'string' ? truncate(e.data, MAX_BODY_SIZE) : '<binary>',
        t_ms: sdkTs(),
      });
    });
    ws.addEventListener('close', (e: CloseEvent) => {
      recordNetworkEvent({ type: 'ws-close', wsId, url, code: e.code, t_ms: sdkTs() });
    });

    const _originalSend = ws.send.bind(ws);
    ws.send = function(data: any) {
      recordNetworkEvent({
        type: 'ws-send', wsId, url,
        data: typeof data === 'string' ? truncate(data, MAX_BODY_SIZE) : '<binary>',
        t_ms: sdkTs(),
      });
      return _originalSend(data);
    };
    return ws;
  }
}) as any;
```

**SSE (EventSource) interception** — Gap in ecosystem. No major recording tool implements this. Custom implementation required.

```typescript
const _OriginalEventSource = window.EventSource;
window.EventSource = new Proxy(_OriginalEventSource, {
  construct(target, args) {
    const [url, init] = args;
    const es = new target(url, init);
    const sseId = nextSseId();

    recordNetworkEvent({ type: 'sse-open', sseId, url, t_ms: sdkTs() });

    const _originalAddEventListener = es.addEventListener.bind(es);
    es.addEventListener = function(type: string, listener: any, options?: any) {
      const wrappedListener = (event: MessageEvent) => {
        recordNetworkEvent({
          type: 'sse-event', sseId, url, eventType: type,
          data: event.data, lastEventId: event.lastEventId,
          t_ms: sdkTs(),
        });
        listener(event);
      };
      return _originalAddEventListener(type, wrappedListener, options);
    };

    // Also intercept onmessage property assignment
    let _onmessage: any = null;
    Object.defineProperty(es, 'onmessage', {
      get: () => _onmessage,
      set: (handler) => {
        _onmessage = handler;
        _originalAddEventListener('message', (event: MessageEvent) => {
          recordNetworkEvent({
            type: 'sse-event', sseId, url, eventType: 'message',
            data: event.data, t_ms: sdkTs(),
          });
        });
      }
    });
    return es;
  }
}) as any;
```

**SDK initialization verification** — Runtime check that our proxies are still in place (libraries that cache `window.fetch` at import time bypass our interception):

```typescript
function verifyInterception(): { fetch: boolean; xhr: boolean; ws: boolean } {
  return {
    fetch: window.fetch !== _originalFetch,  // We replaced it
    xhr: XMLHttpRequest.prototype.open !== _originalXHROpen,
    ws: window.WebSocket !== _OriginalWebSocket,
  };
}
// Call after DOMContentLoaded, warn if any are false
```

### Playwright Recorder: Page-Level Event Capture

The CLI recorder (`eyespy record --url`) uses `addInitScript()` to inject capture-phase DOM event listeners into the page — the same approach Playwright's codegen and Meticulous use internally. CDP's `Input` domain only has *commands* (for injecting synthetic input), NOT events for observing user input. All production recording tools (rrweb, Meticulous, OpenReplay) use page-level `addEventListener` in capture phase.

**Recording UX**: When recording starts, the CLI opens a Playwright browser in headed mode. A small floating overlay (absolute-positioned, z-index: 2147483647) shows a "Stop Recording" button + event counter. The overlay is excluded from selector generation and screenshots. Recording also stops on Ctrl+C in the terminal.

**Architecture**: `addInitScript()` injects the event capture script, which communicates back to Node.js via `page.exposeBinding()`. This runs the capture code in the page's main world (not an isolated world), so it sees all DOM elements including those created by frameworks. For Shadow DOM, capture-phase listeners on `document` receive events that bubble out of open shadow roots.

```typescript
async function startRecording(page: Page, blobStore: BlobStore): Promise<RecordedEvent[]> {
  const events: RecordedEvent[] = [];
  let requestCounter = 0;

  // Expose binding for the injected script to send events back to Node.js
  await page.exposeBinding('__eyespy_recordEvent', async ({ page: p }, event: RecordedEvent) => {
    events.push(event);
  });

  // Inject capture-phase event listeners via addInitScript
  // This survives navigations and runs before any page JS
  await page.context().addInitScript(() => {
    // Selector generation — runs in page context
    function generateSelector(el: Element): { primary: string; fallbacks: string[]; fingerprint: any } {
      const strategies: string[] = [];

      // Strategy 1: data-testid (highest priority)
      const testId = el.getAttribute('data-testid') || el.getAttribute('data-test-id');
      if (testId) strategies.push(`[data-testid="${testId}"]`);

      // Strategy 2: id (if not dynamically generated)
      if (el.id && !/^[0-9]|[:.]|--/.test(el.id)) strategies.push(`#${el.id}`);

      // Strategy 3: role + aria-label
      const role = el.getAttribute('role') || el.tagName.toLowerCase();
      const ariaLabel = el.getAttribute('aria-label');
      if (ariaLabel) strategies.push(`[role="${role}"][aria-label="${ariaLabel}"]`);

      // Strategy 4: structural CSS path (fallback)
      function cssPath(element: Element): string {
        const parts: string[] = [];
        let current: Element | null = element;
        while (current && current !== document.body) {
          let sel = current.tagName.toLowerCase();
          if (current.id && !/^[0-9]|[:.]|--/.test(current.id)) {
            parts.unshift(`#${current.id}`);
            break;
          }
          const parent = current.parentElement;
          if (parent) {
            const siblings = Array.from(parent.children).filter(c => c.tagName === current!.tagName);
            if (siblings.length > 1) {
              sel += `:nth-of-type(${siblings.indexOf(current) + 1})`;
            }
          }
          parts.unshift(sel);
          current = parent;
        }
        return parts.join(' > ');
      }

      const primary = strategies[0] || cssPath(el);
      const fallbacks = strategies.slice(1);
      const path = cssPath(el);
      if (!strategies.includes(path)) fallbacks.push(path);

      return {
        primary,
        fallbacks,
        fingerprint: {
          text: (el as HTMLElement).innerText?.substring(0, 50),
          rect: el.getBoundingClientRect().toJSON(),
          tagName: el.tagName.toLowerCase(),
        },
      };
    }

    // Skip recording overlay clicks
    function isRecordingOverlay(el: Element): boolean {
      return !!(el.closest && el.closest('[data-eyespy-overlay]'));
    }

    // Session start time — all event timestamps are offsets from this
    const sessionStartTime = Date.now();
    function ts() { return Date.now() - sessionStartTime; }

    // Capture-phase listeners on document — see events before any stopPropagation
    document.addEventListener('click', (e) => {
      const el = e.target as Element;
      if (!el || isRecordingOverlay(el)) return;
      (window as any).__eyespy_recordEvent({
        type: 'click', t_ms: ts(), selector: generateSelector(el),
        x: (e as MouseEvent).clientX, y: (e as MouseEvent).clientY,
        button: (e as MouseEvent).button,
        modifiers: { meta: e.metaKey, ctrl: e.ctrlKey, shift: e.shiftKey, alt: e.altKey },
      });
    }, true); // capture phase

    document.addEventListener('dblclick', (e) => {
      const el = e.target as Element;
      if (!el || isRecordingOverlay(el)) return;
      (window as any).__eyespy_recordEvent({
        type: 'dblclick', t_ms: ts(), selector: generateSelector(el),
        x: (e as MouseEvent).clientX, y: (e as MouseEvent).clientY,
      });
    }, true);

    document.addEventListener('input', (e) => {
      const el = e.target as Element;
      if (!el || isRecordingOverlay(el)) return;
      const value = (el as HTMLInputElement).type === 'password' ? '***' : (el as HTMLInputElement).value;
      (window as any).__eyespy_recordEvent({
        type: 'input', t_ms: ts(), selector: generateSelector(el),
        value, inputType: (e as InputEvent).inputType,
      });
    }, true);

    document.addEventListener('keydown', (e) => {
      (window as any).__eyespy_recordEvent({
        type: 'keydown', t_ms: ts(),
        key: (e as KeyboardEvent).key, code: (e as KeyboardEvent).code,
        modifiers: { meta: e.metaKey, ctrl: e.ctrlKey, shift: e.shiftKey, alt: e.altKey },
      });
    }, true);

    document.addEventListener('keyup', (e) => {
      (window as any).__eyespy_recordEvent({
        type: 'keyup', t_ms: ts(),
        key: (e as KeyboardEvent).key, code: (e as KeyboardEvent).code,
      });
    }, true);

    // Scroll — debounced (capture final position, not every pixel)
    let scrollTimer: any = null;
    document.addEventListener('scroll', (e) => {
      clearTimeout(scrollTimer);
      scrollTimer = setTimeout(() => {
        const el = e.target === document ? null : e.target as Element;
        (window as any).__eyespy_recordEvent({
          type: 'scroll', t_ms: ts(),
          selector: el ? generateSelector(el) : null,
          x: el ? el.scrollLeft : window.scrollX,
          y: el ? el.scrollTop : window.scrollY,
        });
      }, 100);
    }, true);

    // Navigation — History API + popstate
    const _pushState = history.pushState.bind(history);
    const _replaceState = history.replaceState.bind(history);
    history.pushState = function(...args: any[]) {
      _pushState(...args);
      (window as any).__eyespy_recordEvent({
        type: 'navigate', t_ms: ts(), url: location.href, navigationType: 'push',
      });
    };
    history.replaceState = function(...args: any[]) {
      _replaceState(...args);
      (window as any).__eyespy_recordEvent({
        type: 'navigate', t_ms: ts(), url: location.href, navigationType: 'replace',
      });
    };
    window.addEventListener('popstate', () => {
      (window as any).__eyespy_recordEvent({
        type: 'navigate', t_ms: ts(), url: location.href, navigationType: 'popstate',
      });
    });
  });

  // Network capture via Playwright's high-level API (Node.js side, not page-injected)
  // This avoids conflicts with app code and captures all traffic including Service Worker bypass
  //
  // CRITICAL: Use Request object identity as Map key, NOT url+method string.
  // String keys collide when multiple concurrent requests hit the same endpoint
  // (e.g., two parallel POST /api/graphql). response.request() returns the SAME
  // Request object that fired in the 'request' event, so identity lookup is safe.
  const requestMap = new Map<Request, { id: string; startTime: number }>();
  const sessionStartMs = Date.now(); // All timestamps relative to this
  const relTs = () => Date.now() - sessionStartMs;

  page.on('request', (request) => {
    const requestId = `req_${++requestCounter}`;
    requestMap.set(request, { id: requestId, startTime: Date.now() });
    // IMPORTANT: Capture request body (postData) for body-fingerprinted FIFO matching.
    // Without this, concurrent POST requests to the same URL (e.g., GraphQL) can't be
    // distinguished during replay. `request.postData()` returns null for GET/HEAD/DELETE.
    const body = request.postData() ?? undefined;
    events.push({
      type: 'network', t_ms: relTs(),
      event: {
        type: 'request', requestId, url: request.url(), method: request.method(),
        headers: sanitizeHeaders(request.headers()), body, t_ms: relTs(),
      },
    });
  });

  page.on('response', async (response) => {
    const meta = requestMap.get(response.request());
    const requestId = meta?.id || `req_${++requestCounter}`;
    const startTime = meta?.startTime || Date.now();
    requestMap.delete(response.request());

    try {
      const body = await response.body();
      const digest = await blobStore.put(body);
      events.push({
        type: 'network', t_ms: relTs(),
        event: {
          type: 'response', requestId, url: response.url(),
          method: response.request().method(), status: response.status(),
          headers: sanitizeHeaders(response.headers()),
          body: { kind: 'blob', digest: `sha256:${digest}`, truncated: false, byteLength: body.length, contentType: response.headers()['content-type'] },
          durationMs: Date.now() - startTime,
        },
      });
    } catch {
      // Some responses have no body (204, redirects, aborted)
    }
  });

  return events; // Caller stops recording on user signal
}
```

**Why page-level listeners, not CDP**: CDP's `Input` domain provides `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` which are *commands* for injecting synthetic input into a page — NOT events for observing real user input. There is no CDP event that fires when a user clicks or types. Playwright's codegen uses `extendInjectedScript()` which injects page-level DOM event listeners. Meticulous uses rrweb which also uses `addEventListener` on `document`. This is the only viable approach for capturing user interactions.

**Shadow DOM handling**: Capture-phase listeners on `document` receive events from open Shadow DOM roots via event bubbling/composition. Closed Shadow DOM roots are invisible — document as a known limitation. `event.composedPath()` provides the full path through shadow boundaries.

**Limitations vs SDK approach**: The Playwright recorder captures interactions only while the recording browser is open. The SDK can record production traffic passively. The Playwright recorder has more reliable network capture (via Playwright's API) but less reliable input capture (page-level listeners can miss `stopImmediatePropagation` on capture phase — rare but possible).

### Recording SDK: Key Data Structures

**Session format** — Flat event array with **relative timestamps** (milliseconds from session start). Network bodies stored separately in blobs.

**Timestamp convention**: All `timestamp` fields in `RecordedEvent` are **milliseconds from session start** (offset, NOT absolute epoch). Recorders compute: `timestamp = Date.now() - sessionStartTime`. This makes sessions portable (replayable regardless of when they were recorded) and makes the orchestrator's delta calculation trivial (`delta = event.t_ms - lastEventTimestamp`, starting from 0). The orchestrator advances the mocked clock by `delta` ms between events, and pauses at `new Date(startTime.getTime() + event.t_ms)` for screenshots — both formulas are correct because `event.t_ms` is an offset.

```typescript
interface RecordedSession {
  formatVersion: 1;              // Schema version — reject with clear error if mismatch
  id: string;                    // UUID v4
  startedAt: string;             // ISO 8601 — absolute wall-clock time of session start
  endedAt: string;
  url: string;                   // Starting URL
  viewport: { width: number; height: number };
  userAgent: string;
  captureMethod: 'playwright' | 'sdk';  // Which recorder produced this session
  observedOrigins?: string[];    // All origins seen during recording (Playwright recorder auto-populates; SDK omits)
  events: RecordedEvent[];       // Sorted by seq ascending, t_ms non-decreasing (ms offset from session start, NOT epoch)
  // Network body storage: Response/request bodies use the `BodyRef` discriminated union
  // on individual NetworkEvents. Inline bodies are stored directly; large bodies are
  // stored in .eyespy/blobs/ and referenced by `sha256:` digest.
  // The SDK receiver converts inline bodies to blob references on ingest when they exceed threshold.
  // No separate networkBodies map — body references live on the events themselves.
}

// Every event has:
//   seq: monotonically increasing integer (0-based). Canonical ordering field.
//        t_ms may collide (same millisecond); seq never does. Screenshot keys,
//        receiver validation, and event identity all use seq.
//   t_ms: milliseconds from session start (offset, NOT epoch).
//        Epoch timestamps MUST NOT appear in stored session events.
type RecordedEvent =
  | { seq: number; type: 'click'; t_ms: number; selector: SelectorBundle; x: number; y: number; button: number; modifiers: Modifiers }
  | { seq: number; type: 'dblclick'; t_ms: number; selector: SelectorBundle; x: number; y: number }
  | { seq: number; type: 'input'; t_ms: number; selector: SelectorBundle; value: string; inputType?: string }
  | { seq: number; type: 'keydown'; t_ms: number; key: string; code: string; modifiers: Modifiers; replay?: boolean }
  | { seq: number; type: 'keyup'; t_ms: number; key: string; code: string; replay?: boolean }
  | { seq: number; type: 'scroll'; t_ms: number; selector: SelectorBundle | null; x: number; y: number }
  | { seq: number; type: 'navigate'; t_ms: number; url: string; navigationType: 'load' | 'push' | 'replace' | 'popstate' }
  | { seq: number; type: 'resize'; t_ms: number; width: number; height: number }
  | { seq: number; type: 'focus'; t_ms: number; selector: SelectorBundle }
  | { seq: number; type: 'blur'; t_ms: number; selector: SelectorBundle }
  | { seq: number; type: 'pointerdown'; t_ms: number; selector: SelectorBundle; x: number; y: number; pointerId: number }
  | { seq: number; type: 'pointermove'; t_ms: number; x: number; y: number; pointerId: number }
  | { seq: number; type: 'pointerup'; t_ms: number; x: number; y: number; pointerId: number }
  | { seq: number; type: 'touchstart'; t_ms: number; selector: SelectorBundle; touches: Touch[] }
  | { seq: number; type: 'touchmove'; t_ms: number; touches: Touch[] }
  | { seq: number; type: 'touchend'; t_ms: number; touches: Touch[] }
  | { seq: number; type: 'touchcancel'; t_ms: number; touches: Touch[] }
  | { seq: number; type: 'pointercancel'; t_ms: number; x: number; y: number; pointerId: number }
  | { seq: number; type: 'change'; t_ms: number; selector: SelectorBundle; value: string }
  | { seq: number; type: 'submit'; t_ms: number; selector: SelectorBundle }
  | { seq: number; type: 'wheel'; t_ms: number; selector: SelectorBundle | null; deltaX: number; deltaY: number }
  | { seq: number; type: 'dragstart'; t_ms: number; selector: SelectorBundle; x: number; y: number }
  | { seq: number; type: 'dragover'; t_ms: number; selector: SelectorBundle; x: number; y: number }
  | { seq: number; type: 'dragend'; t_ms: number; selector: SelectorBundle; x: number; y: number }
  | { seq: number; type: 'drop'; t_ms: number; selector: SelectorBundle; x: number; y: number }
  | { seq: number; type: 'paste'; t_ms: number; selector: SelectorBundle; text: string }
  | { seq: number; type: 'copy'; t_ms: number; selector: SelectorBundle }
  | { seq: number; type: 'cut'; t_ms: number; selector: SelectorBundle }
  | { seq: number; type: 'network'; t_ms: number; event: NetworkEvent }
  | { seq: number; type: 'screenshot-marker'; t_ms: number; label: string };

interface SelectorBundle {
  primary: string;               // Best selector (data-testid > id > role > css path)
  fallbacks: string[];           // Alternative selectors
  fingerprint: {                 // Heuristic for validation
    text?: string;               // First 50 chars of innerText
    rect?: DOMRect;              // Bounding box at record time
    tagName: string;
  };
}

interface Modifiers {
  meta: boolean;
  ctrl: boolean;
  shift: boolean;
  alt: boolean;
}

// Body representation for network events. Handles text, binary, truncated, and omitted bodies unambiguously.
type BodyRef =
  | { kind: 'none' }                                                                    // No body (GET request, 204 response, etc.)
  | { kind: 'inline'; encoding: 'utf8' | 'base64'; data: string; truncated: boolean; byteLength: number; contentType?: string }  // Small bodies stored inline
  | { kind: 'blob'; digest: string /* sha256:... */; truncated: boolean; byteLength: number; contentType?: string };               // Large bodies stored in blob store

type NetworkEvent =
  | { type: 'request'; requestId: string; url: string; method: string; headers?: Record<string, string>; body: BodyRef; t_ms: number }
  | { type: 'response'; requestId: string; url: string; method: string; status: number; headers?: Record<string, string>; body: BodyRef; durationMs: number; error?: string }
  | { type: 'ws-open'; wsId: string; url: string; t_ms: number }
  | { type: 'ws-send'; wsId: string; url: string; data: string; t_ms: number }
  | { type: 'ws-receive'; wsId: string; url: string; data: string; t_ms: number }
  | { type: 'ws-close'; wsId: string; url: string; code: number; t_ms: number }
  | { type: 'sse-open'; sseId: string; url: string; t_ms: number }
  | { type: 'sse-event'; sseId: string; url: string; eventType: string; data: string; lastEventId?: string; t_ms: number };
// Note: Response bodies use BodyRef — inline for small bodies, blob digest for large.
// Blob digests use the canonical format `sha256:<hex64>` and map to .eyespy/blobs/.

// --- Replay types ---

type ReplayResult =
  | { status: 'success'; screenshots: Map<string, Buffer>; coverage?: CoverageBitmap; healingLog: HealingEvent[] }
  | { status: 'replay-error'; reason: string; diagnosticScreenshot?: Buffer | null }
  | { status: 'unreplayable'; reason: string; healingLog: HealingEvent[] };

interface HealingEvent {
  event: RecordedEvent;                // The interaction that needed healing
  originalSelector: string;            // Primary selector that failed
  healedSelector?: string;             // Fallback selector that worked (if any)
  strategy?: string;                   // Which healing strategy matched ("testid", "role+label", etc.)
  confidence?: number;                 // 0.0-1.0 confidence of the healed match
  error?: string;                      // Error message if all strategies failed
  skipped: boolean;                    // Whether the interaction was skipped entirely
}

/** Wraps baselines.json for ergonomic lookup */
interface BaselineManifest {
  version: number;
  baselines: Record<string, Record<string, { digest: string; width: number; height: number }>>;
  getDigest(sessionId: string, screenshotKey: string): string | undefined;
  setDigest(sessionId: string, screenshotKey: string, digest: string, width: number, height: number): void;
}

/** Maps branch index → source location for session skipping */
interface BranchIndex {
  entries: Record<number, { scriptUrl: string; startOffset: number; endOffset: number }>;
}
```

**Session validation** (on load, before replay):
1. Check `formatVersion` — reject with clear error if mismatch (e.g., "Session format v2 requires @eyespy/cli >= 0.3.0")
2. Verify `seq` is strictly increasing and `t_ms` is non-decreasing
3. Verify all `BodyRef(kind='blob')` digests exist in blob store — collect missing digests, warn (not error) with count
4. Verify session has at least one interaction event (not just network events)
5. Verify `captureMethod` field exists (old sessions without it: assume `'playwright'`)
6. Schema validation via TypeScript type guards at runtime (not JSON Schema — too heavy for CLI)

### Replay Engine: PRNG Seeding

The most critical piece of the determinism stack. Must run before ANY page JavaScript via `addInitScript()`, which uses CDP `Page.addScriptToEvaluateOnNewDocument` internally — guaranteed to execute before page scripts.

```javascript
// Injected via page.addInitScript(script, seed)
(function(seed) {
  // xoshiro128** — fast, high-quality 32-bit PRNG
  function xoshiro128ss(a, b, c, d) {
    return function() {
      const t = b << 9;
      let r = a * 5; r = (r << 7 | r >>> 25) * 9;
      c ^= a; d ^= b; b ^= c; a ^= d; c ^= t;
      d = d << 11 | d >>> 21;
      return (r >>> 0) / 4294967296;
    };
  }

  // Seed from string via simple hash
  function hashSeed(str) {
    let h1 = 0xdeadbeef, h2 = 0x41c6ce57, h3 = 0x7f4a7c15, h4 = 0x9e3779b9;
    for (let i = 0; i < str.length; i++) {
      const k = str.charCodeAt(i);
      h1 = Math.imul(h1 ^ k, 2654435761); h2 = Math.imul(h2 ^ k, 1597334677);
      h3 = Math.imul(h3 ^ k, 2246822507); h4 = Math.imul(h4 ^ k, 3266489909);
    }
    h1 = Math.imul(h1 ^ (h1 >>> 16), 2246822507) ^ Math.imul(h2 ^ (h2 >>> 13), 3266489909);
    h2 = Math.imul(h2 ^ (h2 >>> 16), 2246822507) ^ Math.imul(h1 ^ (h1 >>> 13), 3266489909);
    h3 = Math.imul(h3 ^ (h3 >>> 16), 2246822507) ^ Math.imul(h4 ^ (h4 >>> 13), 3266489909);
    h4 = Math.imul(h4 ^ (h4 >>> 16), 2246822507) ^ Math.imul(h3 ^ (h3 >>> 13), 3266489909);
    return [h1 >>> 0, h2 >>> 0, h3 >>> 0, h4 >>> 0];
  }

  const [a, b, c, d] = hashSeed(String(seed));
  const prng = xoshiro128ss(a, b, c, d);

  // Replace Math.random
  Math.random = prng;

  // Replace crypto.getRandomValues
  const _origGetRandomValues = crypto.getRandomValues.bind(crypto);
  crypto.getRandomValues = function(array) {
    // TypedArray — fill with deterministic bytes
    if (array instanceof Uint8Array || array instanceof Uint8ClampedArray) {
      for (let i = 0; i < array.length; i++) array[i] = Math.floor(prng() * 256);
    } else if (array instanceof Uint16Array) {
      for (let i = 0; i < array.length; i++) array[i] = Math.floor(prng() * 65536);
    } else if (array instanceof Uint32Array) {
      for (let i = 0; i < array.length; i++) array[i] = Math.floor(prng() * 4294967296);
    } else {
      // Fallback for Int8Array, Int16Array, Int32Array, etc.
      const bytes = new Uint8Array(array.byteLength);
      for (let i = 0; i < bytes.length; i++) bytes[i] = Math.floor(prng() * 256);
      new Uint8Array(array.buffer, array.byteOffset, array.byteLength).set(bytes);
    }
    return array;
  };

  // Replace crypto.randomUUID
  crypto.randomUUID = function() {
    return '10000000-1000-4000-8000-100000000000'.replace(/[018]/g, function(c) {
      var n = +c;
      return (n ^ Math.floor(prng() * 16) >> (n / 4)).toString(16);
    });
  };
})(arguments[0]); // Playwright passes the seed as arguments[0]
```

### Replay Engine: Network Mocking

**HTTP routes** — Use `browserContext.route()` (survives navigations) with targeted per-origin patterns.

```typescript
// Normalize URLs for consistent FIFO key matching (Phase 1a: basic normalization)
// Without this, trivial differences (trailing slash, default port, param order) cause mock misses.
function normalizeUrl(rawUrl: string): string {
  const url = new URL(rawUrl);
  url.hash = '';                          // Strip fragment
  if (url.port === '80' && url.protocol === 'http:') url.port = '';
  if (url.port === '443' && url.protocol === 'https:') url.port = '';
  url.pathname = url.pathname.replace(/\/+$/, '') || '/'; // Normalize trailing slashes
  url.searchParams.sort();                // Deterministic param order
  return url.toString();
}

// Find the response event matching a request by requestId
function findResponse(events: RecordedEvent[], requestId: string): NetworkEvent | undefined {
  for (const event of events) {
    if (event.type === 'network' && event.event.type === 'response' && event.event.requestId === requestId) {
      return event.event;
    }
  }
  return undefined;
}

// Build network response queues keyed by method + normalized URL + request body fingerprint.
//
// CRITICAL: Why body fingerprinting matters for concurrent requests.
// The naive approach (FIFO keyed by METHOD:URL) breaks when multiple concurrent requests
// hit the same endpoint — common in GraphQL (all POSTs to /graphql), batch APIs, and
// polling. If two POST /api/graphql requests arrive in different order during replay
// vs recording, FIFO returns the wrong response → wrong data renders → false positive diff.
//
// Solution: For methods with bodies (POST, PUT, PATCH), include a truncated SHA-256 hash
// of the request body in the queue key. This gives each unique request its own queue.
// For bodyless methods (GET, DELETE, HEAD, OPTIONS), use METHOD:URL only (FIFO).
// If body hashing produces a queue miss (body changed between record/replay), fall back
// to the METHOD:URL-only queue.
//
// Trade-off: This means a request body MUST be recorded to get correct matching. The
// SDK and Playwright recorder both capture request bodies — but if a body is too large
// and was truncated, the fingerprint won't match. Per-capture-method defaults apply:
// SDK default 256KB, Playwright recorder default 1MB (see `maxBodyBytes` in Decisions).

// Uses createHash from 'crypto' (already imported for blob store)
const BODY_METHODS = new Set(['POST', 'PUT', 'PATCH']);

function requestBodyFingerprint(body: BodyRef): string {
  if (body.kind === 'none') return '';
  // Truncated SHA-256 (first 12 hex chars) — enough to distinguish different requests,
  // short enough to not bloat queue key maps in memory.
  if (body.kind === 'blob') return body.digest.replace('sha256:', '').substring(0, 12);
  return createHash('sha256').update(body.data).digest('hex').substring(0, 12);
}

function buildNetworkQueues(session: RecordedSession): {
  bodyQueues: Map<string, NetworkEvent[]>;   // "METHOD:url:bodyFingerprint" → FIFO (for POST/PUT/PATCH)
  urlQueues: Map<string, NetworkEvent[]>;    // "METHOD:url" → FIFO (fallback, and GET/DELETE/etc.)
} {
  const bodyQueues = new Map<string, NetworkEvent[]>();
  const urlQueues = new Map<string, NetworkEvent[]>();

  for (const event of session.events) {
    if (event.type !== 'network' || event.event.type !== 'request') continue;
    const normalUrl = normalizeUrl(event.event.url);
    const urlKey = `${event.event.method}:${normalUrl}`;
    const response = findResponse(session.events, event.event.requestId);
    if (!response) continue;

    // Always add to URL-only queue (fallback)
    if (!urlQueues.has(urlKey)) urlQueues.set(urlKey, []);
    urlQueues.get(urlKey)!.push(response);

    // For body methods, also add to body-fingerprinted queue
    if (BODY_METHODS.has(event.event.method) && event.event.body.kind !== 'none') {
      const bodyKey = `${urlKey}:${requestBodyFingerprint(event.event.body)}`;
      if (!bodyQueues.has(bodyKey)) bodyQueues.set(bodyKey, []);
      bodyQueues.get(bodyKey)!.push(response);
    }
  }
  return { bodyQueues, urlQueues };
}

// Two-layer routing architecture:
// Layer 1: Global egress policy (catch-all, never fulfills — avoids flakiness #22338)
// Layer 2: Per-origin mock fulfillment (only registered in mock mode)

async function registerRoutes(
  context: BrowserContext,
  queues: { bodyQueues: Map<string, NetworkEvent[]>; urlQueues: Map<string, NetworkEvent[]> },
  mode: 'mock' | 'live',
  allowedOrigins: Set<string>,
  allowLiveExternalEgress: boolean,
  originPolicies?: Record<string, 'mock' | 'live' | 'blocked'>,  // Per-origin overrides
) {
  // --- Layer 1: Global egress enforcement ---
  // This route intercepts ALL requests but NEVER fulfills responses.
  // It only allows or blocks based on origin, then falls back to Layer 2 or the network.
  await context.route('**/*', async (route) => {
    const requestOrigin = new URL(route.request().url()).origin;
    // Per-origin policy override takes precedence
    const originPolicy = originPolicies?.[requestOrigin];
    if (originPolicy === 'blocked') {
      await route.abort('blockedbyclient');
    } else if (allowedOrigins.has(requestOrigin) || allowLiveExternalEgress) {
      await route.fallback(); // Pass to Layer 2 (if registered) or network
    } else {
      await route.abort('blockedbyclient'); // SSRF: non-allowed origin
    }
  });

  // --- Layer 2: Mock fulfillment (mock mode only) ---
  if (mode !== 'mock') return; // Live mode: Layer 1 handles everything

  const origins = new Set<string>();
  for (const key of queues.urlQueues.keys()) {
    const url = key.substring(key.indexOf(':') + 1);
    origins.add(new URL(url).origin);
  }

  for (const origin of origins) {
    await context.route(`${origin}/**`, async (route) => {
      const request = route.request();

      // ResourceType gate: only mock fetch/xhr data requests
      const rt = request.resourceType();
      if (rt !== 'fetch' && rt !== 'xhr') {
        await route.continue(); // Non-data resources pass through (already allowed by Layer 1)
        return;
      }

      const normalUrl = normalizeUrl(request.url());
      const urlKey = `${request.method()}:${normalUrl}`;

      // Two-tier FIFO matching: body-fingerprinted first, URL-only fallback
      let matched: NetworkEvent | undefined;

      if (BODY_METHODS.has(request.method())) {
        const postData = request.postData() ?? '';
        const bodyKey = `${urlKey}:${requestBodyFingerprint(postData)}`;
        const bodyQueue = queues.bodyQueues.get(bodyKey);
        if (bodyQueue && bodyQueue.length > 0) {
          matched = bodyQueue.shift()!;
          const urlQueue = queues.urlQueues.get(urlKey);
          if (urlQueue) {
            const idx = urlQueue.indexOf(matched);
            if (idx >= 0) urlQueue.splice(idx, 1);
          }
        }
      }

      if (!matched) {
        const urlQueue = queues.urlQueues.get(urlKey);
        if (urlQueue && urlQueue.length > 0) {
          matched = urlQueue.shift()!;
        }
      }

      if (matched) {
        const body = matched.body.kind === 'blob' ? await blobStore.get(matched.body.digest.replace('sha256:', ''))
                   : matched.body.kind === 'inline' ? Buffer.from(matched.body.data, matched.body.encoding === 'base64' ? 'base64' : 'utf8')
                   : undefined;
        await route.fulfill({ status: matched.status, headers: matched.headers, body });
      } else {
        await route.abort('connectionrefused'); // Unrecorded fetch/xhr in mock mode
      }
    });
  }
}
```

**WebSocket mocking** — `page.routeWebSocket()` (Playwright 1.48+). Default behavior does NOT connect to real server — we control both sides.

**Critical: Clock interaction** — Route handlers (including `routeWebSocket`) execute in **Node.js**, NOT the browser. Playwright's `clock.install()` only mocks browser-side timers. Using `setTimeout` in a handler uses real time, not mocked time — messages arrive at wrong times relative to the page's frozen clock.

**Solution: Clock-coordinated delivery** — Don't use setTimeout in handlers. Instead, send WS messages immediately when triggered, and control timing from the replay orchestrator using `clock.runFor()`.

```typescript
async function mockWebSockets(page: Page, session: RecordedSession) {
  const wsEvents = session.events.filter(e => e.type === 'network' && e.event.type?.startsWith('ws-'));

  // Group by wsId, build timeline
  const wsTimelines = new Map<string, NetworkEvent[]>();
  for (const event of wsEvents) {
    const wsId = event.event.wsId;
    if (!wsTimelines.has(wsId)) wsTimelines.set(wsId, []);
    wsTimelines.get(wsId)!.push(event.event);
  }

  const wsUrls = new Set<string>();
  for (const event of wsEvents) {
    if (event.event.type === 'ws-open') wsUrls.add(event.event.url);
  }

  // Store WS handles for clock-coordinated delivery from replay loop
  const wsHandles = new Map<string, { ws: WebSocketRoute; pending: NetworkEvent[] }>();

  for (const url of wsUrls) {
    await page.routeWebSocket(new RegExp(escapeRegex(url)), (ws) => {
      const timeline = findWsTimeline(wsTimelines, url);
      if (!timeline) return;

      // Store handle — messages will be sent by the replay orchestrator
      // at the right clock time, not via setTimeout
      wsHandles.set(url, {
        ws,
        pending: timeline.filter(e => e.type === 'ws-receive'),
      });

      ws.onMessage((message) => {
        // Log client→server for debugging / verification
      });
    });
  }

  return wsHandles; // Replay loop uses these to send at clock-coordinated times
}

// In the replay orchestrator:
// Between interactions, check if any WS messages should fire at this clock time
// Call wsHandle.ws.send(data) then clock.runFor(delta) to advance
```

**SSE mocking** — Replace `EventSource` via `addInitScript()` since `route.fulfill()` cannot stream.

**Critical: Clock interaction** — Same issue as WebSocket: this runs in the browser where `setTimeout` is mocked by `clock.install()`. With clock paused, SSE events never fire. Solution: same as WS — the mock stores pending events and exposes a delivery function. The replay orchestrator triggers delivery from Node.js, coordinated with `clock.runFor()`.

```javascript
// Injected via page.addInitScript(script, sseRecordings)
(function(recordings) {
  const _OriginalEventSource = window.EventSource;

  // Registry of active SSE connections for clock-coordinated delivery
  window.__eyespy_sseInstances = window.__eyespy_sseInstances || [];

  class MockEventSource {
    constructor(url, init) {
      this.url = url;
      this.readyState = 0; // CONNECTING
      this._listeners = {};
      this._pendingEvents = [];
      this.onopen = null;
      this.onmessage = null;
      this.onerror = null;

      const recorded = recordings.find(r => url.includes(r.url));
      if (!recorded) {
        return new _OriginalEventSource(url, init);
      }

      // Store pending events for clock-coordinated delivery
      this._pendingEvents = (recorded.events || []).slice();

      // Register for external delivery
      window.__eyespy_sseInstances.push(this);

      // Open immediately (no setTimeout — clock may be paused)
      this.readyState = 1; // OPEN
      // Defer open event to next microtask (Promise-based, not timer-based)
      Promise.resolve().then(() => {
        if (this.onopen) this.onopen(new Event('open'));
        this._dispatch('open', new Event('open'));
      });
    }

    // Called by replay orchestrator via page.evaluate() at the right clock time
    _deliverNext() {
      if (this._pendingEvents.length === 0) return false;
      const evt = this._pendingEvents.shift();
      const messageEvent = new MessageEvent(evt.eventType || 'message', {
        data: evt.data,
        lastEventId: evt.lastEventId || '',
      });
      if (evt.eventType === 'message' && this.onmessage) {
        this.onmessage(messageEvent);
      }
      this._dispatch(evt.eventType || 'message', messageEvent);
      return this._pendingEvents.length > 0;
    }

    addEventListener(type, listener) {
      if (!this._listeners[type]) this._listeners[type] = [];
      this._listeners[type].push(listener);
    }
    removeEventListener(type, listener) {
      if (this._listeners[type]) {
        this._listeners[type] = this._listeners[type].filter(l => l !== listener);
      }
    }
    _dispatch(type, event) {
      (this._listeners[type] || []).forEach(l => l(event));
    }
    close() { this.readyState = 2; }
  }

  MockEventSource.CONNECTING = 0;
  MockEventSource.OPEN = 1;
  MockEventSource.CLOSED = 2;
  window.EventSource = MockEventSource;
})(arguments[0]);

// In the replay orchestrator (Node.js side):
// Between interactions, deliver SSE events at clock-coordinated times:
//   await page.evaluate(() => {
//     for (const sse of window.__eyespy_sseInstances || []) {
//       sse._deliverNext();
//     }
//   });
//   await page.clock.runFor(delta);
```

### Replay Engine: DOM Settle Detection

**Critical clock interaction**: When Playwright's `clock.pauseAt()` is active, ALL browser-side timers are frozen — `setTimeout`, `requestAnimationFrame`, `Date.now()` all return frozen values. A `page.evaluate()` that uses `setTimeout` for quiescence detection **will hang forever**. The settle detection MUST be orchestrated from Node.js using `page.waitForTimeout()` (which uses real wall-clock time) and non-blocking page queries.

```typescript
/**
 * Node.js orchestrated DOM settle detection.
 *
 * Instead of a single page.evaluate() with setTimeout/rAF (which hang when
 * clock is paused), we poll from Node.js using page.waitForTimeout() for
 * real-time delays and page.evaluate() for instant state checks.
 */
async function waitForDomSettle(page: Page, options: { timeout?: number; quiescence?: number } = {}) {
  const { timeout = 10000, quiescence = 300 } = options;
  const deadline = Date.now() + timeout;

  // 1. Install a MutationObserver that sets a flag on any DOM change.
  //    This runs in-page but doesn't use timers — just sets a boolean.
  await page.evaluate(() => {
    (window as any).__eyespy_domChanged = false;
    if ((window as any).__eyespy_observer) (window as any).__eyespy_observer.disconnect();
    const observer = new MutationObserver(() => {
      (window as any).__eyespy_domChanged = true;
    });
    observer.observe(document.body, {
      childList: true, subtree: true, attributes: true, characterData: true
    });
    (window as any).__eyespy_observer = observer;
  });

  // 2. Poll from Node.js: wait quiescence ms, then check if DOM changed
  let stableMs = 0;
  while (stableMs < quiescence && Date.now() < deadline) {
    // Real wall-clock delay (not affected by Playwright clock mock)
    await page.waitForTimeout(50);

    const changed = await page.evaluate(() => {
      const changed = (window as any).__eyespy_domChanged;
      (window as any).__eyespy_domChanged = false; // Reset flag
      return changed;
    });

    if (changed) {
      stableMs = 0; // DOM changed — reset quiescence counter
    } else {
      stableMs += 50;
    }
  }

  // 3. Check resource loading (instant queries, no timers)
  const resourcesReady = await page.evaluate(() => {
    // Fonts
    if (document.fonts.status !== 'loaded') return false;

    // Visible images
    const images = Array.from(document.querySelectorAll('img'));
    const visibleImages = images.filter(img => {
      const rect = img.getBoundingClientRect();
      return rect.top < window.innerHeight && rect.bottom > 0;
    });
    return visibleImages.every(img => img.complete);
  });

  // 4. If resources not ready, wait a bit more and re-check
  if (!resourcesReady && Date.now() < deadline) {
    await page.waitForTimeout(500); // Give fonts/images time to load
    // Don't loop — if still not ready after 500ms, proceed anyway
  }

  // 5. Cleanup observer
  await page.evaluate(() => {
    if ((window as any).__eyespy_observer) {
      (window as any).__eyespy_observer.disconnect();
      delete (window as any).__eyespy_observer;
      delete (window as any).__eyespy_domChanged;
    }
  });
}
```

### Replay Orchestrator (Main Loop)

The orchestrator ties together PRNG seeding, clock control, network mocking, interaction dispatch, WS/SSE delivery, DOM settle, and screenshot capture. This is the most complex component — getting the ordering wrong breaks determinism.

```typescript
/**
 * Replay a single session. Pure function of (session, seed) — each call is
 * independent (critical for smart retry). Returns screenshots + optional coverage.
 */
async function replaySession(
  browser: Browser,
  session: RecordedSession,
  seed: string,
  options: { collectCoverage: boolean; mode: 'mock' | 'live'; screenshotStrategy: string; targetUrl?: string; viewport?: { width: number; height: number }; allowLiveExternalEgress?: boolean; allowedOrigins?: string[]; originPolicies?: Record<string, 'mock' | 'live' | 'blocked'> },
): Promise<ReplayResult> {
  // 1. Fresh BrowserContext (clean state, no bleed between sessions/retries)
  const context = await browser.newContext({
    viewport: options.viewport ?? { width: 1280, height: 720 },
    deviceScaleFactor: 1,
    serviceWorkers: 'block',
  });

  // 2. Determinism setup — ORDER MATTERS
  //    addInitScript runs before page JS on every navigation, in registration order.
  await context.addInitScript(PRNG_SEED_SCRIPT, seed);
  await context.addInitScript(CLOCK_GAP_PATCHES); // document.timeline, MessageChannel
  await context.addInitScript(SSE_MOCK_SCRIPT, extractSseRecordings(session));

  // 3. Network mocking — register routes on context (survives navigations)
  //    In mock mode, ALL recorded origins are mocked (both same-origin API calls and
  //    third-party APIs like Stripe/S3). FIFO queue keys use RECORDED URLs, and route
  //    handlers intercept ALL origins that appear in the recording. URL remapping
  //    (originMap) translates recorded frontend origin → replay target origin so that
  //    same-origin API requests match correctly.
  const networkQueues = buildNetworkQueues(session);
  const effectiveAllowedOrigins = computeAllowedOrigins(session, options);
  await registerRoutes(context, networkQueues, options.mode, effectiveAllowedOrigins, options.allowLiveExternalEgress ?? false);

  // 4. Create page, install clock BEFORE any navigation
  const page = await context.newPage();
  const startTime = new Date(session.startedAt);
  // CRITICAL: Install clock 1s BEFORE target time to avoid Playwright #33926 race condition.
  // Between install() and pauseAt(), the clock runs. If both use the same time, pauseAt
  // can fail with "Cannot fast-forward to the past" when the clock advances past startTime
  // in the gap between the two CDP calls.
  await page.clock.install({ time: new Date(startTime.getTime() - 1000) });
  await page.clock.pauseAt(startTime);

  // 5. WebSocket mocking (requires page for routeWebSocket)
  const wsHandles = await mockWebSockets(page, session);

  // 6. Coverage (first attempt only — caller passes collectCoverage: false for retries)
  if (options.collectCoverage) {
    await page.coverage.startJSCoverage({ resetOnNavigation: false });
  }

  const screenshots = new Map<string, Buffer>();
  const healingLog: HealingEvent[] = [];
  let failedInteractions = 0;
  let totalInteractions = 0;
  let lastEventTimestamp = 0;

  // 6b. URL remapping (recorded origin → replay target origin)
  // The caller passes options.targetUrl (from --url flag). If different from session.url origin,
  // all app-origin URLs are remapped. Third-party origins unchanged.
  const remapUrl = createUrlRemapper(session.url, options.targetUrl ?? session.url);

  // 7. Navigate to starting URL (with origin remapping)
  const startUrl = remapUrl(session.url);
  try {
    await page.goto(startUrl, { waitUntil: 'commit', timeout: 30000 });
  } catch (e) {
    // Navigation timeout recovery
    const diag = await page.screenshot().catch(() => null);
    await context.close();
    return { status: 'replay-error', reason: `Navigation to ${startUrl} failed: ${e.message}`, diagnosticScreenshot: diag };
  }

  await waitForDomSettle(page);
  // Screenshot key uses event seq: nav@e{seq}, cap@e{seq}, final
  // Derive initial screenshot key from first navigate event's seq, or 'initial' if none exists
  const firstNav = session.events.find(e => e.type === 'navigate');
  const initialKey = firstNav ? `nav@e${firstNav.seq}` : 'initial';
  screenshots.set(initialKey, await captureStableScreenshot(page));

  // 8. Replay interactions in chronological order (sorted by seq)
  const interactions = session.events
    .filter(event => event.type !== 'network')
    .sort((a, b) => a.seq - b.seq);

  for (const event of interactions) {
    // Advance mocked clock to this event's time
    const delta = event.t_ms - lastEventTimestamp;
    if (delta > 0) {
      // Deliver any pending WS/SSE messages due during this time window
      await deliverPendingMessages(page, wsHandles, lastEventTimestamp, event.t_ms);
      await page.clock.runFor(delta);
    }
    lastEventTimestamp = event.t_ms;

    // Dispatch the interaction
    totalInteractions++;
    try {
      await dispatchInteraction(page, event, healingLog, remapUrl);
    } catch (e) {
      failedInteractions++;
      healingLog.push({ event, error: e.message, skipped: true });
      if (totalInteractions >= 5 && failedInteractions / totalInteractions > 0.3) {
        await context.close();
        return { status: 'unreplayable', reason: `>${Math.round(failedInteractions/totalInteractions*100)}% interactions failed`, healingLog };
      }
      continue; // Skip failed interaction
    }

    // Wait for DOM to settle
    await waitForDomSettle(page);

    // Tiered screenshot capture — key derived from event seq
    // Clock lifecycle: pause → screenshot → resume (normative invariant)
    if (shouldCaptureScreenshot(event, options.screenshotStrategy)) {
      await page.clock.pauseAt(new Date(startTime.getTime() + event.t_ms));
      const keyPrefix = event.type === 'navigate' ? 'nav' : 'cap';
      screenshots.set(`${keyPrefix}@e${event.seq}`, await captureStableScreenshot(page));
      await page.clock.resume(); // Resume before next interaction — prevents timer freeze during settle
    }
  }

  // 9. Final screenshot (always) — key is always 'final'
  // Clock lifecycle: pause → screenshot (no resume needed — session ends)
  await page.clock.pauseAt(new Date(startTime.getTime() + lastEventTimestamp));
  screenshots.set('final', await captureStableScreenshot(page));

  // 10. Coverage collection
  let coverage: CoverageBitmap | undefined;
  if (options.collectCoverage) {
    const raw = await page.coverage.stopJSCoverage();
    coverage = v8ToBitmap(raw, new URL(session.url).origin);
  }

  await context.close();
  return { status: 'success', screenshots, coverage, healingLog };
}

/**
 * Stabilization loop — take consecutive identical screenshots.
 * Absorbs remaining compositor/rendering timing variance.
 */
async function captureStableScreenshot(page: Page, maxAttempts = 5, timeoutMs = 3000): Promise<Buffer> {
  // animations: 'disabled' is a screenshot option — fast-forwards CSS animations to completion,
  // cancels infinite animations. Applied per-screenshot, not at browser/context level.
  const opts = { type: 'png' as const, animations: 'disabled' as const };
  let previous = await page.screenshot(opts);
  const deadline = Date.now() + timeoutMs;

  for (let i = 1; i < maxAttempts && Date.now() < deadline; i++) {
    await page.waitForTimeout(100); // Real time — not affected by clock mock
    const current = await page.screenshot(opts);
    if (Buffer.compare(previous, current) === 0) return current; // Stable
    previous = current;
  }
  return previous; // Best effort after max attempts
}

/**
 * Map recorded events to Playwright actions.
 * Auto-healing: try primary selector, then fallback cascade.
 */
async function dispatchInteraction(page: Page, event: RecordedEvent, healingLog: HealingEvent[], remapUrl: (url: string) => string): Promise<void> {
  switch (event.type) {
    case 'click': {
      const locator = await resolveSelector(page, event.selector, healingLog);
      await locator.click({ button: buttonName(event.button), modifiers: toModifiers(event.modifiers) });
      break;
    }
    case 'input': {
      const locator = await resolveSelector(page, event.selector, healingLog);
      await locator.fill(event.value);
      break;
    }
    case 'keydown':
      // Key events are NOT replayed by default — text entry is handled by 'input' via
      // locator.fill(). Replaying both would double-type. Only replay key events marked
      // with replay:true (non-text keys, shortcuts like Ctrl+S, Escape, Tab, arrows).
      if (event.replay) await page.keyboard.down(event.key);
      break;
    case 'keyup':
      if (event.replay) await page.keyboard.up(event.key);
      break;
    case 'scroll': {
      if (event.selector) {
        const locator = await resolveSelector(page, event.selector, healingLog);
        await locator.evaluate((el, { x, y }) => { el.scrollTo(x, y); }, { x: event.x, y: event.y });
      } else {
        await page.evaluate(({ x, y }) => window.scrollTo(x, y), { x: event.x, y: event.y });
      }
      break;
    }
    case 'navigate':
      // Navigate events are MARKERS, not replay actions. In SPAs, a prior click/submit
      // already triggered the route change; calling page.goto() would hard-navigate and
      // reset state. Instead, optionally wait for the expected URL to confirm the app
      // reached the right route, then capture a screenshot.
      try {
        await page.waitForURL(remapUrl(event.url), { timeout: 5000 });
      } catch {
        // URL didn't match — log warning but continue (app may use different URL format)
      }
      break;
    // ... dblclick, focus, blur, pointer, touch, drag, paste, copy, cut follow same pattern
  }
}

/**
 * Auto-healing selector resolution. Try primary, then fallback cascade.
 * Phase 1a: strategies 1-5. Phase 1b: full 9-strategy cascade.
 */
async function resolveSelector(page: Page, bundle: SelectorBundle, healingLog: HealingEvent[]): Promise<Locator> {
  // Strategy 1: Primary selector with short timeout
  try {
    const locator = page.locator(bundle.primary);
    await locator.waitFor({ state: 'visible', timeout: 1500 });
    return locator;
  } catch {}

  // Strategies 2-5: Fallback selectors stored in SelectorBundle
  for (const fallback of bundle.fallbacks) {
    try {
      const locator = page.locator(fallback);
      await locator.waitFor({ state: 'visible', timeout: 500 });
      healingLog.push({
        event: {} as RecordedEvent, // Caller fills this
        originalSelector: bundle.primary,
        healedSelector: fallback,
        strategy: classifyStrategy(fallback),
        confidence: strategyConfidence(fallback),
        skipped: false,
      });
      return locator;
    } catch {}
  }

  // Strategy 5: Text content + tag name (not stored in fallbacks — dynamic)
  if (bundle.fingerprint.text && bundle.fingerprint.tagName) {
    const textLocator = page.locator(
      `${bundle.fingerprint.tagName}:has-text("${bundle.fingerprint.text.substring(0, 30)}")`
    );
    try {
      await textLocator.waitFor({ state: 'visible', timeout: 500 });
      if (await textLocator.count() === 1) { // Strict single match
        healingLog.push({
          event: {} as RecordedEvent,
          originalSelector: bundle.primary,
          healedSelector: `text+tag: ${bundle.fingerprint.tagName}/${bundle.fingerprint.text.substring(0, 20)}`,
          strategy: 'text+tag',
          confidence: 0.75,
          skipped: false,
        });
        return textLocator;
      }
    } catch {}
  }

  // All strategies failed
  throw new Error(`Selector resolution failed for: ${bundle.primary}`);
}

function classifyStrategy(selector: string): string {
  if (selector.startsWith('[data-testid')) return 'testid';
  if (selector.includes('[role=') && selector.includes('[aria-label=')) return 'role+label';
  if (selector.includes('[role=')) return 'role+text';
  if (selector.startsWith('#')) return 'id';
  return 'css-path';
}

function strategyConfidence(selector: string): number {
  if (selector.startsWith('[data-testid')) return 0.95;
  if (selector.includes('[role=') && selector.includes('[aria-label=')) return 0.90;
  if (selector.includes('[role=')) return 0.85;
  return 0.70;
}

/**
 * Deliver WS/SSE messages that should arrive between lastTime and currentTime.
 * Messages are delivered immediately (not via setTimeout) — clock timing is
 * controlled by the orchestrator's clock.runFor() calls.
 */
async function deliverPendingMessages(
  page: Page,
  wsHandles: Map<string, { ws: WebSocketRoute; pending: NetworkEvent[] }>,
  fromTimestamp: number,
  toTimestamp: number,
): Promise<void> {
  // WebSocket messages (Node.js side — real timers, deliver immediately)
  for (const [url, handle] of wsHandles) {
    while (handle.pending.length > 0 && handle.pending[0].t_ms <= toTimestamp) {
      const msg = handle.pending.shift()!;
      handle.ws.send(msg.data);
    }
  }
  // SSE messages (browser side — deliver via page.evaluate)
  await page.evaluate((ts) => {
    for (const sse of (window as any).__eyespy_sseInstances || []) {
      while (sse._pendingEvents.length > 0 && sse._pendingEvents[0].t_ms <= ts) {
        sse._deliverNext();
      }
    }
  }, toTimestamp);
}

// URL remapping: replace recorded app origin with replay target origin.
// Initialized by orchestrator caller: remapUrl = createUrlRemapper(session.url, targetUrl);
// Third-party origins (Stripe, S3, etc.) pass through unchanged.
function createUrlRemapper(recordedBaseUrl: string, replayBaseUrl: string) {
  const recordedOrigin = new URL(recordedBaseUrl).origin;
  const replayOrigin = new URL(replayBaseUrl).origin;
  if (recordedOrigin === replayOrigin) return (url: string) => url; // No-op
  return (url: string) => {
    try {
      const parsed = new URL(url);
      if (parsed.origin === recordedOrigin) {
        return url.replace(recordedOrigin, replayOrigin);
      }
      return url; // Third-party — unchanged
    } catch { return url; }
  };
}
// Note: `remapUrl` is created inside `replaySession()` and passed as a parameter to
// `dispatchInteraction()`. The initial page.goto also uses it. Never access via closure.

// Build branch index from extracted branches for session skipping
function buildBranchIndex(branches: Array<{ scriptUrl: string; start: number; end: number }>): BranchIndex {
  const entries: Record<number, { scriptUrl: string; startOffset: number; endOffset: number }> = {};
  for (let i = 0; i < branches.length; i++) {
    entries[i] = { scriptUrl: branches[i].scriptUrl, startOffset: branches[i].start, endOffset: branches[i].end };
  }
  return { entries };
}

// Extract SSE recordings from session for addInitScript injection
function extractSseRecordings(session: RecordedSession): Array<{ url: string; events: NetworkEvent[] }> {
  const sseMap = new Map<string, { url: string; events: NetworkEvent[] }>();
  for (const event of session.events) {
    if (event.type !== 'network') continue;
    if (event.event.type === 'sse-open') {
      sseMap.set(event.event.sseId, { url: event.event.url, events: [] });
    } else if (event.event.type === 'sse-event') {
      const entry = sseMap.get(event.event.sseId);
      if (entry) entry.events.push(event.event);
    }
  }
  return Array.from(sseMap.values());
}

// Find WebSocket timeline for a given URL from pre-built timelines
function findWsTimeline(wsTimelines: Map<string, NetworkEvent[]>, url: string): NetworkEvent[] | undefined {
  for (const [wsId, events] of wsTimelines) {
    const openEvent = events.find(e => e.type === 'ws-open');
    if (openEvent && openEvent.url === url) return events;
  }
  return undefined;
}

// SHA-256 hash of a Buffer
function sha256(data: Buffer): string {
  return createHash('sha256').update(data).digest('hex');
}

// Utility: escape string for use in RegExp constructor
function escapeRegex(str: string): string { return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'); }

// Utility: map numeric button to Playwright button name
function buttonName(button: number): 'left' | 'right' | 'middle' {
  return button === 2 ? 'right' : button === 1 ? 'middle' : 'left';
}

// Utility: map our Modifiers type to Playwright's modifier keys
function toModifiers(m: Modifiers): Array<'Alt' | 'Control' | 'Meta' | 'Shift'> {
  const result: Array<'Alt' | 'Control' | 'Meta' | 'Shift'> = [];
  if (m.alt) result.push('Alt');
  if (m.ctrl) result.push('Control');
  if (m.meta) result.push('Meta');
  if (m.shift) result.push('Shift');
  return result;
}
```

**Diffing in-memory screenshots against baselines**: The orchestrator has `Map<string, Buffer>` (screenshots) but baselines are SHA-256 hashes in `baselines.json` pointing to blobs on disk. The bridge uses `sharp(Buffer)` directly (sharp accepts both file paths and Buffers):

```typescript
interface ScreenshotDiff {
  key: string;                   // e.g., "nav@e0", "cap@e5", "final"
  identical: boolean;
  exceedsThreshold: boolean;
  diffPixels?: number;
  diffRatio?: number;
  currentHash: string;
  baselineHash?: string;
  diffImageHash?: string;
}

type FinalResult =
  | ReplayResult & { diffs?: ScreenshotDiff[]; confirmedDiffs?: string[]; totalAttempts?: number };

async function diffScreenshots(
  screenshots: Map<string, Buffer>,
  baselines: BaselineManifest,
  sessionId: string,
  blobStore: BlobStore,
  diffOptions: { threshold?: number; maxDiffPixelRatio?: number } = {},
): Promise<ScreenshotDiff[]> {
  const results: ScreenshotDiff[] = [];
  for (const [key, buffer] of screenshots) {
    // Digest of PNG bytes — matches blob store and baseline manifest format
    const currentDigest = `sha256:${sha256(buffer)}`;
    const baselineDigest = baselines.getDigest(sessionId, key);

    if (!baselineDigest) {
      results.push({ key, identical: false, exceedsThreshold: false, currentHash: currentDigest, baselineHash: undefined });
      continue; // New screenshot — no baseline to compare against
    }

    if (currentDigest === baselineDigest) {
      results.push({ key, identical: true, exceedsThreshold: false, currentHash: currentDigest, baselineHash: baselineDigest });
      continue; // Digest match — skip pixelmatch
    }

    // Digest mismatch — run pixel comparison using Buffers directly (sharp accepts Buffer)
    // BlobStore.get() takes raw hex, so strip the sha256: prefix
    const baselineBuffer = await blobStore.get(baselineDigest.replace('sha256:', ''));
    const diff = await compareScreenshotsFromBuffers(baselineBuffer, buffer, diffOptions);
    results.push({ key, ...diff, currentHash, baselineHash });
  }
  return results;
}
```

**`compareScreenshots` Buffer variant**: The diff engine accepts both file paths and Buffers. `sharp()` accepts either, so the core comparison function works on raw data:

```typescript
async function compareScreenshotsFromBuffers(
  baseline: Buffer, current: Buffer, options: { threshold?: number; maxDiffPixelRatio?: number } = {}
): Promise<{ identical: boolean; exceedsThreshold: boolean; diffPixels: number; diffRatio: number; diffImageHash?: string }> {
  // sharp accepts Buffer directly — no temp files needed
  const [baseRaw, curRaw] = await Promise.all([
    sharp(baseline).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
    sharp(current).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
  ]);
  // ... same pixelmatch logic as compareScreenshots ...
}
```

**Smart retry orchestration** (wraps `replaySession`):
```typescript
async function replayWithRetry(browser: Browser, session: RecordedSession, seed: string, baselines: BaselineManifest): Promise<FinalResult> {
  // First attempt: collect coverage + screenshots
  const first = await replaySession(browser, session, seed, { collectCoverage: true, mode: 'mock', screenshotStrategy: 'tiered' });
  if (first.status !== 'success') return first;

  // Diff against baselines
  const diffs = await diffScreenshots(first.screenshots, baselines, session.id, blobStore);
  const diffedKeys = diffs.filter(d => d.exceedsThreshold).map(d => d.key);

  if (diffedKeys.length === 0) return { ...first, diffs }; // All pass — done

  // Retry: replay 2 more times, only compare the specific screenshots that diffed
  const retryResults: Map<string, Buffer>[] = [];
  for (let attempt = 0; attempt < 2; attempt++) {
    const retry = await replaySession(browser, session, seed, { collectCoverage: false, mode: 'mock', screenshotStrategy: 'tiered' });
    if (retry.status === 'success') retryResults.push(retry.screenshots);
  }

  // Majority vote: diff must appear in 2-of-3 attempts to count.
  // IMPORTANT: Confirm by pixel threshold, NOT digest comparison. PNG metadata/encoding
  // variance can cause digest mismatches even when the pixel diff is under threshold.
  const confirmedDiffs = diffedKeys.filter(key => {
    let diffCount = 1; // First attempt already showed diff (by pixel threshold)
    for (const screenshots of retryResults) {
      const shot = screenshots.get(key);
      if (shot) {
        const baselineDigest = baselines.getDigest(session.id, key);
        if (!baselineDigest) { diffCount++; continue; }
        const baselineBuffer = blobStore.get(baselineDigest.replace('sha256:', ''));
        const retryDiff = compareScreenshotsFromBuffers(baselineBuffer, shot, diffOptions);
        if (retryDiff.exceedsThreshold) diffCount++;
      }
    }
    return diffCount >= 2; // 2-of-3 must exceed pixel threshold
  });

  return { ...first, confirmedDiffs, totalAttempts: 1 + retryResults.length };
}
```

**Screenshot strategy decision function** (implements the tiered capture described above):
```typescript
function shouldCaptureScreenshot(event: RecordedEvent, strategy: string): boolean {
  if (strategy === 'every') return true;
  if (strategy === 'manual') return event.type === 'screenshot-marker';

  // 'tiered' (default):
  switch (event.type) {
    case 'navigate':        return true;  // Always capture after navigation
    case 'click':           return true;  // Capture with stabilization
    case 'dblclick':        return true;
    // Input: only capture on blur/submit (handled by next event), not mid-typing
    case 'input':           return false;
    case 'keydown':         return false;
    case 'keyup':           return false;
    case 'scroll':          return false;
    case 'resize':          return true;  // Layout change
    case 'pointerdown':     return false;
    case 'pointermove':     return false;
    case 'pointerup':       return false;
    case 'touchstart':      return false;
    case 'touchmove':       return false;
    case 'touchend':        return false;
    case 'focus':           return false;
    case 'blur':            return true;  // Capture after input sequence completes
    case 'paste':           return true;  // Content change
    case 'copy':            return false; // No visual change
    case 'cut':             return true;  // Content removal
    case 'dragstart':       return false; // Wait for drop
    case 'dragover':        return false;
    case 'dragend':         return false;
    case 'drop':            return true;  // Content change
    case 'change':          return true;  // Form value committed
    case 'submit':          return true;  // Form submission
    case 'wheel':           return false; // Scroll variant
    case 'touchcancel':     return false;
    case 'pointercancel':   return false;
    case 'screenshot-marker': return true;
    default:                return false;
  }
}
```

### Coverage: V8 Format and Bitmap Conversion

**Playwright coverage format** (from `page.coverage.stopJSCoverage()`):

Playwright's high-level API returns `JSCoverageEntry[]` directly — NOT the raw CDP `{ result: [...] }` wrapper. Each entry has `url`, `scriptId`, `source` (full script text), and `functions`.

```typescript
// Playwright's return type (NOT the raw CDP format)
interface JSCoverageEntry {
  url: string;
  scriptId: string;
  source?: string;              // Full script source text (when available)
  functions: Array<{
    functionName: string;
    ranges: Array<{
      startOffset: number;      // Byte offset in source
      endOffset: number;
      count: number;            // Execution count
    }>;
    isBlockCoverage: boolean;
  }>;
}
```

**Bitmap conversion** — Convert byte ranges to a compact bit vector for fast set operations.

```typescript
interface CoverageBitmap {
  sessionId: string;
  version: { gitSha: string; sourceContentHash: string }; // sourceContentHash = SHA-256 of V8 script sources
  totalBranches: number;
  coveredBranches: number;
  bitmap: Uint8Array;            // 1 bit per branch, packed
  branchIndex: BranchIndex;      // Maps branch ID → (scriptUrl, startOffset, endOffset)
}

function v8ToBitmap(coverage: JSCoverageEntry[], appOrigin: string): CoverageBitmap {
  const branches: Array<{ scriptUrl: string; start: number; end: number; covered: boolean }> = [];

  for (const script of coverage) {
    // Filter: only app scripts, not third-party/CDN
    if (!script.url.startsWith(appOrigin)) continue;

    for (const func of script.functions) {
      if (!func.isBlockCoverage) continue;
      // Skip first range (function itself) — sub-ranges are the branches
      for (let i = 1; i < func.ranges.length; i++) {
        const range = func.ranges[i];
        branches.push({
          scriptUrl: script.url,
          start: range.startOffset,
          end: range.endOffset,
          covered: range.count > 0,
        });
      }
    }
  }

  // Pack into bitmap
  const byteLength = Math.ceil(branches.length / 8);
  const bitmap = new Uint8Array(byteLength);
  for (let i = 0; i < branches.length; i++) {
    if (branches[i].covered) {
      bitmap[i >> 3] |= (1 << (i & 7));
    }
  }

  return {
    sessionId: '',
    version: { gitSha: '', sourceContentHash: '' },
    totalBranches: branches.length,
    coveredBranches: branches.filter(b => b.covered).length,
    bitmap,
    branchIndex: buildBranchIndex(branches),
  };
}
```

**Greedy set cover** — BitSet-optimized for performance. Intersection is just bitwise AND.

```typescript
function greedySetCover(
  bitmaps: CoverageBitmap[],
  targetRatio: number = 1.0
): { selected: string[]; coverageRatio: number } {
  // Build universe from union of all bitmaps
  const universeSize = bitmaps[0]?.totalBranches || 0;
  const uncovered = new Uint8Array(Math.ceil(universeSize / 8));
  // Set all bits in universe
  for (let i = 0; i < universeSize; i++) {
    for (const bm of bitmaps) {
      uncovered[i >> 3] |= bm.bitmap[i >> 3];
    }
  }
  const totalReachable = popcount(uncovered);
  const target = Math.ceil(totalReachable * targetRatio);

  const selected: string[] = [];
  let coveredCount = 0;
  const remaining = new Set(bitmaps.map((_, i) => i));

  while (coveredCount < target && remaining.size > 0) {
    let bestIdx = -1;
    let bestNew = 0;

    for (const idx of remaining) {
      // Count newly covered branches: popcount(bitmap AND uncovered)
      let newlyCovered = 0;
      for (let byte = 0; byte < uncovered.length; byte++) {
        newlyCovered += popcount8(bitmaps[idx].bitmap[byte] & uncovered[byte]);
      }
      if (newlyCovered > bestNew) {
        bestNew = newlyCovered;
        bestIdx = idx;
      }
    }

    if (bestIdx === -1 || bestNew === 0) break;

    selected.push(bitmaps[bestIdx].sessionId);
    // Remove covered branches from uncovered
    for (let byte = 0; byte < uncovered.length; byte++) {
      uncovered[byte] &= ~bitmaps[bestIdx].bitmap[byte];
    }
    coveredCount += bestNew;
    remaining.delete(bestIdx);
  }

  return { selected, coverageRatio: coveredCount / totalReachable };
}

// popcount for Uint8Array (sum of all set bits)
function popcount(bytes: Uint8Array): number {
  let total = 0;
  for (let i = 0; i < bytes.length; i++) total += popcount8(bytes[i]);
  return total;
}

// popcount for single byte
function popcount8(n: number): number {
  n = n - ((n >> 1) & 0x55);
  n = (n & 0x33) + ((n >> 2) & 0x33);
  return (n + (n >> 4)) & 0x0f;
}
```

### Coverage: Session Skipping (TurboSnap Equivalent)

**Source map dependency**: Session skipping requires mapping V8 script URLs (e.g., `http://localhost:3000/assets/index-a1b2c3.js`) back to local file paths (e.g., `src/components/App.tsx`). This requires source maps. **When source maps are unavailable** (stripped in production builds, not served by dev server): warn `"Source maps not found for N scripts. Session skipping disabled — replaying all sessions."` and fall back to replaying all sessions. This is safe (slower, not incorrect). Source map resolution runs once during `eyespy generate` and caches the mapping in `.eyespy/coverage/source-map-cache.json`.

```typescript
/**
 * Given a git diff and coverage bitmaps, determine which sessions
 * need to be replayed. Sessions whose coverage doesn't overlap
 * with changed files can be skipped entirely.
 *
 * Chromatic's TurboSnap traces Webpack module dependency graphs.
 * We use a simpler approach: source map resolution for URL → file mapping.
 * Falls back to replaying all sessions when source maps are unavailable.
 */
function getAffectedSessions(
  changedFiles: string[],       // From `git diff --name-only HEAD~1`
  bitmaps: CoverageBitmap[],
  sourceMap: Map<string, string> // scriptUrl → local file path
): { replay: string[]; skip: string[] } {
  // Map changed files to script URLs
  const changedScripts = new Set<string>();
  for (const file of changedFiles) {
    for (const [scriptUrl, localPath] of sourceMap) {
      if (localPath.endsWith(file) || file.endsWith(localPath)) {
        changedScripts.add(scriptUrl);
      }
    }
  }

  const replay: string[] = [];
  const skip: string[] = [];

  for (const bitmap of bitmaps) {
    // Check if any branch in this bitmap belongs to a changed script
    let overlaps = false;
    for (const [branchId, info] of Object.entries(bitmap.branchIndex.entries)) {
      if (changedScripts.has(info.scriptUrl)) {
        // Check if this specific branch is covered
        const idx = parseInt(branchId);
        if (bitmap.bitmap[idx >> 3] & (1 << (idx & 7))) {
          overlaps = true;
          break;
        }
      }
    }

    if (overlaps) {
      replay.push(bitmap.sessionId);
    } else {
      skip.push(bitmap.sessionId);
    }
  }

  return { replay, skip };
}
```

### Diff Engine: pixelmatch Integration

pixelmatch operates on raw RGBA pixel buffers. sharp converts PNG → raw RGBA.

```typescript
import pixelmatch from 'pixelmatch';
import sharp from 'sharp';
import crypto from 'crypto';

interface DiffResult {
  identical: boolean;            // Hash match — no diff needed
  diffPixels: number;
  diffRatio: number;             // diffPixels / totalPixels
  exceedsThreshold: boolean;
  baselineHash: string;
  currentHash: string;
  diffImageHash?: string;        // Hash in content-addressable store
}

async function compareScreenshots(
  baselinePath: string,
  currentPath: string,
  options: {
    threshold?: number;          // 0.0 - 1.0, default 0.1 (YIQ color distance)
    maxDiffPixelRatio?: number;  // e.g. 0.01 = 1% of pixels
    maxDiffPixels?: number;
    ignoreAntialiasing?: boolean; // Ignore anti-aliased pixel diffs, default true
    maskRegions?: Array<{ x: number; y: number; width: number; height: number }>;
  } = {}
): Promise<DiffResult> {
  // ignoreAntialiasing maps to pixelmatch's inverted `includeAA` option:
  // ignoreAntialiasing: true → includeAA: false (detect AA, exclude from diff count)
  const { threshold = 0.1, maxDiffPixelRatio = 0.005, ignoreAntialiasing = true } = options;
  const includeAA = !ignoreAntialiasing;

  // Content-hash short-circuit: hash PNG bytes (NOT decoded RGBA) for consistency
  // with blob store digests and baseline manifest. All digests are of encoded PNG bytes.
  const [baselinePng, currentPng] = await Promise.all([
    readFile(baselinePath),
    readFile(currentPath),
  ]);
  const baselineHash = crypto.createHash('sha256').update(baselinePng).digest('hex');
  const currentHash = crypto.createHash('sha256').update(currentPng).digest('hex');
  if (baselineHash === currentHash) {
    return { identical: true, diffPixels: 0, diffRatio: 0, exceedsThreshold: false, baselineHash, currentHash };
  }

  // Decode to raw RGBA for pixel comparison
  const [baselineBuffer, currentBuffer] = await Promise.all([
    sharp(baselinePng).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
    sharp(currentPng).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
  ]);

  const { width, height } = baselineBuffer.info;
  const totalPixels = width * height;

  // Apply region masks (zero out masked areas in both images)
  if (options.maskRegions) {
    for (const mask of options.maskRegions) {
      zeroRegion(baselineBuffer.data, width, mask);
      zeroRegion(currentBuffer.data, width, mask);
    }
  }

  // Run pixelmatch
  const diffOutput = new Uint8Array(width * height * 4);
  const diffPixels = pixelmatch(
    new Uint8ClampedArray(baselineBuffer.data.buffer),
    new Uint8ClampedArray(currentBuffer.data.buffer),
    diffOutput,
    width,
    height,
    { threshold, includeAA, diffColor: [255, 0, 0], alpha: 0.3 }
  );

  const diffRatio = diffPixels / totalPixels;
  const maxPixels = options.maxDiffPixels ?? Math.ceil(totalPixels * maxDiffPixelRatio);
  const exceedsThreshold = diffPixels > maxPixels;

  // Store diff image if there are differences
  let diffImageHash: string | undefined;
  if (diffPixels > 0) {
    const diffPng = await sharp(Buffer.from(diffOutput.buffer), {
      raw: { width, height, channels: 4 }
    }).png().toBuffer();
    diffImageHash = await blobStore.put(diffPng);
  }

  return { identical: false, diffPixels, diffRatio, exceedsThreshold, baselineHash, currentHash, diffImageHash };
}

function zeroRegion(data: Buffer, imageWidth: number, mask: { x: number; y: number; width: number; height: number }) {
  for (let row = mask.y; row < mask.y + mask.height; row++) {
    for (let col = mask.x; col < mask.x + mask.width; col++) {
      const offset = (row * imageWidth + col) * 4;
      data[offset] = data[offset + 1] = data[offset + 2] = 0;
      data[offset + 3] = 255;
    }
  }
}
```

### Storage: Content-Addressable Blob Store

Two-level directory sharding to avoid filesystem inode limits.

```typescript
import { createHash } from 'crypto';
import { mkdir, writeFile, readFile, rename, stat, readdir, unlink, rmdir } from 'fs/promises';
import { join } from 'path';

class BlobStore {
  constructor(private rootDir: string) {}

  /** Store data, return SHA-256 hash */
  async put(data: Buffer): Promise<string> {
    const hash = createHash('sha256').update(data).digest('hex');
    const dir = join(this.rootDir, hash.substring(0, 2), hash.substring(2, 4));
    const filePath = join(dir, hash);

    // Skip if already exists (content-addressable = immutable)
    try {
      await stat(filePath);
      return hash; // Already stored
    } catch {}

    await mkdir(dir, { recursive: true });

    // Atomic write: write to tmp, then rename
    const tmpPath = `${filePath}.${process.pid}.${Date.now()}.tmp`;
    await writeFile(tmpPath, data);
    await rename(tmpPath, filePath);

    return hash;
  }

  /** Retrieve data by hash */
  async get(hash: string): Promise<Buffer> {
    const filePath = join(this.rootDir, hash.substring(0, 2), hash.substring(2, 4), hash);
    return readFile(filePath);
  }

  /** Check if hash exists */
  async has(hash: string): Promise<boolean> {
    try {
      await stat(join(this.rootDir, hash.substring(0, 2), hash.substring(2, 4), hash));
      return true;
    } catch { return false; }
  }

  /** Garbage collect: remove blobs not referenced by any session/run */
  async gc(referencedHashes: Set<string>): Promise<{ deleted: number; freedBytes: number }> {
    let deleted = 0, freedBytes = 0;
    const l1Dirs = await readdir(this.rootDir);
    for (const l1 of l1Dirs) {
      const l2Dirs = await readdir(join(this.rootDir, l1)).catch(() => []);
      for (const l2 of l2Dirs as string[]) {
        const files = await readdir(join(this.rootDir, l1, l2)).catch(() => []);
        for (const file of files as string[]) {
          if (!referencedHashes.has(file)) {
            const path = join(this.rootDir, l1, l2, file);
            const info = await stat(path);
            freedBytes += info.size;
            await unlink(path);
            deleted++;
          }
        }
      }
    }
    return { deleted, freedBytes };
  }
}
```

### Lessons from Existing Projects

| Project | Key Lesson for Eyespy |
|---------|------------------------|
| **OpenReplay** | Direct `window.fetch` replacement (not Proxy). `Response.clone()` before consuming. Worker thread for heavy processing. |
| **Sentry** | 10K mutation/10s throttle prevents runaway recording. Breadcrumb model for event buffering. |
| **rrweb** | Numeric node IDs (not CSS selectors) for DOM mirroring. Wrong model for our replay use case — we need selectors because we replay in a different browser context. |
| **Highlight.io** | WebSocket interception via Proxy constructor. Batched transport with circuit breaker. Session window limit (4 hours). |
| **PostHog** | Privacy-first: opt-in recording, mask all inputs by default, `data-ph-no-capture` attribute. |
| **Lost Pixel** | Baseline directory per test with sequential numbering (`001-*.png`). Update command that moves current → baseline. |
| **Argos CI** | Dual-threshold diffing: lenient pass first (skip clearly identical), strict pass on remainder. Reduces compute 60-80%. |
| **Chromatic (TurboSnap)** | Traces Webpack dependency graph to skip unaffected stories. Our source-map-based approach is simpler (no bundler plugin needed) but requires source maps to be served. |
| **Playwright** | `addInitScript()` = CDP `Page.addScriptToEvaluateOnNewDocument` — runs before ANY page JS. `animations: 'disabled'` is protocol-level, not CSS injection. `toHaveScreenshot()` waits for consecutive identical screenshots — could borrow for settle detection. |
| **v8-to-istanbul** | Coverage conversion: V8 byte ranges → Istanbul function/branch/statement maps. We skip Istanbul format entirely — go straight to bitmaps. |
| **pixelmatch** | Requires raw RGBA buffers (sharp for conversion). YIQ color space for perceptual accuracy. Anti-aliasing detection reduces false positives by ~30%. |

### CLI Architecture & CI Integration

**Commander.js subcommand pattern** — Each command in its own file, composed via `addCommand()`:

```typescript
// src/cli/index.ts
const program = new Command();
program
  .name('eyespy')
  .version(VERSION)
  .option('--config <path>', 'Config file path', '.eyespy/config.json')
  .option('--json', 'Force JSON output (auto-detected when piped)')
  .option('--no-color', 'Disable colors')
  .option('-v, --verbose', 'Verbose logging');

program.addCommand(makeRecordCommand());
program.addCommand(makeReplayCommand());
program.addCommand(makeDiffCommand());
// ... etc

// Each command accesses global options via cmd.optsWithGlobals()
```

**JSON vs TTY detection** — `process.stdout.isTTY` is `undefined` (not `false`) when piped. Always use `!process.stdout.isTTY`:

```typescript
function shouldOutputJson(opts: { json?: boolean }): boolean {
  if (opts.json) return true;
  if (!process.stdout.isTTY) return true; // Piped
  return false;
}
```

**Progress bars** — `cli-progress` for deterministic operations (diffing N screenshots), `ora` spinners for indeterminate ones (launching browser). Both write to stderr, not stdout. Suppress when `--json` or piped.

**Exit codes**:

| Code | Meaning | When |
|------|---------|------|
| 0 | Pass | All screenshots match baselines |
| 1 | Diff found | Visual differences above threshold |
| 2 | Error | Browser crash, config error, timeout |

Matches pytest's 0/1/2 convention. GitHub Actions treats 1 and 2 identically as failures — use `continue-on-error: true` + manual exit code check to distinguish.

**JUnit XML** — Minimum viable structure for GitHub Actions (`dorny/test-reporter`), GitLab, CircleCI:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="Eyespy" tests="5" failures="2" errors="0" time="12.34">
  <testsuite name="homepage" tests="3" failures="1" errors="0" time="5.67">
    <testcase name="hero-section" classname="homepage" time="1.23"/>
    <testcase name="navigation" classname="homepage" time="2.01">
      <failure message="Pixel diff 2.3% exceeds threshold" type="VisualDiff">
Baseline hash: sha256:a1f2b3c...
Diff pixels: 1,847 (2.3%)
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

Use `junit-report-builder` npm package. GitLab requires `<testsuites>` wrapper (won't parse bare `<testsuite>`).

**HTML report** — Standalone single-file with all images base64-inlined (reg-cli pattern):

1. Template HTML with `{{css}}`, `{{nonce}}`, `{{js}}`, `{{data}}` placeholders
2. Bundled viewer app (vanilla JS or Preact, ~5KB) injected as `<script nonce="{{nonce}}">`
3. Report data as `<script type="application/json" id="eyespy-data">{{data}}</script>` (non-executable)
4. Images as `data:image/png;base64,...` data URLs
5. Nonce generated as 16 random bytes (base64) at report build time

Four viewer modes: side-by-side, slider (CSS `clip-path`), blend (opacity crossfade), toggle (click to switch).

**Security**: All session-derived strings (session IDs, file names, error messages) MUST be HTML-escaped before injection into the report template. Report data MUST be embedded as `<script type="application/json" id="eyespy-data">...</script>` (non-executable JSON, not a JS assignment). The report MUST use nonce-based CSP: `<meta http-equiv="Content-Security-Policy" content="default-src 'none'; img-src data:; style-src 'unsafe-inline'; script-src 'nonce-{RANDOM}'; base-uri 'none'; frame-ancestors 'none'">`. Only the viewer script tag gets the nonce. This means even if HTML escaping is missed somewhere, injected scripts cannot execute because they lack the nonce.

**Size warning**: Base64 inflates images ~33%. 50 screenshots at 200KB each = 13MB HTML. For suites over 30 screenshots, default to folder-based report with separate image files, standalone HTML as opt-in (`--inline-report`).

**GitHub Actions workflow**:

```yaml
jobs:
  visual-test:
    runs-on: ubuntu-latest
    container:
      image: mcr.microsoft.com/playwright:v1.50.0-noble
      options: --ipc=host
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with: { node-version: 'lts/*' }
      - run: npm ci
      - name: Run Eyespy
        run: npx eyespy ci --junit results.xml --html report.html
        # Auto-pulls missing baseline blobs from remote storage if configured.
        # Set EYESPY_REMOTE_BUCKET and AWS credentials for S3/R2 backends.
        env:
          EYESPY_REMOTE_BUCKET: ${{ vars.EYESPY_REMOTE_BUCKET }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: eyespy-report
          path: |
            report.html
            .eyespy/runs/latest/diffs/
          retention-days: 30
      - uses: dorny/test-reporter@v1
        if: ${{ !cancelled() }}
        with:
          name: Eyespy Results
          path: results.xml
          reporter: java-junit
```

`--ipc=host` is critical — Docker's default 64MB `/dev/shm` crashes Chromium. Alternative: `--shm-size=1gb`. Use `if: ${{ !cancelled() }}` (not `if: always()`) for artifact upload.

**PR comment workflow (Phase 1a — no server needed)**:

Even without the Phase 2 server, teams need visual diff feedback in PRs. A lightweight GitHub Action step posts a PR comment with results:

```yaml
      - name: Comment PR with Eyespy results
        if: ${{ github.event_name == 'pull_request' && !cancelled() }}
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const result = JSON.parse(fs.readFileSync('.eyespy/runs/latest/summary.json', 'utf8'));
            const total = result.sessions.length;
            const passed = result.sessions.filter(s => s.status === 'pass').length;
            const failed = result.sessions.filter(s => s.status === 'diff').length;
            const errors = result.sessions.filter(s => s.status === 'error').length;

            const icon = failed > 0 ? '🔴' : '🟢';
            const body = [
              `## ${icon} Eyespy Visual Regression: ${passed}/${total} passed`,
              failed > 0 ? `**${failed} sessions with visual diffs** — [View Report](${process.env.REPORT_URL})` : '',
              errors > 0 ? `⚠️ ${errors} replay errors` : '',
              '',
              '| Session | Status | Diff |',
              '|---------|--------|------|',
              ...result.sessions.map(s =>
                `| ${s.id.substring(0,8)} | ${s.status === 'pass' ? '✅' : s.status === 'diff' ? '❌' : '⚠️'} | ${s.diffCount || 0} screenshots |`
              ),
              '',
              `📦 [Download full report](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`
            ].filter(Boolean).join('\n');

            // Update existing comment or create new one
            const { data: comments } = await github.rest.issues.listComments({
              ...context.repo, issue_number: context.issue.number
            });
            const existing = comments.find(c => c.body.includes('Eyespy Visual Regression'));
            if (existing) {
              await github.rest.issues.updateComment({ ...context.repo, comment_id: existing.id, body });
            } else {
              await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });
            }
```

This provides immediate feedback in PRs without any server infrastructure. The full HTML report is available as a GitHub Actions artifact.

### Build Toolchain & SDK Bundling

**esbuild for recorder-sdk** — IIFE format, ES2020 target (NOT ES5):

```javascript
// packages/recorder-sdk/build.mjs
import * as esbuild from 'esbuild';
import { gzipSync } from 'zlib';
import { readFileSync } from 'fs';

const result = await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  format: 'iife',
  globalName: 'Eyespy',
  outfile: 'dist/eyespy.min.js',
  platform: 'browser',
  target: ['es2020'],           // Not ES5 — see rationale below
  minify: true,
  treeShaking: true,
  metafile: true,
  define: {
    'process.env.NODE_ENV': '"production"',
    '__DEBUG__': 'false',
  },
  pure: ['console.log', 'console.debug'],
});

// Size budget enforcement
const output = readFileSync('dist/eyespy.min.js');
const gzipped = gzipSync(output);
const sizeKB = (gzipped.length / 1024).toFixed(2);
console.log(`Bundle: ${(output.length / 1024).toFixed(2)}KB min / ${sizeKB}KB gz`);
if (gzipped.length > 18 * 1024) {
  console.error(`EXCEEDS 18KB BUDGET! Current: ${sizeKB}KB`);
  process.exit(1);
}
```

**Why ES2020, not ES5**: ES5 produces 15-30% larger bundles (must lower arrow functions, const/let, template literals, optional chaining, etc). ES2020 covers Chrome 80+, Firefox 80+, Safari 14+, Edge 80+ — sufficient for a developer tool. Proxy can't be polyfilled anyway, so ES5 wouldn't help.

**fflate size impact**: Full library = ~31KB min / ~11.5KB gz. Gzip-only subset (what we use) = ~8-10KB min / ~3KB gz. Import only what's needed: `import { gzipSync } from 'fflate'` (tree-shakeable) not `import * as fflate from 'fflate'`.

Budget breakdown: 18KB gz total = ~3KB fflate + ~15KB recording logic. Tight but achievable — rrweb core is 40-50KB min but our scope is narrower.

**Detailed size estimate** (gzipped):
| Component | Estimate | Notes |
|-----------|----------|-------|
| fflate (gzip only) | ~3KB | Tree-shaked import |
| Fetch interception | ~1KB | Monkey-patch + clone + record |
| XHR interception | ~1KB | Prototype patch pattern |
| WS/SSE interception | ~1.5KB | Proxy constructors |
| Event capture (15 types) | ~2.5KB | Listeners + handlers |
| Selector generation | ~1.5KB | 4-strategy generator |
| Transport (batch + retry) | ~2KB | Fetch + beacon + circuit breaker |
| Privacy controls | ~1KB | Header strip + field masking |
| Session lifecycle | ~0.5KB | Init + flush + metadata |
| Compression bridge | ~0.5KB | CompressionStream detection |
| **Total** | **~14.5KB** | **3.5KB buffer** |

If budget is exceeded: first cut is WS/SSE interception (Phase 1b only), saving ~1.5KB. Second cut: simplify selector generation to `data-testid` + `id` + CSS path only (drop role+aria), saving ~0.5KB.

**size-limit for CI** (Sentry's approach):

```javascript
// .size-limit.js
module.exports = [
  {
    name: 'Eyespy Recorder SDK (gzipped)',
    path: 'packages/recorder-sdk/dist/eyespy.min.js',
    limit: '18 KB',
    gzip: true,
  },
];
```

Use `andresz1/size-limit-action` GitHub Action to post size comparison on PRs.

**tsup for Node packages** (replay-engine, diff-engine, test-generator, shared):

```typescript
// packages/replay-engine/tsup.config.ts
export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],
  dts: true,
  sourcemap: true,
  clean: true,
  external: ['playwright'],    // Never bundle Playwright
  target: 'node18',
});
```

CLI uses CJS only, no .d.ts: `format: ['cjs'], dts: false`.

**package.json exports** for dual format:
```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

**Turborepo config**:

```json
{
  "$schema": "https://turborepo.dev/schema.json",
  "globalDependencies": ["tsconfig.base.json"],
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"], "outputs": ["coverage/**"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "size-check": { "dependsOn": ["build"], "outputs": [] }
  }
}
```

`^build` = "build my dependencies first" — ensures `shared` builds before everything else. `lint` has no deps, runs in parallel with everything.

**Biome config** (replaces ESLint + Prettier):

```json
{
  "formatter": { "indentStyle": "space", "indentWidth": 2, "lineWidth": 100 },
  "linter": {
    "rules": {
      "recommended": true,
      "correctness": { "noUnusedVariables": "error", "noUnusedImports": "error" }
    }
  },
  "assist": { "actions": { "source": { "organizeImports": "on" } } },
  "javascript": { "formatter": { "quoteStyle": "single", "semicolons": "always" } }
}
```

Use `biome ci .` in CI (read-only, proper exit codes), `biome check --apply .` in dev.

### Playwright Docker & Determinism

**Official images**: `mcr.microsoft.com/playwright:v1.50.0-noble` (Ubuntu 24.04 LTS). ~2GB, includes Chromium + Firefox + WebKit + Node.js + system fonts. **Always pin exact version matching your `@playwright/test` dependency.**

**Pre-installed fonts**: Liberation (Arial/Times compatible), Noto Color Emoji, IPA Gothic (Japanese), WQY Zenhei (Chinese), FreeFonts, and font infrastructure (fontconfig, freetype). If your app uses web fonts via `@font-face`, they're downloaded at runtime and deterministic. System font fallbacks differ from macOS — `Helvetica` falls back to `Liberation Sans`.

**Chromium flags for determinism** — all four are needed:

| Flag | What it eliminates |
|------|-------------------|
| `--font-render-hinting=none` | Hinting engine differences (glyph shapes) |
| `--disable-font-subpixel-positioning` | Sub-pixel placement differences (GPU-dependent) |
| `--disable-lcd-text` | LCD subpixel rendering (display-dependent RGB/BGR) |
| `--disable-skia-runtime-opts` | CPU-specific SIMD rasterization differences |

Combined with `--disable-gpu` and `--force-color-profile=srgb` from the spec. Text appears "softer" with all flags — acceptable for regression testing.

**`animations: 'disabled'`** — Per-screenshot option, protocol-level (NOT CSS injection). Finite animations fast-forwarded to completion state. Infinite animations cancelled and reset. Elements end up in their "after" state, not "before". Must be passed on every `page.screenshot()` call.

**Screenshot stability mechanism** (`toHaveScreenshot()` internals — we borrow this for our settle detection):

1. Take screenshot A
2. Take screenshot B
3. If pixel-identical → stable, use B
4. If different → discard A, set A = B, take new B, repeat
5. Poll intervals: `[0, 100, 250, 500]` then `1000ms` each
6. Default timeout: 5000ms

A page with a ticking clock never stabilizes. Our PRNG seeding + scoped clock handles this.

**`addInitScript()` timing guarantees**:
- Runs BEFORE any `<script>` tags (inline or external), deferred scripts, module scripts, DOMContentLoaded
- Uses CDP `Page.addScriptToEvaluateOnNewDocument` internally
- **Survives navigations** — re-evaluated on every page load, including link clicks and form submissions
- `browserContext.addInitScript()` applies to ALL pages including popups; `page.addInitScript()` applies to one page only
- **Ordering**: Scripts at the SAME level (e.g., multiple `context.addInitScript()` calls) execute in registration order. Interleaving between context-level and page-level scripts is NOT guaranteed. Since Eyespy uses only `context.addInitScript()`, our PRNG → clock patches → SSE mock ordering is preserved.
- Runs in page JS context (not Node.js), cannot use `require()`

**CDP coverage + route interception coexistence**: No documented conflicts. They operate at different layers:
- `page.route()` / `browserContext.route()` → network layer (before responses reach renderer)
- `Profiler.startPreciseCoverage` → V8 engine layer (profiling executed JS)

Both can be used simultaneously. Playwright's own test suite validates this.

**Using built-in coverage API** (simpler than raw CDP for our needs):

```typescript
await page.coverage.startJSCoverage({
  resetOnNavigation: false,    // Accumulate across navigations
  reportAnonymousScripts: false,
});
// ... replay session ...
const coverage = await page.coverage.stopJSCoverage();
// coverage: [{ url, scriptId, source, functions: [{ functionName, ranges }] }]
```

Set `resetOnNavigation: false` to accumulate coverage across SPA navigations within a session.

**Variable fonts warning**: Inter Variable and similar variable fonts have known rendering differences in Docker (Playwright #29596). Prefer static font files for visual testing.

### Playwright Version Pinning Strategy

**The problem**: Playwright releases monthly. Each release bundles a NEW Chromium version. New Chromium versions change rendering behavior — subpixel rounding, font metrics, compositing algorithms, GPU rasterization. This means **upgrading Playwright can break ALL existing baselines**, even when the app hasn't changed. For visual regression testing, this is catastrophic — users see 100% of screenshots fail and lose trust in the tool.

**Pinning requirements**:
1. Pin `@playwright/test` to an EXACT version in `package.json` (e.g., `"1.50.0"`, NOT `"^1.50.0"`)
2. Pin the Docker image to the matching version (`mcr.microsoft.com/playwright:v1.50.0-noble`)
3. Store the Playwright version in `.eyespy/config.json` for validation at runtime
4. On `eyespy ci`: compare the installed Playwright version against `config.json` — warn if mismatched

**Upgrade workflow** (when teams decide to upgrade Playwright):
1. `npm install @playwright/test@latest` + update Docker image tag
2. Run `eyespy ci --url <url>` — all baselines will likely fail (Chromium rendering changed)
3. Review diffs to confirm they're rendering-only (not real regressions): `eyespy diff --report`
4. `eyespy approve` to accept new baselines with the new Chromium renderer
5. Commit updated `baselines.json` + `package.json` + `config.json` together
6. This is a known "baseline reset" — document it in the commit message

**Risk**: Teams may fear upgrading and stay on stale Playwright versions, missing security patches and features. Mitigation: Eyespy release notes should document which Playwright versions are tested and whether a baseline reset is expected.

**Minimum Playwright version**: 1.48 (required for `clock.install()` with #31924 fix, `routeWebSocket()`, and `animations: 'disabled'` as screenshot option).

---

## Build Sequence

### Phase 0: Proof of Concept (2-3 days, 1 developer)

**Goal**: Validate that deterministic replay produces pixel-identical screenshots across 5+ app archetypes that stress different aspects of the determinism stack.

The PoC is a single test harness (`proof-of-concept.ts`, ~500 lines) that runs a determinism matrix. Each app archetype targets a specific risk. If ANY archetype produces non-identical screenshots across 10 runs, we stop and investigate before committing to the architecture.

**Shared setup** (all archetypes):
1. Launches Playwright with the 6 individual compositor flags (NOT `--deterministic-mode` — see Browser flags section). Plus `--font-render-hinting=none`, `--disable-gpu`, `--force-color-profile=srgb`, etc.
2. Injects PRNG seeding via `addInitScript()` (Math.random, crypto.randomUUID, crypto.getRandomValues)
3. Installs `clock.install()` with fixed time + patches for `document.timeline.currentTime` and `MessageChannel`
4. Takes screenshots with Node.js orchestrated polling (waitForDomSettle) + stabilization loop
5. Runs 10 times in Docker, compares with pixelmatch (0 diff pixels required)

**App archetypes** (each targets a different determinism risk):

| # | App | What it stresses | Risk being validated |
|---|-----|-----------------|---------------------|
| A | **TodoMVC React 18** | React concurrent scheduler (MessageChannel), `useState`/`useReducer` batching | MessageChannel mock routes through `setTimeout(fn,0)` — does React 18 still render deterministically when its scheduler can't distinguish MessageChannel priority from setTimeout priority? |
| B | **CSS-in-JS app** (Emotion + styled-components) | Runtime CSS class name generation, style injection order | Emotion uses `murmurhash2` on style content — deterministic IF same styles injected in same order. Styled-components uses counter-based class names — deterministic IF component tree evaluates in same order. PRNG seeding doesn't affect these (they don't use Math.random), but concurrent rendering could change evaluation order. |
| C | **MUI DataGrid** (Material UI) | Complex component library with internal animations, transitions, portals, Popper.js positioning, intersection observers | Component libraries often use `requestAnimationFrame`, `setTimeout` for positioning, `ResizeObserver` for layout. Tests that the anti-flake stack + clock mocking handles complex real-world UI libraries. |
| D | **Date picker** (react-datepicker or Day.js calendar) | `Date`, `Intl.DateTimeFormat`, locale-dependent formatting | Validates that `clock.install()` + locale determinism produces identical calendar renders. Many date libraries cache `Date.now()` at import time — PRNG/clock seeding must run before the library imports. |
| E | **Randomness-heavy page** | `crypto.randomUUID()`, `Math.random()`, animated CSS, CSS transitions, `Date.now()` display, `performance.now()` measurements | Kitchen sink test for the PRNG + clock + animation stack. Includes both `setTimeout`-based animation and CSS `@keyframes` animation. |
| F | **Next.js 14 app router** (optional, if setup time < 1 hour) | SSR hydration, React Server Components, streaming HTML, client components with `useId()` | `useId()` generates deterministic IDs based on component tree position — should be stable. Streaming HTML means the page builds incrementally — DOM settle detection must handle this. |

**Test procedure per archetype**:
1. Navigate to the app
2. Perform 3-5 hardcoded interactions (add todo, open date picker, sort grid, etc.)
3. Take screenshot after each interaction (with DOM settle + stabilization)
4. Repeat 10 times with same seed in Docker
5. pixelmatch all 10 runs — 0 diff pixels for ALL screenshots

**Exit criteria**: 10/10 runs produce pixel-identical screenshots in the Playwright Docker image for ALL archetypes A-E. Archetype F is stretch goal.

**Failure protocol**: If an archetype fails:
1. Log which screenshot and which pixel coordinates differ
2. Save diff images for visual inspection
3. Bisect: disable determinism features one-by-one to isolate the source
4. If the failure is in a specific library (e.g., MUI), determine if it's fixable (add another mock) or fundamental (need different approach)
5. **Do not proceed to Phase 1a** with unresolved failures — each archetype validates a real-world scenario that users will encounter

**Risks being validated** (any failure = stop and investigate before proceeding):
- Deterministic replay with Playwright's stack (no Chromium fork)
- `clock.install()` + `MessageChannel` patch works with React 18 scheduler
- CSS-in-JS class generation is deterministic (same styles → same class names across runs)
- Complex component libraries (MUI, date pickers) render identically under the anti-flake stack
- Node.js orchestrated polling works with paused clock
- Individual compositor flags (without `--enable-begin-frame-control`) produce stable screenshots
- PRNG seeding catches `crypto.randomUUID()` / `crypto.getRandomValues()` before any page JS
- SSR hydration + streaming HTML settles deterministically (if archetype F included)

---

### Phase 1a: MVP (3-4 weeks, 1-2 developers)

**Goal**: `npx eyespy record --url <url>` → `npx eyespy ci --url <url>` → HTML diff report.

**Week 1: Foundation**
- Monorepo scaffolding (pnpm workspaces, Turborepo, Biome, Vitest)
- `@eyespy/shared`: Frozen TypeScript interfaces (`RecordedSession` with `formatVersion`, `RecordedEvent`, `SelectorBundle`, `DiffResult`, `RunResult`, config schema), seeded PRNG (xoshiro128** for browser injection, xoshiro256** for Node), content-addressable blob store (~100 lines)
- `.eyespy/` directory structure, `eyespy init` auto-creation (config.json + baselines.json + .gitignore updates)

**Week 2: Replay Engine (CRITICAL PATH)**
- Playwright launcher with 6 individual compositor flags + font/GPU/color flags + `animations: 'disabled'` (NOT `--deterministic-mode`)
- PRNG injection via `context.addInitScript()` (Math.random, crypto.randomUUID, crypto.getRandomValues)
- Full `clock.install()` with `pauseAt`/`runFor`/`resume` lifecycle (Playwright >= 1.48)
- Clock gap patches via `addInitScript()`: `document.timeline.currentTime` (#38951), `MessageChannel` timing
- HTTP network mocking (targeted per-origin routes via `browserContext.route()`, FIFO queue by method+fullUrl)
- **ResourceType-based mock routing**: mock fetch/XHR, allow document/script/stylesheet/image/font through, abort unrecorded fetch/XHR in mock mode
- Fresh browser context per session (no localStorage/IndexedDB/cookies bleed)
- DOM settle detection (Node.js orchestrated polling: MutationObserver flag + `page.waitForTimeout()` real-time polling + font loading + img.complete, 300ms quiescence)
- Tiered screenshot capture with stabilization loop (two-consecutive-identical)
- Smart retry (1x replay, re-confirm diffs 2x, majority vote — default ON)
- Basic auto-healing (strategies 1-5: primary retry, testid, role+label, role+text, text+tag)
- Navigation timeout recovery (diagnostic screenshot, log pending requests, mark as error)
- Hard timeouts: 120s/session, 30s/navigation
- **Determinism test**: 10-run identical screenshot assertion in Docker

**Week 3: Recorder + Diff Engine (parallel tracks)**
- *Playwright Recorder*: Page-level event capture via `addInitScript()` — inject capture-phase DOM event listeners (`click`, `input`, `keydown`, etc.) that fire before any app code. `page.exposeBinding()` sends events back to Node.js. Network via `page.on('request'/'response')` with async `response.body()` capture. Selector generation via in-page `generateSelector()` function (data-testid > id > role+aria > CSS path). Recording overlay (floating "Stop" button) excluded from selectors/screenshots. Save as `RecordedSession` JSON.
- *Diff engine*: pixelmatch + sharp integration, SHA-256 hash shortcircuit, dimension check (error if mismatch), HTML report with base64-inlined images (inline for <= 30 screenshots, folder-based for > 30).

**Week 4: CLI Integration + CI**
- CLI commands: `init`, `record`, `replay`, `diff`, `approve`, `ci`, `status`, `push-baselines`, `pull-baselines`
- Exit codes 0/1/2
- JSON output when piped (`!process.stdout.isTTY`)
- Progress bars (cli-progress) + spinners (ora), suppressed in JSON mode
- GitHub Actions workflow example with Playwright Docker image
- E2E test: Record on TodoMVC → replay → approve baselines → change CSS → replay → detect diff

**Exit criteria**: A developer can `npm install @eyespy/cli`, run `eyespy record --url http://localhost:3000`, click around, then `eyespy ci --url http://localhost:3000`, and get an HTML report showing visual diffs. Works in GitHub Actions with `--ipc=host`.

---

### Phase 1b: Feature Completion (3-4 weeks, 2 developers)

**Goal**: Meticulous parity — coverage-based test selection, SDK recording, WS/SSE, dedup.

**Track 1: Coverage + Test Generation**
- V8 coverage collection via `page.coverage.startJSCoverage({ resetOnNavigation: false })`
- Bitmap extraction (V8 byte ranges → bit vector per branch)
- Version stamping with `sourceContentHash` (not git SHA)
- Source map resolution: parse source maps to build `scriptUrl → localFilePath` mapping
- Greedy set cover algorithm (BitSet-optimized)
- Coverage-guided session skipping (TurboSnap equivalent, using source map resolution)
- `eyespy generate` command
- Staleness detection: invalidate only when `sourceContentHash` changes

**Track 2: Recording SDK + Advanced Mocking**
- Browser SDK (fetch/XHR monkey-patch, event capture, selector generation, transport)
- esbuild → < 18KB gzipped IIFE with `size-limit` enforcement
- Privacy controls: header stripping, configurable field deny-list, production kill switch, mutation throttle
- `eyespy record --listen` always-on receiver
- WebSocket mocking via `page.routeWebSocket()` (Playwright 1.48+)
- SSE mocking via `addInitScript()` EventSource replacement
- Framework testing: React 18/19, Next.js 14/15, Vue 3, Angular 17+ (zone.js ordering), Svelte 5

**Track 3: Polish**
- Remote baseline storage: `eyespy push-baselines` / `eyespy pull-baselines` with pluggable `StorageBackend` (S3/R2/MinIO)
- Advanced auto-healing (full 9-strategy cascade, healing overlay, confidence scoring, dependency graph, skip-cascade propagation, 30% abort threshold)
- Region masking for dynamic content
- JUnit XML report for CI dashboards
- `eyespy prune` with retention policies
- `eyespy sanitize` for sharing sessions
- Fuzzy URL matching for network mock misses (ignore timestamp/nonce params)
- Folder-based HTML report for large suites

**Exit criteria**: `eyespy generate` produces a minimal test suite from 50+ sessions. Session skipping achieves 80%+ time savings. Storage dedup achieves 60%+ savings.

---

### Phase 2: Server + Dashboard + GitHub + Storybook (6-8 weeks)

- **Storybook integration**: `eyespy storybook` command — connects to Storybook's test runner, treats each story as a recording session (navigate to story → screenshot). Leverages coverage-based pruning to skip unchanged stories. Competes directly with Chromatic's TurboSnap but local-first and free. Integration architecture: use `@storybook/test-runner` (Playwright-based) as a peer dependency, inject Eyespy's determinism stack into the test runner's browser context.
- Fastify REST API with GitHub OAuth + RBAC
- PostgreSQL 16 + Drizzle (normalized schema — NO JSONB arrays for events)
- BullMQ + Redis workers with checkpointing + graceful shutdown
- Dashboard (React 19 + Vite): diff viewer, session browser, coverage reports, baseline approval
- GitHub integration: deployment_status webhooks, PR comments, status checks
- URL resolution: Vercel/Netlify deployment detection + configurable URL patterns
- Audit logging for baseline approvals
- Branch-aware baselines (per-branch baseline sets)
- Docker Compose deployment
- **Minimum Prometheus metrics** (required at Phase 2 GA, not deferred): `eyespy_replay_session_duration_ms` histogram, `eyespy_replay_flake_retry_count` counter, `eyespy_route_miss_total{policy}` counter, `eyespy_diff_pixels_ratio` histogram, `eyespy_receiver_requests_total{code}` counter, `eyespy_blob_integrity_failures_total` counter, `eyespy_queue_lag_seconds` gauge. Exposed on `/metrics` endpoint (Prometheus text format).

### Phase 3: Scale + Polish

- Blob storage CDN + replication (S3 Cross-Region, CloudFront for read-heavy CI)
- Horizontal worker scaling + CI job sharding for > 100 sessions
- Coverage treemap in dashboard
- Session playback viewer
- Advanced Prometheus dashboards + Grafana templates
- OpenAPI spec
- Incremental blob GC (reference counting instead of full scan)
- Advanced: BeginFrame (Linux/Windows), Web Worker time mocking, canvas stabilization

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **No Math.random/crypto seeding** | P0 | `addInitScript()` seeded PRNG. Table-stakes. Validated in Phase 0 PoC. |
| **Clock API edge cases (post-#31924 fix)** | P0 | Full `clock.install()` on >= 1.48. Fix landed but is "best-effort". Phase 0 PoC validates. Custom `addInitScript` patches for `document.timeline.currentTime` (#38951) and `MessageChannel` timing (React 18+ scheduler). Fallback: Date-only mock via `addInitScript()` if edge cases hit. |
| **Clock install/pauseAt race (#33926)** | P0 | Between `clock.install({ time: T })` and `clock.pauseAt(T)`, the clock runs. If both use the same time, `pauseAt` can fail with "Cannot fast-forward to the past." Fix: install 1s before target time. Already applied in orchestrator code. |
| **Compositor flag effectiveness (without `--deterministic-mode`)** | P0 | `--deterministic-mode` is UNUSABLE — it enables `--enable-begin-frame-control` which hangs `page.screenshot()` ([#31829](https://github.com/microsoft/playwright/issues/31829)). We use the 6 individual compositor flags instead. Phase 0 PoC validates that these alone produce stable screenshots. Chromatic proves this approach works at scale (7.3B+ tests, vanilla Chrome, no BeginFrame control). |
| **Baseline blob availability (team sharing)** | P1 | `baselines.json` manifest is git-tracked but actual PNGs are in local blob store (gitignored). Teams need `eyespy pull-baselines` before first CI run. Document onboarding. Remote storage (S3/R2) is pluggable. |
| **catch-all route causes flakiness (#22338)** | P0 | #22338 applies to catch-all *fulfill* routes. Our Layer 1 catch-all (`**/*`) only calls `route.fallback()` or `route.abort()`, never `route.fulfill()` — this is safe. Layer 2 uses targeted per-origin routes for mock fulfillment. |
| **CORS preflights bypass routes (#37245)** | P0 | If OPTIONS preflights are intercepted, auto-fulfill with 204 + dynamic Allow headers. If Playwright doesn't surface them, fail fast with `E_CORS_PREFLIGHT_UNSUPPORTED` and recommend live mode for that origin. |
| **Network mock miss (request not in recording)** | P0 | Log diagnostic, abort request in mock mode. Fuzzy URL matching (Phase 1b) for timestamp/nonce params. Report all misses in run summary. |
| **Selector breakage during replay** | P0 | Auto-healing cascade (9 strategies). Basic healing (strategies 1-5) in Phase 1a. Full cascade + healing overlay in Phase 1b. 30% skip threshold aborts session. |
| **Navigation timeout (30s)** | P0 | Capture diagnostic screenshot, log pending network requests, mark as replay-error (exit code 2), skip to next session. |
| **Source map resolution (blocks session skipping)** | P0 | Parse source maps during coverage collection to build scriptUrl → localFilePath mapping. Without this, TurboSnap equivalent can't work. Deferred to Phase 1b. |
| **SDK initialization race** | P0 | Must be first script. Runtime proxy verification. Periodic re-check. (Phase 1b — SDK not in Phase 1a) |
| **No production kill switch** | P0 | allowedHosts + default 0% sampling. (Phase 1b — SDK not in Phase 1a) |
| **Response body security** | P1 | Strip sensitive headers (Auth, Cookie, Set-Cookie). Keep bodies intact for replay. Document risk. `eyespy sanitize` command (Phase 1b) for sharing sessions. Production kill switch prevents accidental prod recording. |
| **Coverage bitmap staleness** | P1 | Version-stamp with `sourceContentHash` (NOT git SHA). Only invalidate when bundle changes. If >50% stale and unreplayable, fall back to "replay all". |
| **Storage explosion** | P1 | SHA-256 dedup (Phase 1b) + retention + session skipping. Flat file storage in Phase 1a is acceptable at MVP scale (< 20 sessions). |
| **Screenshot dimension mismatch** | P1 | Explicit dimension check before diffing. Report as error, don't attempt resize. |
| **Local vs CI baseline divergence** | P1 | Mandate Docker for CI baselines. Document that local runs may differ. Don't allow `eyespy approve` on non-Docker environments by default (override: `--force`). |
| **HTML report size (> 30 screenshots)** | P1 | Auto-switch to folder-based report. Inline base64 only for small suites. |
| Font rendering varies across machines | P1 | Mandate Playwright Docker image for CI. Full flag set (`--font-render-hinting=none` etc). Variable fonts have known Docker issues (#29596) — prefer static font files. |
| Angular zone.js conflict | P1 | Document init ordering. (Phase 1b — SDK not in Phase 1a) |
| Shadow DOM invisible | P1 | Document limitation. Open shadow DOM partially supported. |
| Streaming/chunked responses | P1 | Buffer to string for replay. `route.fulfill()` cannot stream. Document behavioral difference. |
| Third-party scripts in coverage | P1 | Filter by URL prefix matching app origin (`script.url.startsWith(appOrigin)`). |
| **FIFO queue parallel request misordering** | P0 | Two concurrent POST /api/graphql requests get the same `METHOD:URL` key. If they arrive in different order during replay, FIFO returns the wrong response → false positive diff. Fix: two-tier matching — body-fingerprinted queue for POST/PUT/PATCH (truncated SHA-256 of request body), falling back to URL-only FIFO. Already implemented in `buildNetworkQueues` / `registerRoutes`. Validated in Phase 0 PoC with concurrent request test. |
| **FIFO queue depletion on smart retry** | P0 | `.shift()` consumes queue entries. Each retry needs fresh browser context + rebuilt queues from original session data. Validated by architecture: `replaySession()` is a pure function of (session, seed). |
| **WS/SSE handlers run in Node.js (WS) or browser with frozen clock (SSE)** | P0 | WS: `routeWebSocket` handlers use real timers, not mocked clock. SSE: `addInitScript` mock uses browser `setTimeout` which is frozen. Solution for both: clock-coordinated delivery — store pending messages, deliver via replay orchestrator synchronized with `clock.runFor()`. |
| **Source maps unavailable for session skipping** | P1 | Session skipping requires source maps to map scriptUrl → localFilePath. Production builds often strip source maps. Fallback: warn and replay all sessions. Document that `--enable-source-maps` or serving `.map` files is required for session skipping to work. |
| **DOM settle detection hangs with paused clock** | P0 | `setTimeout`/`rAF`/`Date.now()` inside `page.evaluate()` are ALL frozen when `clock.pauseAt()` is active. Solution: Node.js orchestrated polling — use `page.waitForTimeout()` (real time) + instant `page.evaluate()` state checks. Never use browser-side timers for settle detection. |
| **Coverage collected during smart retry** | P1 | Coverage should be OFF during retry attempts (2nd/3rd replays). We already have the bitmap from the first run. Collecting during retries wastes CPU and produces identical results. |
| **V8 source hash non-determinism across rebuilds** | P1 | Modern bundlers are NOT deterministic (chunk hashing, timestamps). Use `sourceContentHash` from `page.coverage.stopJSCoverage()` output — hash V8's actual executed scripts, not build artifacts. |
| **CI cold start (missing blobs)** | P1 | `baselines.json` in git but blobs are local. Auto-detect missing blobs, warn, treat all screenshots as "new" (exit 0). Suggest `eyespy pull-baselines`. |
| **Page-level event listener interference** | P1 | Recorder uses capture-phase `addEventListener` on `document`. Apps that call `stopImmediatePropagation()` on capture phase can block event recording. Mitigation: inject listeners as the very first script via `addInitScript()` — our listeners register before any app code. If still missed, log a warning about uncaptured interactions. |
| **CSS-in-JS class name non-determinism** | P1 | Emotion uses `murmurhash2(styleContent)` — deterministic for same content. Styled-components uses a counter-based class name — deterministic if components evaluate in same order. Tailwind (utility-first) is not affected. Risk: React concurrent mode's time-slicing could change component evaluation order → different styled-components counter → different class names → layout shift. Mitigation: Our `MessageChannel` mock forces React's scheduler into a predictable sequence. Phase 0 PoC archetype B validates this. If it fails, fallback: force React into synchronous mode via `ReactDOM.flushSync` wrapper (documented workaround for users). |
| **React 18 concurrent rendering + MessageChannel mock** | P1 | We mock `MessageChannel.postMessage` to route through `setTimeout(fn, 0)`. React 18's scheduler uses MessageChannel specifically for its "between frames" priority (higher than setTimeout). Routing through setTimeout changes scheduling priority — React batches differently → potentially different component tree at screenshot time. Risk is theoretical — our clock mock freezes ALL timers anyway, so priority doesn't matter when clock is paused. But during `clock.runFor()`, setTimeout callbacks fire in registration order, which may differ from MessageChannel's "fire before rAF" semantics. Phase 0 validates. |
| **Playwright version upgrade breaks all baselines** | P1 | Each Playwright release bundles a new Chromium. New Chromium = different rendering. Upgrading Playwright requires a full baseline reset (`eyespy approve`). Document the upgrade workflow. Pin exact versions. Warn on version mismatch in `eyespy ci`. |
| **Web Workers bypass PRNG seeding** | Scoped out | `addInitScript()` only runs in main page context. Workers that call `crypto.getRandomValues()` or `Math.random()` get real random values. Most SPAs don't use Workers for UUID generation. Document limitation. Phase 1b: investigate `worker.evaluate()` for dedicated workers. Shared/Service Workers are already blocked. |
| Canvas/WebGL non-deterministic | Scoped out | Document limitation. Mask canvas regions in diffs. |
| Cross-origin iframes invisible | Scoped out | Document limitation. Same-origin iframes supported. |
| Service Workers bypass everything | Scoped out | Block SWs during replay (`serviceWorkers: 'block'`). |

---

## Existential Risks (What Kills This Project)

These are not P0/P1 implementation risks — they're strategic risks that could make the entire product concept unviable.

1. **Determinism is fundamentally unachievable without a Chromium fork.** If Phase 0 PoC shows > 2% false positive rate across the archetype matrix even with the full anti-flake stack, the "good enough determinism" bet fails. Meticulous forked Chromium for a reason. Mitigation: smart retry absorbs some flakiness, and pixel thresholds absorb rendering variance. If PoC false positives are 2-5%, we might survive with higher thresholds + smart retry. If > 5%, the product doesn't work.

2. **Nobody records sessions.** The recording step is friction. Developers might prefer writing Playwright tests directly (they already know how) over recording sessions. Meticulous solved this with production recording (zero developer effort) — we can't do that in Phase 1 (SDK is Phase 1b). If Phase 1a's Playwright recorder feels clunky compared to writing `toHaveScreenshot()` assertions, adoption stalls. Mitigation: The `eyespy record --url` UX must be effortless. Phase 1b's SDK enables production/staging recording.

3. **Coverage-based test selection doesn't reduce enough.** The value proposition depends on "record 50 sessions, we auto-select the 8 that matter." If the greedy set cover algorithm can't achieve < 20% selection ratio with > 70% coverage, the feature doesn't justify the complexity. Mitigation: validate with real-world apps in Phase 1b. If coverage overlap is low (each session covers unique branches), the algorithm works. If overlap is high (all sessions cover the same hot paths), selection doesn't help much.

4. **Playwright changes break us.** Monthly Playwright releases can change clock behavior, route interception, screenshot APIs. A single breaking change could require weeks of debugging and a forced baseline reset for all users. Mitigation: pin exact versions, extensive test suite against Playwright internals, document upgrade path.

5. **The "just use Chromatic" objection.** Chromatic has Storybook integration, cloud infrastructure, team features, and a free tier for small projects. "Why would I use a CLI tool when Chromatic just works?" Mitigation: self-hosted/local-first for teams that can't use cloud services + the record/replay workflow for teams that don't use Storybook.

---

## Open Questions (Remaining)

1. **Multi-origin sessions**: OAuth redirects lose recording context. Separate sessions for V1.
2. **Baseline branching**: Single baseline set for V1. Branch-aware in Phase 2.
3. **SDK receiver port**: `eyespy record --listen` spins up a temp HTTP server. Default port 8479, configurable via `--port`.
4. **CI concurrency sharding**: At > 100 sessions, need to shard across multiple CI jobs. Design the sharding mechanism (session ID ranges? round-robin?) before Phase 1b ships.
5. **Phase 2 PostgreSQL schema**: Design the server-side schema during Phase 1b even if not implemented until Phase 2. Retrofitting a schema onto an existing CLI data model causes impedance mismatch.
6. **Replay target URL differs from recording URL**: Sessions record against `http://localhost:3000` but CI may use a different host/port (Docker networking, preview deployments). **Phase 1a solution**: `eyespy replay --url http://ci-container:3000` extracts the origin from the recorded session (`session.url`) and from the `--url` flag, then applies origin substitution to: (a) the initial `page.goto()` URL, (b) all `navigate` event URLs, (c) FIFO queue keys (remap recorded origin to replay origin). Network mocking routes remain targeted by recorded *third-party* origins (Stripe, S3, etc.) which don't change. Only the *app origin* is remapped. **Phase 1b**: configurable multi-origin URL mapping in config for complex setups (e.g., API gateway on different host than frontend).
7. **Concurrent recording + replay**: Can a user record new sessions while replaying old ones? Currently no resource conflict (different browser instances), but both write to `.eyespy/blobs/` — blob store's atomic write (write → rename) handles this safely. Document as supported.
8. **Local development workflow (non-Docker)**: Developers will run `eyespy` locally before committing. Local screenshots will differ from Docker CI screenshots (different OS font rendering, GPU, display). How do we handle this?
   - **Recording**: Always local (headed browser). No Docker needed.
   - **Replay (local)**: Run with `--local` flag (or auto-detect non-Docker). Use **lenient diff thresholds** (`maxDiffPixelRatio: 0.02` locally vs `0.005` in CI). Report diffs but don't fail — print `"Local replay: 3 screenshots differ from baselines (expected — local rendering differs from CI Docker). Run in Docker for authoritative results."`
   - **Replay (CI)**: Docker mandatory. Strict thresholds. Exit code 1 on diff.
   - **`eyespy approve`**: Refuse by default on non-Docker (`--force` override). Baselines MUST be generated in Docker for cross-machine consistency.
   - **Quick local check**: `eyespy replay --local --url http://localhost:3000` runs replay, captures screenshots, diffs with lenient thresholds, generates report. Developers use this to quickly check if their changes look right — not for authoritative pass/fail.
   - **Docker for local authoritative checks**: `docker run --rm -v $(pwd):/work -w /work mcr.microsoft.com/playwright:v1.50.0-noble npx eyespy ci --url http://host.docker.internal:3000`. Document this in `eyespy init` output.

## PRNG Design Note

Two PRNG variants are used for different contexts:
- **xoshiro128\*\*** (32-bit, 4x32-bit state): Injected into browser pages via `addInitScript()`. JS lacks native 64-bit integers, so 32-bit is correct. Used for `Math.random`, `crypto.getRandomValues`, `crypto.randomUUID` seeding.
- **xoshiro256\*\*** (64-bit, 4x64-bit state): Used in Node.js for `@eyespy/shared` PRNG module. Powers the `derive()` function for independent child PRNG streams (one per session, one per subsystem). Uses BigInt for full 64-bit precision.

---

## Competitive Landscape & Moat Analysis

**Threat 1: Playwright's built-in `toHaveScreenshot()`**
Playwright already has visual regression testing. Why would anyone use Eyespy?
- `toHaveScreenshot()` requires writing test scripts manually. Eyespy auto-generates tests from recorded sessions.
- Playwright's approach is one-screenshot-per-assertion. Eyespy captures entire user flows with coverage-guided selection.
- No coverage-based test pruning, no session skipping, no smart retry in Playwright's built-in.
- **Risk**: If Playwright adds recording → replay → screenshot, Eyespy is dead. Probability: low (Playwright team is focused on the test framework, not recording SaaS features).
- **Moat**: Coverage-based test selection is the real differentiator. "Record once, auto-select minimal test suite, skip unaffected sessions" is a workflow Playwright can't replicate without building a test generation engine.

**Threat 2: Meticulous open-sourcing**
Meticulous could open-source their core (unlikely — VC-funded, $12M raised) or a competitor could clone their approach.
- **If Meticulous open-sources**: They have a custom Chromium fork for determinism. We use vanilla Playwright. Their approach is more robust for determinism but harder to maintain. We'd compete on "no custom browser" simplicity.
- **If another startup enters**: First-mover in open-source matters. Having a working CLI with `npm install` adoption before a competitor launches is the moat.
- **Moat**: MIT license + CLI-first (no server required) + coverage-based selection. The combination doesn't exist.

**Threat 3: Chromatic / Storybook ecosystem**
Chromatic dominates Storybook visual testing. If Eyespy doesn't integrate with Storybook, it loses the largest potential audience.
- **Phase 2+ opportunity**: `eyespy storybook` command that records each story as a session. Integrates with Storybook's test runner. Leverages coverage-based pruning to skip unchanged stories (similar to TurboSnap).
- **Differentiation from Chromatic**: Chromatic is cloud-only ($149+/mo for teams). Eyespy is local-first, free, self-hosted.

**Threat 4: AI-powered visual testing (Applitools, Percy)**
These tools use AI/ML for "visual AI" diffing instead of pixel comparison. Smarter but expensive and cloud-dependent.
- **Our approach**: Pixel comparison is deterministic, reproducible, and free. Smart retry handles flakiness. Pixel thresholds + region masking handle intentional changes.
- **Not competing on AI diffing** — competing on the recording → replay → auto-test-selection pipeline.

**Strategic priorities for adoption**:
1. **Phase 1 must nail the developer experience.** `npx eyespy record` → `npx eyespy ci` → HTML report. If this takes more than 15 minutes, developers won't adopt.
2. **Phase 1b's coverage-based test selection is the moat.** This is what makes Eyespy more than "yet another screenshot diffing tool."
3. **Storybook integration is table-stakes for Phase 2.** Without it, we miss the largest visual testing market.
4. **Self-hosted/local-first is the wedge against all cloud competitors.** Teams that can't send code to third-party services (fintech, healthcare, government) have no good option today.

---

## Success Criteria

- **Time to first diff**: < 15 min from `npm install` to first screenshot diff report
- **Recording overhead**: < 5% CPU, < 18KB SDK
- **Replay determinism**: >= 98% identical screenshots across runs (with full anti-flake stack + Docker)
- **Coverage efficiency**: 10-20 sessions → >= 70% branch coverage for typical SPA
- **CI turnaround**: < 5 min for 20-session suite (with session skipping)
- **False positive rate**: < 2% of diffs caused by non-determinism
- **Storage**: < 200MB per run after dedup (vs 1.2GB without)

---

## Verification Plan

**Phase 0:**
1. **Determinism matrix**: Run 5+ app archetypes (TodoMVC React 18, CSS-in-JS, MUI DataGrid, date picker, randomness-heavy page) 10 times each in Docker. Assert 0 diff pixels across all runs for all archetypes. This is the go/no-go gate. Optional archetype F (Next.js 14) is stretch goal.

**Phase 1a:**
2. **Unit tests**: Selector generation, event serialization, network route matching, PRNG seeding, DOM settle detection, pixelmatch integration, HTML report generation
3. **Integration tests**: Replay engine against known pages with recorded network, auto-healing cascade against intentionally broken selectors, navigation timeout recovery, smart retry with fresh contexts (verify FIFO queues rebuilt correctly on each attempt), URL normalization for FIFO matching (trailing slashes, default ports, query param ordering), **timestamp convention** (verify all recorders produce relative-offset timestamps, NOT epoch ms), **URL remapping** (verify navigate events use remapped URLs when `--url` differs from recorded origin), **body-fingerprinted FIFO matching** (two concurrent POST /api/graphql with different bodies → correct response served regardless of arrival order), **Playwright version mismatch warning** (verify eyespy ci warns when installed Playwright differs from config.json)
3b. **Playwright recorder tests**: Record a 10-action sequence on TodoMVC via page-level event capture, verify all events captured with correct selectors (including events inside open Shadow DOM), verify network bodies stored in blob store, verify round-trip (record → save → load → replay produces matching screenshots), verify recording overlay is excluded from captures
4. **E2E test**: Record session on TodoMVC → replay → diff → approve baselines → change CSS → replay → detect diff → approve new baselines
5. **Determinism regression**: 10-run identical screenshot assertion runs in CI on every commit

**Phase 1b:**
6. **Coverage tests**: V8 bitmap extraction, set cover algorithm (with synthetic bitmaps), session skipping (verify correct sessions are skipped for a given diff)
7. **SDK tests**: Event capture in real browser (React 18/19, Next.js 14/15, Vue 3, Angular 17+, Svelte 5), fetch/XHR/WS/SSE interception, transport batching, privacy controls
8. **Advanced healing tests**: Full cascade against progressively degraded DOMs, healing overlay persistence, dependency graph cascade-skip
9. **Performance test**: Replay 20 sessions with session skipping in < 5 min
10. **Storage test**: Verify dedup achieves >= 60% savings across 5 consecutive runs
11. **Size budget**: SDK build stays < 18KB gzipped (enforced by `size-limit` in CI)

## Existing Tools to Leverage (Buy vs Build)

| Subsystem | Buy | Build | Notes |
|-----------|-----|-------|-------|
| Visual diff | `pixelmatch` + `sharp` | Region masking, hash shortcircuit | pixelmatch does 95% |
| Browser automation | `playwright` | PRNG injection, settle detection, route wiring | 70% Playwright APIs, 30% custom glue |
| CLI framework | `commander` + `ora` + `cli-progress` | Command implementations | Standard tools |
| HTML report | — | ~200 lines template | No good library match. String template + inlined assets |
| JUnit XML | `junit-report-builder` | — | Mature, handles CI quirks |
| Content-addressable store | — | ~100 lines | `cacache` too opinionated. Simple to build. |
| Seeded PRNG | — | ~50 lines | Need exact `addInitScript` pattern. `seedrandom` doesn't fit. |
| Coverage bitmap | — | ~150 lines | `v8-to-istanbul` goes to Istanbul format; we skip to bitmaps. |
| Set cover algorithm | — | ~80 lines | No npm package for bitwise set cover. |
| Network interception (SDK) | — | ~300 lines | OpenReplay is AGPL, Sentry is 100K+ lines. Build focused impl. |
| Selector generation | `finder` (MIT) for SDK | Custom `generateSelector()` for recorder | `finder` for SDK (npm). Recorder uses own in-page function (data-testid > id > role > CSS path) since it runs inside `addInitScript()`. |
| Compression (SDK) | `fflate` (~3KB gz) | — | Tree-shakeable, browser-native fallback |
| Bundling | `esbuild` (SDK) + `tsup` (Node) | — | Standard toolchain |
| Size enforcement | `size-limit` | — | Sentry's approach, GitHub Action for PR comments |
