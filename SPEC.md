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
- **Network**: Block all unrecorded origins in mock mode (fully isolated replay)
- **Auto-healing replay**: Basic fallback cascade in 1a, advanced healing overlay + confidence scoring in 1b
- **Response bodies**: No size limit, store in blobs. Strip sensitive headers only (Auth, Cookie, Set-Cookie). Document security implications. No body redaction (breaks replay).
- **Shared package**: Types + shared logic (selector matching, session validation, network normalization)
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
- **Mutation throttle**: Stop recording after 10,000 DOM mutations per 10-second window (Sentry pattern)

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
- **Playwright `animations: 'disabled'`**: Built-in option that handles CSS animations + CSS transitions + WAAPI. Protocol-level, not CSS injection. Finite animations fast-forwarded to completion. Infinite animations cancelled. No conflict with clock mocking (different layers).
- Additional anti-flake CSS: `caret-color: transparent; scroll-behavior: auto !important`
- **Browser flags**: `--deterministic-mode` (meta-flag that sets `--run-all-compositor-stages-before-draw`, `--enable-begin-frame-control`, `--disable-threaded-animation`, `--disable-threaded-scrolling`, `--disable-checker-imaging`, `--disable-image-animation-resync`) plus `--font-render-hinting=none`, `--disable-gpu`, `--force-color-profile=srgb`, `--hide-scrollbars`, `--disable-font-subpixel-positioning`, `--disable-lcd-text`, `--disable-skia-runtime-opts`, `--disable-partial-raster`
- Fixed viewport: 1280x720, deviceScaleFactor: 1
- **Fresh browser context per session**: Each replay session starts with clean state — no localStorage, IndexedDB, cookies, or Service Workers from previous sessions. `browserContext` created with `storageState: undefined`, `serviceWorkers: 'block'`.
- **Font loading gate**: Wait for `document.fonts.ready` AND `document.fonts.status === 'loaded'`
- **Docker mandatory**: Mandate official Playwright Docker image for CI. Document that local replays may differ from CI.

**Network mocking**: **Targeted route patterns** — route only recorded origins (e.g., `page.route('**/api.stripe.com/**')`) instead of catch-all `**/*`. Catch-all introduces flakiness (#22338).
- FIFO per URL+method. Unmatched requests: abort (mock mode) or pass through (live mode)
- **WebSocket mocking**: `page.routeWebSocket()` (Playwright 1.48+). Replay recorded frames in order with timing.
- **SSE mocking**: Replace `EventSource` via `addInitScript()` with a mock that serves recorded events.
- **CORS preflights**: Cannot intercept (#37245). Document limitation. Mock mode should serve responses without CORS restrictions.
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

**Network mock miss handling**: When a request during replay doesn't match any recorded response:
- Log the unmatched URL + method + query params as a diagnostic
- In mock mode: abort the request (`route.abort('connectionrefused')`) — don't hang
- In live mode: pass through to real backend
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
- **Clock lifecycle per screenshot**: `clock.pauseAt()` before capture (freeze all timers), take stabilized screenshot, `clock.resume()` before next interaction.
- Configurable via `screenshotStrategy` in config: `"tiered"` (default), `"every"` (every event), `"manual"` (only explicit `screenshot-marker` events)

**Smart retry**: Replay once. If any screenshot diff detected vs baseline, replay 2 more times to confirm. Majority vote (2-of-3 must show diff for it to count). Only pays 3x cost for actual diffs (~5-20% of screenshots). **Default ON, Phase 1a.**

**Fresh context per retry** (critical implementation detail): FIFO network queues are consumed by `.shift()` during replay — after the first replay, all queues are empty. Each retry attempt MUST:
1. Create a fresh `BrowserContext` (no shared state from previous attempt)
2. Rebuild FIFO queues from the original session data (`buildNetworkQueues(session)`)
3. Re-register all routes on the new context
4. Re-inject PRNG seed and clock (deterministic — same seed produces same sequence)

This means the retry function signature is: `replaySession(session, seed) → screenshots[]`. The orchestrator calls it up to 3 times for sessions with diffs, each call is fully independent. The orchestrator only compares specific screenshots that diffed, not the entire session.

**Coverage collection**: `page.coverage.startJSCoverage({ resetOnNavigation: false })`. Extract branch-level bitmap. Version-stamp with `sourceContentHash` — SHA-256 of the concatenated `entry.source` strings from coverage output. This hashes what V8 actually executed, not build artifacts. Modern bundlers (webpack, vite, esbuild) are NOT deterministic between builds — same source can produce different output due to chunk hashing, timestamps, parallel processing artifacts. Hashing V8's script source at runtime eliminates this problem entirely. Only invalidate bitmaps when the source hash changes. `resetOnNavigation: false` accumulates coverage across SPA navigations within a session.

**Coverage limitation (mocked network)**: V8 coverage tracks which branches execute. With network mocking, responses come from FIFO queues, not real servers. If the app has different code paths for different response shapes/timings/errors, coverage won't capture all production paths. This is inherent — coverage reflects the recorded session's behavior, not all possible behaviors. Document this to users.

**DOM settle detection**: MutationObserver + requestAnimationFrame drain. Wait for 300ms of no DOM changes. Also wait for `document.fonts.ready`, `img.complete` on visible images, and rAF drain (two frames). **Clock interaction**: The settle detection runs inside `page.evaluate()` where `Date.now()` is mocked. Use `page.waitForTimeout()` (real time) as the outer timeout wrapper to prevent hangs when clock is paused.

**Resource constraints**: Chrome uses 3-20GB RAM per instance. Default concurrency: `Math.min(2, Math.floor(os.totalmem() / 5GB))`. Hard timeout: 120s per session replay, 30s per navigation.

### 3. Test Generator (Coverage-Based Selection)

**Algorithm**:
1. For each recorded session: replay with V8 coverage ON, extract branch-level coverage bitmap
2. Version-stamp bitmap with `(gitSha, sourceContentHash)`
3. Deduplicate: sessions with identical bitmaps → keep one (same flow, different data)
4. Greedy set cover: iteratively select session covering most uncovered branches
5. Stop when target coverage reached (default 100% of reachable branches)
6. Result: minimal set of K sessions (K << N total) that maximize coverage

**Staleness management**:
- Bitmaps stamped with code version. If git SHA differs from current HEAD, mark as "stale-pending"
- On replay against new code: re-collect coverage and update bitmap (self-healing)
- If >50% of sessions are stale and unreplayable, fall back to "replay all"
- Coverage-guided session skipping (TurboSnap equivalent): only replay sessions whose bitmaps overlap with files changed in the current diff. 80%+ time savings.

**File storage** (CLI-first):
- `.eyespy/coverage/` — bitmap files per session, keyed by session ID
- `.eyespy/suites/` — generated suite config (list of session IDs + coverage stats)

### 4. Visual Diff Engine

- pixelmatch with configurable tolerance (`threshold`, `maxDiffPixels`, `maxDiffPixelRatio`)
- Anti-aliasing detection ON by default
- **Dimension check**: If baseline and current screenshots differ in dimensions, report as error (not diff). Likely means viewport changed or scrollbar appeared. Don't attempt resize — the mismatch itself is the signal.
- Region masking: rectangles or CSS selectors for dynamic content (timestamps, ads, avatars) **(Phase 1b)**
- Content-hash short-circuit: SHA-256 hash screenshot before diffing. If hash matches baseline, skip pixelmatch entirely.
- Three-panel output: expected | actual | diff (red pixels)
- **HTML report**: Standalone HTML file with base64-inlined images for <= 30 screenshots. **Folder-based report** (images as separate files + index.html) for > 30 screenshots. Four viewer modes: side-by-side, slider (CSS `clip-path`), blend (opacity crossfade), toggle (click to switch).

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

JSON output when piped (`| jq`), human-friendly TTY output otherwise. Exit code 0 = all pass, 1 = diffs detected, 2 = replay failures.

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

**Baseline manifest format**:
```json
{
  "version": 1,
  "baselines": {
    "session-abc123": {
      "001-page-load": { "hash": "sha256:e3b0c44...", "width": 1280, "height": 720 },
      "002-click-submit": { "hash": "sha256:a1f2b3c...", "width": 1280, "height": 720 }
    }
  }
}
```

**Team sharing**: By default, blobs are local. For teams, configure a remote storage backend:
```bash
eyespy push-baselines              # Upload baseline blobs to remote
eyespy pull-baselines              # Download baseline blobs from remote
```
Pluggable storage interface: `StorageBackend { upload(hash, data), download(hash), has(hash) }`. Built-in: local filesystem (default), S3-compatible (AWS S3, Cloudflare R2, MinIO, Backblaze B2). Designed following reg-suit's plugin architecture.

**CI cold start (missing blobs)**: When `baselines.json` exists (git-tracked) but referenced blob files are missing (new developer clone, fresh CI without `pull-baselines`), auto-detect and handle gracefully:
1. On `eyespy ci` / `eyespy diff`: Check if all hashes in `baselines.json` exist in local blob store
2. If any missing: print warning `"⚠ N baseline blobs missing. Run 'eyespy pull-baselines' or 'eyespy approve' to establish local baselines. Treating all screenshots as new."`
3. Treat all screenshots as "new" (no diff possible), exit 0 (not exit 1 — missing blobs aren't a regression)
4. If remote storage is configured but `pull-baselines` wasn't run: suggest `eyespy pull-baselines` in the warning
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
    "includeAA": false
  },
  "replay": {
    "screenshotStrategy": "tiered",
    "quiescenceMs": 300,
    "sessionTimeoutMs": 120000,
    "navigationTimeoutMs": 30000,
    "concurrency": "auto",
    "smartRetry": true,
    "mode": "mock"
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
    "excludePaths": [],
    "sensitiveHeaders": ["Authorization", "Cookie", "Set-Cookie"]
  },
  "report": {
    "format": "html",
    "inlineThreshold": 30
  }
}
```

---

## Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Recording SDK | **TypeScript → ES2020 via esbuild** | IIFE bundle, < 18KB gz. ES2020 covers Chrome 80+/Firefox 80+/Safari 14+. ES5 adds 15-30% bloat and Proxy can't be polyfilled anyway. |
| Replay engine | **Playwright >= 1.48** | Clock API (with #31924 fix), routeWebSocket, animations:disabled |
| CLI | **Commander.js** | Lightweight, standard |
| Image diff | **pixelmatch + sharp** | Industry standard |
| Coverage | **Playwright `page.coverage` API** (V8 Profiler) | Branch-level precision, simpler than raw CDP |
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
│   │       ├── settle/        # MutationObserver + rAF drain + font loading + img.complete
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
// Store original before any framework loads
const _originalFetch = window.fetch.bind(window);
const _originalXHROpen = XMLHttpRequest.prototype.open;
const _originalXHRSend = XMLHttpRequest.prototype.send;

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
    timestamp: Date.now(),
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

    recordNetworkEvent({ type: 'ws-open', wsId, url, timestamp: Date.now() });

    ws.addEventListener('message', (e: MessageEvent) => {
      recordNetworkEvent({
        type: 'ws-receive', wsId, url,
        data: typeof e.data === 'string' ? truncate(e.data, MAX_BODY_SIZE) : '<binary>',
        timestamp: Date.now(),
      });
    });
    ws.addEventListener('close', (e: CloseEvent) => {
      recordNetworkEvent({ type: 'ws-close', wsId, url, code: e.code, timestamp: Date.now() });
    });

    const _originalSend = ws.send.bind(ws);
    ws.send = function(data: any) {
      recordNetworkEvent({
        type: 'ws-send', wsId, url,
        data: typeof data === 'string' ? truncate(data, MAX_BODY_SIZE) : '<binary>',
        timestamp: Date.now(),
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

    recordNetworkEvent({ type: 'sse-open', sseId, url, timestamp: Date.now() });

    const _originalAddEventListener = es.addEventListener.bind(es);
    es.addEventListener = function(type: string, listener: any, options?: any) {
      const wrappedListener = (event: MessageEvent) => {
        recordNetworkEvent({
          type: 'sse-event', sseId, url, eventType: type,
          data: event.data, lastEventId: event.lastEventId,
          timestamp: Date.now(),
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
            data: event.data, timestamp: Date.now(),
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

### Playwright Recorder: CDP-Based Event Capture

The CLI recorder (`eyespy record --url`) uses Playwright's CDP session to listen for raw browser input events — the same mechanism Playwright's codegen uses internally. This captures events at the browser level, not the page level, so it works regardless of page JavaScript, Shadow DOM, or framework quirks.

```typescript
async function startRecording(page: Page): Promise<RecordedEvent[]> {
  const events: RecordedEvent[] = [];
  const cdp = await page.context().newCDPSession(page);

  // Enable DOM domain for element resolution
  await cdp.send('DOM.enable');

  // Listen for mouse events at the Input domain level
  cdp.on('Input.dispatchMouseEvent', async (params) => {
    if (params.type !== 'mousePressed') return; // Only record clicks, not moves

    // Resolve (x, y) coordinates to a DOM node
    const { nodeId } = await cdp.send('DOM.getNodeForLocation', {
      x: Math.round(params.x),
      y: Math.round(params.y),
      includeUserAgentShadowDOM: false,
    });

    // Get node attributes for selector generation
    const { node } = await cdp.send('DOM.describeNode', { nodeId, depth: 0 });

    // Generate selector bundle from the resolved node
    const selector = await generateSelectorFromNode(page, cdp, nodeId, node);

    events.push({
      type: 'click',
      timestamp: Date.now(),
      selector,
      x: params.x,
      y: params.y,
      button: params.button || 0,
      modifiers: {
        meta: !!(params.modifiers & 4),
        ctrl: !!(params.modifiers & 2),
        shift: !!(params.modifiers & 8),
        alt: !!(params.modifiers & 1),
      },
    });
  });

  // Keyboard events via Input domain
  cdp.on('Input.dispatchKeyEvent', (params) => {
    if (params.type === 'keyDown') {
      events.push({
        type: 'keydown',
        timestamp: Date.now(),
        key: params.key || '',
        code: params.code || '',
        modifiers: {
          meta: !!(params.modifiers & 4),
          ctrl: !!(params.modifiers & 2),
          shift: !!(params.modifiers & 8),
          alt: !!(params.modifiers & 1),
        },
      });
    }
  });

  // Network capture via Playwright's high-level API (simpler than raw CDP)
  page.on('request', (request) => {
    events.push({
      type: 'network',
      timestamp: Date.now(),
      event: {
        type: 'request',
        requestId: request.url() + ':' + Date.now(), // Unique enough for recording
        url: request.url(),
        method: request.method(),
        headers: sanitizeHeaders(request.headers()),
        timestamp: Date.now(),
      },
    });
  });

  page.on('response', async (response) => {
    try {
      const body = await response.body(); // Async — may fail for redirects
      const bodyHash = await blobStore.put(body);
      events.push({
        type: 'network',
        timestamp: Date.now(),
        event: {
          type: 'response',
          requestId: response.url() + ':' + Date.now(),
          url: response.url(),
          method: response.request().method(),
          status: response.status(),
          headers: sanitizeHeaders(response.headers()),
          bodyHash,
          durationMs: 0, // Timing not critical for recording
        },
      });
    } catch {} // Some responses have no body (204, redirects)
  });

  return events; // Caller stops recording on user signal
}
```

**Selector generation from CDP node** — Use `DOM.getNodeForLocation` to resolve click coordinates to a DOM element, then generate selectors in-page:

```typescript
async function generateSelectorFromNode(
  page: Page, cdp: CDPSession, nodeId: number, node: Protocol.DOM.Node
): Promise<SelectorBundle> {
  // Resolve nodeId to a Playwright-usable object
  const { object } = await cdp.send('DOM.resolveNode', { nodeId });
  const remoteObjectId = object.objectId;

  // Run selector generation in page context using the resolved element
  const result = await page.evaluate(
    ({ objectId }) => {
      // __eyespy_resolvedElement is set by a helper that maps objectId → element
      const el = (window as any).__eyespy_resolvedElement;
      if (!el) return null;

      const strategies: string[] = [];

      // Strategy 1: data-testid (highest priority)
      const testId = el.getAttribute('data-testid') || el.getAttribute('data-test-id');
      if (testId) strategies.push(`[data-testid="${testId}"]`);

      // Strategy 2: id (if not dynamically generated)
      if (el.id && !/^[0-9]|[:.]|--/.test(el.id)) strategies.push(`#${el.id}`);

      // Strategy 3: role + aria-label
      const role = el.getAttribute('role') || el.tagName.toLowerCase();
      const ariaLabel = el.getAttribute('aria-label');
      if (ariaLabel) strategies.push(`${role}[aria-label="${ariaLabel}"]`);

      // Strategy 4: structural CSS path (fallback)
      // Walk up the tree building a unique path
      function cssPath(element: Element): string {
        const parts: string[] = [];
        let current: Element | null = element;
        while (current && current !== document.body) {
          let selector = current.tagName.toLowerCase();
          if (current.id && !/^[0-9]|[:.]|--/.test(current.id)) {
            selector = `#${current.id}`;
            parts.unshift(selector);
            break;
          }
          const parent = current.parentElement;
          if (parent) {
            const siblings = Array.from(parent.children).filter(c => c.tagName === current!.tagName);
            if (siblings.length > 1) {
              const idx = siblings.indexOf(current) + 1;
              selector += `:nth-of-type(${idx})`;
            }
          }
          parts.unshift(selector);
          current = parent;
        }
        return parts.join(' > ');
      }

      const primary = strategies[0] || cssPath(el);
      const fallbacks = strategies.slice(1);
      if (!strategies.includes(cssPath(el))) fallbacks.push(cssPath(el));

      return {
        primary,
        fallbacks,
        fingerprint: {
          text: el.innerText?.substring(0, 50),
          rect: el.getBoundingClientRect().toJSON(),
          tagName: el.tagName.toLowerCase(),
        },
      };
    },
    {} // Placeholder — actual element binding happens via CDP
  );

  return result || { primary: `body`, fallbacks: [], fingerprint: { tagName: 'body' } };
}
```

**Note on element resolution**: CDP `DOM.getNodeForLocation` returns the topmost element at the given coordinates, accounting for z-index and stacking contexts. This is more reliable than page-side `document.elementFromPoint()` because it works even when the element is inside a Shadow DOM or behind an overlay. The actual element binding between CDP nodeId and page JavaScript uses `DOM.resolveNode` → `Runtime.callFunctionOn` with the resolved `RemoteObjectId`.

**Input capture**: The CDP `Input` domain listeners are passive observers — they don't intercept or modify input. The user interacts normally with the page while we record. This is fundamentally different from the SDK approach (which monkey-patches APIs) — CDP recording is invisible to the page.

### Recording SDK: Key Data Structures

**Session format** — Flat event array with monotonic timestamps. Network bodies stored separately in blobs.

```typescript
interface RecordedSession {
  formatVersion: 1;              // Schema version — reject with clear error if mismatch
  id: string;                    // UUID v4
  startedAt: string;             // ISO 8601
  endedAt: string;
  url: string;                   // Starting URL
  viewport: { width: number; height: number };
  userAgent: string;
  captureMethod: 'playwright' | 'sdk';  // Which recorder produced this session
  events: RecordedEvent[];       // Chronological, monotonic timestamps
  networkBodies: Record<string, string>; // requestId → blob hash (large bodies stored separately)
}

type RecordedEvent =
  | { type: 'click'; timestamp: number; selector: SelectorBundle; x: number; y: number; button: number; modifiers: Modifiers }
  | { type: 'dblclick'; timestamp: number; selector: SelectorBundle; x: number; y: number }
  | { type: 'input'; timestamp: number; selector: SelectorBundle; value: string; inputType?: string }
  | { type: 'keydown'; timestamp: number; key: string; code: string; modifiers: Modifiers }
  | { type: 'keyup'; timestamp: number; key: string; code: string }
  | { type: 'scroll'; timestamp: number; selector: SelectorBundle | null; x: number; y: number }
  | { type: 'navigate'; timestamp: number; url: string; navigationType: 'push' | 'replace' | 'popstate' }
  | { type: 'resize'; timestamp: number; width: number; height: number }
  | { type: 'focus'; timestamp: number; selector: SelectorBundle }
  | { type: 'blur'; timestamp: number; selector: SelectorBundle }
  | { type: 'pointerdown'; timestamp: number; selector: SelectorBundle; x: number; y: number; pointerId: number }
  | { type: 'pointermove'; timestamp: number; x: number; y: number; pointerId: number }
  | { type: 'pointerup'; timestamp: number; x: number; y: number; pointerId: number }
  | { type: 'touchstart'; timestamp: number; selector: SelectorBundle; touches: Touch[] }
  | { type: 'touchmove'; timestamp: number; touches: Touch[] }
  | { type: 'touchend'; timestamp: number; touches: Touch[] }
  | { type: 'dragstart'; timestamp: number; selector: SelectorBundle; x: number; y: number }
  | { type: 'drop'; timestamp: number; selector: SelectorBundle; x: number; y: number }
  | { type: 'paste'; timestamp: number; selector: SelectorBundle; text: string }
  | { type: 'copy'; timestamp: number; selector: SelectorBundle }
  | { type: 'cut'; timestamp: number; selector: SelectorBundle }
  | { type: 'network'; timestamp: number; event: NetworkEvent }
  | { type: 'screenshot-marker'; timestamp: number; label: string };

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

type NetworkEvent =
  | { type: 'request'; requestId: string; url: string; method: string; headers?: Record<string, string>; body?: string; timestamp: number }
  | { type: 'response'; requestId: string; url: string; method: string; status: number; headers?: Record<string, string>; bodyHash: string; durationMs: number; error?: string }
  | { type: 'ws-open'; wsId: string; url: string; timestamp: number }
  | { type: 'ws-send'; wsId: string; url: string; data: string; timestamp: number }
  | { type: 'ws-receive'; wsId: string; url: string; data: string; timestamp: number }
  | { type: 'ws-close'; wsId: string; url: string; code: number; timestamp: number }
  | { type: 'sse-open'; sseId: string; url: string; timestamp: number }
  | { type: 'sse-event'; sseId: string; url: string; eventType: string; data: string; lastEventId?: string; timestamp: number };
// Note: Response bodies stored in blob store, referenced by bodyHash.
// The bodyHash maps to a SHA-256 key in .eyespy/blobs/.
```

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
// Build FIFO queues keyed by method + full URL (not just origin)
function buildNetworkQueues(session: RecordedSession): Map<string, NetworkEvent[]> {
  const queues = new Map<string, NetworkEvent[]>(); // "METHOD:fullUrl" → FIFO queue
  for (const event of session.events) {
    if (event.type !== 'network' || event.event.type !== 'request') continue;
    const key = `${event.event.method}:${event.event.url}`;
    if (!queues.has(key)) queues.set(key, []);
    // Find matching response
    const response = findResponse(session.events, event.event.requestId);
    if (response) queues.get(key)!.push(response);
  }
  return queues;
}

// Register routes per origin (NOT catch-all)
async function registerRoutes(context: BrowserContext, session: RecordedSession, mode: 'mock' | 'live') {
  const origins = new Set<string>();
  for (const event of session.events) {
    if (event.type === 'network' && event.event.type === 'request') {
      origins.add(new URL(event.event.url).origin);
    }
  }

  // FIFO queues keyed by "METHOD:fullUrl" — same key format used in lookup below
  const queues = buildNetworkQueues(session);

  for (const origin of origins) {
    // Targeted pattern: only intercept this specific origin
    await context.route(`**/${new URL(origin).host}/**`, async (route) => {
      const request = route.request();
      const key = `${request.method()}:${request.url()}`;
      const queue = queues.get(key);

      if (queue && queue.length > 0) {
        const recorded = queue.shift()!;
        const body = await loadBlob(recorded.bodyHash); // From content-addressable store
        await route.fulfill({
          status: recorded.status,
          headers: recorded.headers,
          body: body,
        });
      } else if (mode === 'mock') {
        await route.abort('connectionrefused');
      } else {
        await route.continue(); // Live mode: pass through
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

```javascript
// Injected via page.addInitScript(script, sseRecordings)
(function(recordings) {
  const _OriginalEventSource = window.EventSource;

  class MockEventSource {
    constructor(url, init) {
      this.url = url;
      this.readyState = 0; // CONNECTING
      this._listeners = {};
      this.onopen = null;
      this.onmessage = null;
      this.onerror = null;

      const recorded = recordings.find(r => url.includes(r.url));
      if (!recorded) {
        // Fallback to real EventSource if not recorded.
        // NOTE: Returning an object from a class constructor overrides `this` in JS.
        // This works but is fragile — the returned object IS the instance.
        return new _OriginalEventSource(url, init);
      }

      // Simulate connection
      setTimeout(() => {
        this.readyState = 1; // OPEN
        if (this.onopen) this.onopen(new Event('open'));
        this._dispatch('open', new Event('open'));

        // Replay recorded events with timing
        let baseTime = recorded.events[0]?.timestamp || 0;
        for (const evt of recorded.events) {
          const delay = evt.timestamp - baseTime;
          setTimeout(() => {
            const messageEvent = new MessageEvent(evt.eventType || 'message', {
              data: evt.data,
              lastEventId: evt.lastEventId || '',
            });
            if (evt.eventType === 'message' && this.onmessage) {
              this.onmessage(messageEvent);
            }
            this._dispatch(evt.eventType || 'message', messageEvent);
          }, delay);
        }
      }, 10);
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

  // Preserve static properties
  MockEventSource.CONNECTING = 0;
  MockEventSource.OPEN = 1;
  MockEventSource.CLOSED = 2;
  window.EventSource = MockEventSource;
})(arguments[0]);
```

### Replay Engine: DOM Settle Detection

```typescript
async function waitForDomSettle(page: Page, options: { timeout?: number; quiescence?: number } = {}) {
  const { timeout = 10000, quiescence = 300 } = options;

  await page.evaluate(async (quiescenceMs) => {
    return new Promise<void>((resolve, reject) => {
      let timer: ReturnType<typeof setTimeout>;
      let settled = false;
      const deadline = Date.now() + 10000; // Hard deadline inside page

      // 1. MutationObserver — detect DOM changes
      const observer = new MutationObserver(() => {
        clearTimeout(timer);
        timer = setTimeout(checkSettle, quiescenceMs);
      });
      observer.observe(document.body, {
        childList: true, subtree: true, attributes: true, characterData: true
      });

      // 2. Check settle conditions
      async function checkSettle() {
        if (settled) return;
        if (Date.now() > deadline) { finish(); return; }

        // Wait for fonts
        if (document.fonts.status !== 'loaded') {
          await document.fonts.ready;
          timer = setTimeout(checkSettle, quiescenceMs);
          return;
        }

        // Wait for visible images
        const images = Array.from(document.querySelectorAll('img'));
        const visibleImages = images.filter(img => {
          const rect = img.getBoundingClientRect();
          return rect.top < window.innerHeight && rect.bottom > 0;
        });
        const unloaded = visibleImages.filter(img => !img.complete);
        if (unloaded.length > 0) {
          await Promise.all(unloaded.map(img =>
            new Promise(r => { img.onload = r; img.onerror = r; })
          ));
          timer = setTimeout(checkSettle, quiescenceMs);
          return;
        }

        // Wait for rAF drain (one full frame)
        await new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r)));
        finish();
      }

      function finish() {
        settled = true;
        observer.disconnect();
        clearTimeout(timer);
        resolve();
      }

      // Start the quiescence timer
      timer = setTimeout(checkSettle, quiescenceMs);
    });
  }, quiescence);
}
```

### Coverage: V8 Format and Bitmap Conversion

**V8 coverage raw format** (from CDP `Profiler.takePreciseCoverage`):

```typescript
interface V8CoverageResult {
  result: Array<{
    scriptId: string;
    url: string;
    functions: Array<{
      functionName: string;
      ranges: Array<{
        startOffset: number;  // Byte offset in source
        endOffset: number;
        count: number;        // Execution count
      }>;
      isBlockCoverage: boolean;
    }>;
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

function v8ToBitmap(coverage: V8CoverageResult, appOrigin: string): CoverageBitmap {
  const branches: Array<{ scriptUrl: string; start: number; end: number; covered: boolean }> = [];

  for (const script of coverage.result) {
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

// popcount for single byte
function popcount8(n: number): number {
  n = n - ((n >> 1) & 0x55);
  n = (n & 0x33) + ((n >> 2) & 0x33);
  return (n + (n >> 4)) & 0x0f;
}
```

### Coverage: Session Skipping (TurboSnap Equivalent)

```typescript
/**
 * Given a git diff and coverage bitmaps, determine which sessions
 * need to be replayed. Sessions whose coverage doesn't overlap
 * with changed files can be skipped entirely.
 *
 * Chromatic's TurboSnap traces Webpack module dependency graphs.
 * We use a simpler approach: URL prefix matching on coverage scripts.
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
    includeAA?: boolean;         // Include anti-aliased pixels, default false
    maskRegions?: Array<{ x: number; y: number; width: number; height: number }>;
  } = {}
): Promise<DiffResult> {
  const { threshold = 0.1, maxDiffPixelRatio = 0.005, includeAA = false } = options;

  // Read both images
  const [baselineBuffer, currentBuffer] = await Promise.all([
    sharp(baselinePath).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
    sharp(currentPath).raw().ensureAlpha().toBuffer({ resolveWithObject: true }),
  ]);

  // Content-hash short-circuit
  const baselineHash = crypto.createHash('sha256').update(baselineBuffer.data).digest('hex');
  const currentHash = crypto.createHash('sha256').update(currentBuffer.data).digest('hex');
  if (baselineHash === currentHash) {
    return { identical: true, diffPixels: 0, diffRatio: 0, exceedsThreshold: false, baselineHash, currentHash };
  }

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
| **Chromatic (TurboSnap)** | Traces Webpack dependency graph to skip unaffected stories. Our URL-prefix approach is simpler but less precise. |
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

1. Template HTML with `{{css}}`, `{{js}}`, `{{data}}` placeholders
2. Bundled viewer app (vanilla JS or Preact, ~5KB) injected inline
3. Report data as `window.__EYESPY_DATA__` JSON blob
4. Images as `data:image/png;base64,...` data URLs

Four viewer modes: side-by-side, slider (CSS `clip-path`), blend (opacity crossfade), toggle (click to switch).

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

**`animations: 'disabled'`** — Protocol-level, NOT CSS injection. Finite animations fast-forwarded to completion state. Infinite animations cancelled and reset. Elements end up in their "after" state, not "before".

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
- Order between multiple context-level and page-level scripts is NOT guaranteed — don't depend on it
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

---

## Build Sequence

### Phase 0: Proof of Concept (1-2 days, 1 developer)

**Goal**: Validate that deterministic replay produces pixel-identical screenshots.

Single file (`proof-of-concept.ts`, ~200 lines) that:
1. Launches Playwright with `--deterministic-mode` + full anti-flake flags (`--font-render-hinting=none`, `--disable-gpu`, etc.)
2. Injects PRNG seeding via `addInitScript()` (Math.random, crypto.randomUUID, crypto.getRandomValues)
3. Installs `clock.install()` with fixed time + patches for `document.timeline.currentTime` and `MessageChannel`
4. Navigates to TodoMVC React, performs 5 hardcoded actions
5. Takes screenshots after each action with DOM settle-wait
6. Runs the sequence 10 times in Docker
7. Compares all screenshots with pixelmatch
8. Reports: 0 diff pixels = success. Any diff = must debug before proceeding.

**Exit criteria**: 10/10 runs produce pixel-identical screenshots in the Playwright Docker image.

**Risk being validated**: Deterministic replay is achievable with Playwright's stack without forking Chromium. **If this fails, the entire product concept needs re-evaluation.**

---

### Phase 1a: MVP (3-4 weeks, 1-2 developers)

**Goal**: `npx eyespy record --url <url>` → `npx eyespy ci --url <url>` → HTML diff report.

**Week 1: Foundation**
- Monorepo scaffolding (pnpm workspaces, Turborepo, Biome, Vitest)
- `@eyespy/shared`: Frozen TypeScript interfaces (`RecordedSession` with `formatVersion`, `RecordedEvent`, `SelectorBundle`, `DiffResult`, `RunResult`, config schema), seeded PRNG (xoshiro128** for browser injection, xoshiro256** for Node), content-addressable blob store (~100 lines)
- `.eyespy/` directory structure, `eyespy init` auto-creation (config.json + baselines.json + .gitignore updates)

**Week 2: Replay Engine (CRITICAL PATH)**
- Playwright launcher with `--deterministic-mode` + full anti-flake flags + `animations: 'disabled'`
- PRNG injection via `context.addInitScript()` (Math.random, crypto.randomUUID, crypto.getRandomValues)
- Full `clock.install()` with `pauseAt`/`runFor`/`resume` lifecycle (Playwright >= 1.48)
- Clock gap patches via `addInitScript()`: `document.timeline.currentTime` (#38951), `MessageChannel` timing
- HTTP network mocking (targeted per-origin routes via `browserContext.route()`, FIFO queue by method+fullUrl)
- **Block unrecorded origins** (abort in mock mode)
- Fresh browser context per session (no localStorage/IndexedDB/cookies bleed)
- DOM settle detection (MutationObserver + rAF + font loading + img.complete, 300ms quiescence)
- Tiered screenshot capture with stabilization loop (two-consecutive-identical)
- Smart retry (1x replay, re-confirm diffs 2x, majority vote — default ON)
- Basic auto-healing (strategies 1-5: primary retry, testid, role+label, role+text, text+tag)
- Navigation timeout recovery (diagnostic screenshot, log pending requests, mark as error)
- Hard timeouts: 120s/session, 30s/navigation
- **Determinism test**: 10-run identical screenshot assertion in Docker

**Week 3: Recorder + Diff Engine (parallel tracks)**
- *Playwright Recorder*: CDP-based recording using `CDPSession` — `Input.dispatchMouseEvent`/`Input.dispatchKeyEvent` listeners for raw user events, `DOM.getNodeForLocation(x,y)` to resolve click coordinates to DOM elements, then selector generation on the resolved node. Network via `page.on('request'/'response')` with async `response.body()` capture. Save as `RecordedSession` JSON. Use `addInitScript()` for selector fingerprint helpers (innerText, boundingRect, data-testid resolution).
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

### Phase 2: Server + Dashboard + GitHub Integration (6-8 weeks)

- Fastify REST API with GitHub OAuth + RBAC
- PostgreSQL 16 + Drizzle (normalized schema — NO JSONB arrays for events)
- BullMQ + Redis workers with checkpointing + graceful shutdown
- Dashboard (React 19 + Vite): diff viewer, session browser, coverage reports, baseline approval
- GitHub integration: deployment_status webhooks, PR comments, status checks
- URL resolution: Vercel/Netlify deployment detection + configurable URL patterns
- Audit logging for baseline approvals
- Branch-aware baselines (per-branch baseline sets)
- Docker Compose deployment

### Phase 3: Scale + Polish

- S3-compatible blob storage
- Horizontal worker scaling + CI job sharding for > 100 sessions
- Coverage treemap in dashboard
- Session playback viewer
- Prometheus metrics
- OpenAPI spec
- Incremental blob GC (reference counting instead of full scan)
- Advanced: BeginFrame (Linux/Windows), Web Worker time mocking, canvas stabilization

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **No Math.random/crypto seeding** | P0 | `addInitScript()` seeded PRNG. Table-stakes. Validated in Phase 0 PoC. |
| **Clock API edge cases (post-#31924 fix)** | P0 | Full `clock.install()` on >= 1.48. Fix landed but is "best-effort". Phase 0 PoC validates. Custom `addInitScript` patches for `document.timeline.currentTime` (#38951) and `MessageChannel` timing (React 18+ scheduler). Fallback: Date-only mock via `addInitScript()` if edge cases hit. |
| **`--deterministic-mode` flag availability** | P0 | Meta-flag sets 6 compositor/animation flags. Verify presence in target Playwright Chromium version during Phase 0 PoC. If missing, set component flags individually. |
| **Baseline blob availability (team sharing)** | P1 | `baselines.json` manifest is git-tracked but actual PNGs are in local blob store (gitignored). Teams need `eyespy pull-baselines` before first CI run. Document onboarding. Remote storage (S3/R2) is pluggable. |
| **catch-all route causes flakiness (#22338)** | P0 | Targeted per-origin routes, not `**/*`. |
| **CORS preflights bypass routes (#37245)** | P0 | Document limitation. Playwright auto-fulfills OPTIONS with 204 when interception active. |
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
| **FIFO queue depletion on smart retry** | P0 | `.shift()` consumes queue entries. Each retry needs fresh browser context + rebuilt queues from original session data. Validated by architecture: `replaySession()` is a pure function of (session, seed). |
| **WS/SSE route handlers run in Node.js, not browser** | P0 | `routeWebSocket` handlers use real timers, not mocked clock. Solution: clock-coordinated delivery — send WS messages from replay orchestrator synchronized with `clock.runFor()`, not via setTimeout in handlers. |
| **V8 source hash non-determinism across rebuilds** | P1 | Modern bundlers are NOT deterministic (chunk hashing, timestamps). Use `sourceContentHash` from `page.coverage.stopJSCoverage()` output — hash V8's actual executed scripts, not build artifacts. |
| **CI cold start (missing blobs)** | P1 | `baselines.json` in git but blobs are local. Auto-detect missing blobs, warn, treat all screenshots as "new" (exit 0). Suggest `eyespy pull-baselines`. |
| **CDP Input domain listener availability** | P1 | CDP `Input.dispatchMouseEvent` listener is how Playwright's codegen works internally. Well-supported. Fallback: page-level event listeners via `addInitScript()` (less reliable for Shadow DOM). |
| Canvas/WebGL non-deterministic | Scoped out | Document limitation. Mask canvas regions in diffs. |
| Cross-origin iframes invisible | Scoped out | Document limitation. Same-origin iframes supported. |
| Service Workers bypass everything | Scoped out | Block SWs during replay (`serviceWorkers: 'block'`). |

---

## Open Questions (Remaining)

1. **Multi-origin sessions**: OAuth redirects lose recording context. Separate sessions for V1.
2. **Baseline branching**: Single baseline set for V1. Branch-aware in Phase 2.
3. **SDK receiver port**: `eyespy record --listen` spins up a temp HTTP server. Default port 8479, configurable via `--port`.
4. **CI concurrency sharding**: At > 100 sessions, need to shard across multiple CI jobs. Design the sharding mechanism (session ID ranges? round-robin?) before Phase 1b ships.
5. **Phase 2 PostgreSQL schema**: Design the server-side schema during Phase 1b even if not implemented until Phase 2. Retrofitting a schema onto an existing CLI data model causes impedance mismatch.

## PRNG Design Note

Two PRNG variants are used for different contexts:
- **xoshiro128\*\*** (32-bit, 4x32-bit state): Injected into browser pages via `addInitScript()`. JS lacks native 64-bit integers, so 32-bit is correct. Used for `Math.random`, `crypto.getRandomValues`, `crypto.randomUUID` seeding.
- **xoshiro256\*\*** (64-bit, 4x64-bit state): Used in Node.js for `@eyespy/shared` PRNG module. Powers the `derive()` function for independent child PRNG streams (one per session, one per subsystem). Uses BigInt for full 64-bit precision.

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
1. **Determinism test**: Replay same 5-action sequence 10 times in Docker. Assert 0 diff pixels across all runs. This is the go/no-go gate.

**Phase 1a:**
2. **Unit tests**: Selector generation, event serialization, network route matching, PRNG seeding, DOM settle detection, pixelmatch integration, HTML report generation
3. **Integration tests**: Replay engine against known pages with recorded network, auto-healing cascade against intentionally broken selectors, navigation timeout recovery, smart retry with fresh contexts (verify FIFO queues rebuilt correctly on each attempt)
3b. **CDP recorder tests**: Record a 10-action sequence on TodoMVC via CDP, verify all events captured with correct selectors, verify network bodies stored in blob store, verify round-trip (record → save → load → replay produces matching screenshots)
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
| Selector generation | `finder` (MIT) for SDK | Playwright's built-in for recorder | `finder` generates CSS selectors from DOM elements |
| Compression (SDK) | `fflate` (~3KB gz) | — | Tree-shakeable, browser-native fallback |
| Bundling | `esbuild` (SDK) + `tsup` (Node) | — | Standard toolchain |
| Size enforcement | `size-limit` | — | Sentry's approach, GitHub Action for PR comments |
