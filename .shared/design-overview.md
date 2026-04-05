# World Monitor -- Design Overview

> Real-time global intelligence dashboard -- AI-powered news aggregation, geopolitical monitoring, and infrastructure tracking in a unified situational awareness interface.

## Technology Stack

- **Language**: TypeScript (frontend + server + edge functions)
- **Build**: Vite (SPA bundler), Tauri 2.x (desktop shell, Rust)
- **Runtime**: Browser SPA, Vercel Edge Functions, Railway (Node.js relay), Tauri + Node.js sidecar (desktop)
- **Data**: Upstash Redis (cache), Convex Cloud (contact/waitlist), IndexedDB (client persistence)
- **RPC**: Protocol Buffers via sebuf framework (code generation for client stubs + server types)

---

## Entry Points

### 1. Browser SPA (`index.html` -> `src/main.ts`)

Primary entry point. Initializes Sentry, Vercel analytics, UTM tracking, then creates the `App` instance. The `App.init()` method runs an 8-phase boot sequence:

1. **Storage + i18n** -- IndexedDB init, language detection, locale loading (21 languages, RTL support)
2. **ML Worker** -- ONNX model preparation (MiniLM-L6 embeddings, sentiment, summarization, NER)
3. **Sidecar** -- Wait for desktop sidecar readiness (desktop builds only)
4. **Bootstrap** -- Two-tier concurrent hydration from `/api/bootstrap` (fast 3s + slow 5s timeouts) with IndexedDB fallback for offline starts
5. **Layout** -- `PanelLayoutManager` renders map container and panel grid
6. **UI** -- SignalModal, IntelligenceGapBadge, BreakingNewsBanner, CorrelationEngine
7. **Data** -- Parallel `loadAllData()` + viewport-conditional `primeVisiblePanelData()`
8. **Refresh** -- Variant-specific polling intervals via `RefreshScheduler` (exponential backoff, tab-pause, staggered flush)

**Key files**: `src/main.ts`, `src/App.ts`, `src/app/` (6 manager classes)

### 2. Vercel Edge Functions (`api/`)

71 files across 34 domain subdirectories (news, market, military, aviation, climate, cyber, maritime, seismology, etc.). Each is a self-contained JavaScript file deployed as a Vercel Edge Function. They cannot import from `src/` or `server/` at runtime -- only same-directory `_*.js` helpers and npm packages.

Per-domain edge function bundles are created via `server/gateway.ts` (`createDomainGateway(routes)`), which applies a 10-step pipeline: origin check, CORS, preflight, API key validation, rate limiting, route matching, POST-to-GET compat, handler execution, ETag generation, cache headers.

**Key files**: `api/`, `api/_cors.js`, `api/_rate-limit.js`, `api/_api-key.js`, `api/_relay.js`

### 3. Vercel Middleware (`middleware.ts`)

Runs before every request. Filters bot traffic (blocks crawlers on API/asset paths, allows social preview bots for OG endpoints). Detects site variant from hostname (`tech.worldmonitor.app` -> tech, `finance.worldmonitor.app` -> finance, etc.).

### 4. Railway Relay (`scripts/ais-relay.cjs`)

Long-running Node.js service on Railway. Runs:
- WebSocket proxy for AIS (ship tracking) stream
- Continuous seed loops for market data, aviation delays, GPSJAM, risk scores, UCDP events, positive events
- RSS proxy
- OREF alert polling

### 5. Tauri Desktop (`src-tauri/src/main.rs`)

Rust-based desktop shell (macOS, Windows, Linux). Manages:
- Platform keyring for secret storage (macOS Keychain, Windows Credential Manager)
- Node.js sidecar spawn and lifecycle (`src-tauri/sidecar/local-api-server.mjs`)
- Three trusted windows: main, settings, live-channels
- IPC commands for secret read/write, sidecar port probing

The sidecar dynamically loads Edge Function handler modules from `api/`, injects secrets via environment variables, and patches `globalThis.fetch` to force IPv4.

### 6. Seed Scripts (`scripts/seed-*.mjs`)

Cron-triggered scripts (Railway) that fetch upstream data, transform it, and write to Redis via `atomicPublish()` with lock-based stampede protection. Each write also records `seed-meta:<key>` for health monitoring.

### 7. Docker Container (`Dockerfile`, `docker/`)

Multi-arch image (amd64, arm64) serving the built SPA via nginx, with API proxy to upstream.

---

## Exit Points

| Exit Point | Destination | Purpose |
|---|---|---|
| `/api/*` fetch calls | Vercel Edge Functions | All panel data, bootstrap hydration, health checks |
| Desktop sidecar fetch | `http://127.0.0.1:<dynamic-port>/api/*` | Desktop offline-capable API via local Node.js |
| Cloud API fallback | `api.worldmonitor.app` | Desktop fallback when sidecar fails |
| Redis writes | Upstash Redis | Cache layer (both edge functions and seed scripts) |
| Convex mutations | Convex Cloud | Contact form submissions, waitlist registrations |
| Upstream API fetches | 30+ external sources | Finnhub, Yahoo, CoinGecko, FRED, ACLED, UCDP, GDELT, OpenSky, AIS, FIRMS, etc. |
| Sentry | sentry.io | Error tracking (browser) |
| Vercel Analytics | vercel.com | Page view analytics |
| IndexedDB writes | Client browser | Persistent cache, vector store, settings |
| WebSocket | AIS stream (via Railway) | Real-time ship tracking |

---

## Data Flow

### Primary Request Path (Browser)

```
User Interaction
    |
    v
Panel / Service calls fetch("/api/<domain>/v1/<rpc>")
    |
    v
[Desktop only: runtime.ts intercepts, routes to sidecar at 127.0.0.1:<port>]
    |
    v
Vercel Middleware (bot filter, variant detection)
    |
    v
Vercel Edge Function (api/<domain>/...)
    |
    v
server/gateway.ts pipeline:
  1. Origin check (403 if disallowed)
  2. CORS headers
  3. OPTIONS preflight
  4. API key validation
  5. Rate limiting (per-endpoint + global)
  6. Route matching (static Map O(1) + dynamic param scan)
  7. POST-to-GET compat
  8. Handler execution
  9. ETag (FNV-1a) + 304 Not Modified
  10. Cache tier headers (fast/medium/slow/static/daily/no-store)
    |
    v
server/worldmonitor/<domain>/v1/handler.ts
    |
    v
server/_shared/redis.ts -> cachedFetchJson()
  (concurrent requests coalesce into single upstream fetch + Redis write)
    |
    v
Upstash Redis (cache hit) OR Upstream API (cache miss -> fetch + cache)
    |
    v
JSON response -> Browser -> Panel.setContent(html) (debounced 150ms)
```

### Bootstrap Hydration Path

```
App.init() phase 4
    |
    v
fetchBootstrapData() -- two concurrent requests:
  - Fast tier (3s timeout): high-priority keys
  - Slow tier (5s timeout): lower-priority keys
    |
    v
/api/bootstrap -> Redis MGET (batch read of all cached keys)
    |
    v
hydrationCache Map (consumed once by panels via getHydratedData(key))
    |
    +--- On failure: IndexedDB persistent cache fallback (24h max age)
```

### Seed Pipeline (Background)

```
Railway cron / ais-relay.cjs seed loops
    |
    v
scripts/seed-*.mjs -> fetch upstream API
    |
    v
Transform / normalize data
    |
    v
atomicPublish() -> Redis SET with NX lock
  + seed-meta:<key> with { fetchedAt, recordCount }
    |
    v
/api/health.js reads seed-meta for staleness monitoring
```

### Client-Side ML Pipeline

```
News items arrive (via panels)
    |
    v
ml.worker.ts (ONNX via @xenova/transformers)
  - MiniLM-L6 embeddings
  - Sentiment analysis
  - Summarization
  - Named Entity Recognition
    |
    v
vector-db.ts (IndexedDB-backed vector store for semantic search)
    |
    v
analysis.worker.ts
  - Jaccard similarity clustering
  - Cross-domain correlation detection
    |
    v
CorrelationEngine adapters (military, escalation, economic, disaster)
    |
    v
CorrelationPanel, InsightsPanel, CrossSourceSignalsPanel
```

---

## Key Dependencies

### Frontend
- **deck.gl** + **maplibre-gl** -- WebGL flat map rendering (ScatterplotLayer, GeoJsonLayer, PathLayer, IconLayer, etc.)
- **globe.gl** + **three.js** -- 3D interactive globe
- **@xenova/transformers** -- ONNX inference in Web Workers
- **supercluster** -- Marker clustering on maps
- **PMTiles** -- Self-hosted vector basemap tiles
- **DOMPurify** -- HTML sanitization for panel content
- **papaparse** -- CSV parsing for data files
- **marked** -- Markdown rendering
- **@sentry/browser** -- Error tracking
- **@vercel/analytics** -- Page analytics
- **canvas-confetti** -- Celebration effects (happy variant)

### Server / Edge
- **Upstash Redis** (REST API) -- Cache layer with stampede protection
- **Vercel Edge Runtime** -- Serverless edge function execution
- **sebuf** (custom protobuf framework) -- Code generation for RPC client/server stubs

### Desktop
- **Tauri 2.x** -- Rust desktop shell (macOS, Windows, Linux)
- **Node.js sidecar** -- Local API server loading edge function handlers

### Tooling
- **Vite** -- Build + dev server
- **Biome** -- Linter/formatter
- **Playwright** -- E2E testing with visual regression
- **buf** -- Protobuf code generation
- **Husky** -- Git hooks (pre-push: typecheck, esbuild bundle check, lint)

---

## Architectural Abstractions

### Variant System

5 site variants from a single codebase: `full` (world), `tech`, `finance`, `commodity`, `happy`. Detected by hostname at runtime or `VITE_VARIANT` at build time. Controls: default panels, map layers, refresh intervals, theme, UI text, feed selection. Configuration defined in `src/config/variant.ts` and `src/config/variant-meta.ts`. Variant change resets all user settings to variant defaults.

### Panel Component Model

All 105 panel components extend a `Panel` base class. Panels:
- Render via `setContent(html)` (debounced 150ms)
- Use event delegation on a stable `this.content` element
- Support resizable row/col spans persisted to localStorage
- Are instantiated by `PanelLayoutManager` based on variant config

### Domain Gateway Factory

`server/gateway.ts` provides `createDomainGateway(routes)` -- a factory that generates per-domain edge function bundles. This splits the server code so Vercel bundles only one domain per function, reducing cold-start cost by approximately 20x. Each bundle gets the full 10-step pipeline (CORS, auth, rate limiting, caching, ETag) for free.

### Cache Architecture (Four Tiers)

1. **Bootstrap seed** -- Railway writes to Redis on schedule
2. **In-memory** -- Per Vercel instance, short TTL
3. **Redis** -- Cross-instance via Upstash, `cachedFetchJson()` coalesces concurrent misses
4. **Upstream fetch** -- Result cached back to Redis + `seed-meta` written for health monitoring

Cache tiers per RPC: fast (300s), medium (600s), slow (1800s), static (7200s), daily (86400s), no-store.

### Proto/RPC Contract System

Protocol Buffers via the sebuf framework define the API contract:
- `proto/` definitions with `(sebuf.http.config)` annotations mapping RPCs to HTTP verbs/paths
- `buf generate` produces `src/generated/client/` (TypeScript RPC client stubs) and `src/generated/server/` (TypeScript server types)
- CI enforces generated code freshness via `.github/workflows/proto-check.yml`

### State Management

No external state library. `AppContext` is a central mutable object holding: map references, panel instances, panel/layer settings, all cached data, in-flight request tracking, and UI component references. URL state syncs bidirectionally via `src/utils/urlState.ts` (debounced 250ms).

### Smart Polling

`RefreshScheduler` wraps `startSmartPollLoop()` which supports: exponential backoff (max 4x), viewport-conditional refresh (only if panel is near viewport), tab-pause (suspend when document is hidden), and staggered flush on tab visibility restore (150ms delays to avoid thundering herd).

### Desktop Fetch Patching

`installRuntimeFetchPatch()` replaces `window.fetch` in the Tauri renderer. All `/api/*` requests route to the local Node.js sidecar with `Authorization: Bearer <token>` (5-min TTL from Tauri IPC). On sidecar failure, requests transparently fall back to the cloud API.

---

## Directory Reference

```
.
├── api/                    Vercel Edge Functions (71 files, 34 domain dirs)
│   ├── _*.js               Shared helpers (CORS, rate-limit, API key, relay)
│   └── <domain>/           aviation, climate, conflict, cyber, market, military, ...
├── blog-site/              Static blog (built into public/blog/)
├── convex/                 Convex backend (contact form, waitlist, counters)
├── data/                   Static reference data (conservation, renewable, happiness)
├── docker/                 Dockerfile + nginx config
├── docs/                   Mintlify documentation site
├── e2e/                    Playwright E2E specs
├── proto/                  Protobuf service definitions (sebuf framework)
├── scripts/                Seed scripts, ais-relay, build helpers
├── server/                 Server-side TypeScript (bundled into edge functions)
│   ├── _shared/            Redis, rate-limit, LLM, caching utilities
│   ├── gateway.ts          Domain gateway factory (10-step pipeline)
│   ├── router.ts           O(1) static + dynamic route matching
│   └── worldmonitor/       29 domain handler directories
├── shared/                 Cross-platform JSON configs (stocks, crypto, RSS domains)
├── src/                    Browser SPA (TypeScript)
│   ├── app/                6 orchestration managers (layout, data, refresh, events, search, country-intel)
│   ├── bootstrap/          Chunk reload recovery
│   ├── components/         105 panel subclasses + DeckGLMap + GlobeMap
│   ├── config/             Variant, panel, layer, feed, market configurations
│   ├── generated/          Proto-generated client/server stubs (DO NOT EDIT)
│   ├── locales/            i18n translation files (21 languages)
│   ├── services/           100+ domain service modules
│   ├── types/              TypeScript type definitions
│   ├── utils/              Shared utilities (circuit-breaker, theme, URL state)
│   └── workers/            Web Workers (analysis, ML, vector DB)
├── src-tauri/              Tauri 2.x desktop shell (Rust)
│   ├── src/main.rs         Keyring, IPC, window management
│   └── sidecar/            Node.js local API server
└── tests/                  Unit/integration tests (node:test)
```
