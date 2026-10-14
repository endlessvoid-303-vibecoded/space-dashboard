# Space Dashboard — Planning Spec (SvelteKit Edition)

A reactive 3D globe visualising global launch sites — each marked by the operating
company/agency logo — with live upcoming-launch data. Powered by The Space Devs
**Launch Library 2 API** (v2.3.0).

> **Revised from the original spike.** The spike locked Next.js / Globe.gl. The
> user has requested **Svelte** instead. This spec overrides all Next.js
> references with **SvelteKit** while keeping every other decision intact.

---

## 1. Vision

A rotating 3D globe of Earth. At every active launch pad, a sprite billboard
shows the **logo of the company/agency that operates the pad** (SpaceX, NASA,
Blue Origin, ESA, etc.). Hover reveals the pad, its location, the operator, and
the **next upcoming launch**. Click (desktop) / tap (mobile) opens a side panel
with full launch details and any embedded webcast.

> **“For nerds” scope:** Raw data inspection, rocket specs, mission payloads,
> trajectory info, multiple simultaneous views (globe + list + timeline), live
> countdown, status badges, and webcast embeds.

---

## 2. Tech-stack decision (locked)

| Layer | Choice | Why |
|---|---|---|
| **Globe** | **Globe.gl** (Three.js via standalone call) | Purpose-built 3D globe; native lat/lon sprite markers with custom **texture = company logo**; hover/click; responsive WebGL; 251 sprites trivial for it. Used via `Globe()` standalone function inside Svelte `onMount`. |
| **Framework** | **SvelteKit** (App Router style file-based routing) | Reactive by default (no virtual DOM churn for 60-fps globe); trivial to wrap imperative Globe.gl in `onMount`; server endpoints double as LL2 caching proxy; first-class Vercel adapter. |
| **Styling** | **Tailwind CSS 3** | Rapid prototyping; dark/space theme tokens; responsive grid for the “nerd layout”. |
| **API client** | **TypeScript + fetch + zod** | Typed responses; small; zod guards LL2’s messy nested JSON. |
| **Server cache** | SvelteKit server endpoints (`/api/`) + in-memory Map + LRU eviction | Acts as a caching proxy so all browser clients share a single LL2 rate-limit budget. |
| **Client cache** | localStorage (ETag-keyed) | Offline-tolerable; avoids re-fetch when the tab is reopened. |
| **Rate limiting** | `api-throttle` probe + exponential backoff + budget-aware scheduler | 15 req/hour per IP — must be a good citizen. |
| **Testing** | **Vitest** (unit) + **Playwright** (E2E) | Vitest is Vite-native (zero-config with SvelteKit); Playwright for globe render + interaction. |
| **Deploy** | **Vercel** (`@sveltejs/adapter-vercel`) | Zero-config from GitHub; global edge; server endpoints become serverless functions. |
| **Lint/Format** | ESLint + Prettier + `svelte-preprocess` + `svelte-eslint-parser` | Code-quality gate for PRs. |

### Why not plain React?
Globe.gl ships a React wrapper, but the user explicitly asked for Svelte.
SvelteKit gives us server endpoints (crucial for the caching-proxy pattern) and
a smaller bundle than Next.js for this use-case.

---

## 3. API baseline (verified)

Base: `https://ll.thespacedevs.com/2.3.0/`

| Endpoint | Count | Pages (limit=100) | Used for | Key fields |
|---|---|---|---|---|
| `pads/` | 251 | 3 | **Globe markers** — coords + operator logos | `id,name,latitude,longitude,active,location.name,location.country.name,agencies[].{id,name,abbrev,type,logo.image_url}` |
| `launches/upcoming/` | 472 | 5 | Next launch per pad | `id,name,net,pad.id,launch_service_provider.{name,abbrev,type,logo.image_url},rocket.configuration.name,vid_urls,status` |
| `agencies/` | 350 | 4 | Fallback / enriched operator info | `id,name,abbrev,type,logo.image_url,country` |
| `celestial_bodies/` | — | 1 | Earth texture | `image.image_url` (CC-BY NASA blue marble) |
| `api-throttle/` | — | 0 | Rate-limit budget | `your_request_limit` (15), `limit_frequency_secs` (3600), `current_use`, `next_use_secs` |

**Max page size:** The API enforces `limit=100` (verified: `limit=200` → 429).

### Rate-limit budget per full refresh

| Fetch | Requests |
|---|---|
| Pads (3 pages × 100) | 3 |
| Upcoming launches (5 pages × 100) | 5 |
| Earth texture (celestial_bodies) | 1 |
| **Total per full refresh** | **9** |

With a 15 req/hour budget, this leaves **6 requests** for error recovery / ad-hoc
refreshes. The server-side cache TTL is tuned to stay well within budget:

| Resource | Cache TTL | Refresh frequency | Daily budget used |
|---|---|---|---|
| Earth texture | ∞ (cache forever) | once | 1 (once ever) |
| Pads | 24 h | once/day | 3 |
| Upcoming launches | 2 h | 12×/day | 60 → **throttled!** |

**Correction:** 5-page launches refresh at 2h TTL would use 5×12=60 requests/day
= 9/hour on average, but bursts would exceed 15/hour. Revised strategy:

| Resource | Cache TTL | Rationale |
|---|---|---|
| Earth texture | ∞ | Never changes. |
| Pads | 24 h | Pads rarely change; 3 req once/day. |
| Upcoming launches | 6 h | Launches shift often enough; 5 req × 4 = 20/day, ~3–5/hour spread. |
| Agencies | 24 h | Logos rarely change; 4 req once/day (batch with pads refresh). |
| **Budget check** | | 3+5+1 = 9 per 6-hour window; 4 such windows/day = 36/day but never > 9 in any single hour. |

The server checks `api-throttle` before each burst; if `current_use >= 12`, it
defers the refresh and serves stale cache with an `X-Stale: true` header.

---

## 4. Data model (TypeScript)

```ts
// Core types — all validated at the API-client boundary with zod

GlobePad {
  id: number
  name: string            // e.g. "Launch Complex 39A"
  lat: number             // pad.latitude
  lon: number             // pad.longitude
  location: string        // location.name
  country: string         // location.country.name
  operators: Operator[]   // pad.agencies (usually 1-2)
  nextLaunch: Launch|null // earliest upcoming launch at this pad
}

Operator {
  id: number
  name: string
  abbrev: string | null
  type: string | null     // "Government", "Commercial", "Startup"
  country: string | null
  logoUrl: string | null  // logo.image_url — null → fallback to abbrev badge
}

Launch {
  id: string
  name: string            // "Falcon 9 Block 5 | Starlink Group 15-25"
  net: Date               // "2026-10-13T00:28:11Z" — converted to Date
  netPrecision: string    // from launch window config
  status: LaunchStatus    // upcoming, hold, go, launched, etc.
  pad: number             // pad.id FK
  provider: Operator      // launch_service_provider
  rocket: string          // rocket.configuration.name
  mission: string | null
  webcastUrl: string | null
  missionPatchUrl: string | null
  image: string | null
}

LaunchStatus =
  | "Go for Launch"
  | "Launch Imminent"
  | "Launch Success"
  | "Hold"
  | "To Be Determined"
  | "TBD"
  | "Failure"

// API response wrappers
LL2ListResponse<T> = {
  count: number
  next: string | null
  previous: string | null
  results: T[]
}
```

### Next-launch enrichment algorithm

```ts
// fetch all upcoming launches (5 pages), group by pad.id, pick earliest net
function enrichPadsWithNextLaunch(pads: GlobePad[], launches: Launch[]): GlobePad[] {
  const byPad = groupBy(launches, l => l.pad)
  for (const pad of pads) {
    const padLaunches = byPad.get(pad.id) || []
    pad.nextLaunch = padLaunches.reduce(
      (earliest, l) => !earliest || l.net < earliest.net ? l : earliest,
      null as Launch | null
    )
  }
  return pads
}
```

---

## 5. Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Browser (SvelteKit client)            │
│                                                          │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │  Globe.svelte│  │  SidePanel.svelte│  │  LaunchList.svelte│ │
│  │(Globe.gl in  │  │(pad + launch │  │(table view +    │ │
│  │ onMount)     │  │ details)      │  │ countdown column) │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ │
│         │   polls every  │                      │          │
│         ▼   30s          ▼                      │          │
│  ┌──────────────┐  ┌────────────┐              │          │
│  │ /api/pads    │  │ /api/launches│            │          │
│  │ (/store cache)│  │ (/store cache)│          │          │
│  └──────┬───────┘  └──────┬───────┘            │          │
│         │                │                      │          │
└─────────┼────────────────┼────────────────────┼──────────┘
          │                │                    │
┌─────────▼────────────────▼────────────────────▼──────────┐
│  SvelteKit Server (Vercel serverless / edge)             │
│                                                            │
│  ┌──────────────────────────────────────────┐              │
│  │  In-memory Cache (Map<string, CachedRes>) │             │
│  │  - Key: endpoint+etag                    │              │
│  │  - Value: {data, etag, expiresAt, stale} │              │
│  │  - LRU eviction, max 50 entries          │              │
│  └──────────┬───────────────────────────────┘              │
│            │ checks api-throttle before fetch              │
│            ▼                                               │
│  ┌──────────────────────────────────────────┐              │
│  │  LL2 API Client (fetch + zod + backoff)  │              │
│  │  - Probes api-throttle/ every burst      │              │
│  │  - On 429: exponential backoff + jitter  │              │
│  │  - Sends If-Modified-Since / If-None-Match│              │
│  └──────────────────────────────────────────┘              │
│                                                            │
│  ┌──────────────────────────────────────────┐              │
│  │  /api/pads      → cache → LL2 pads/      │              │
│  │  /api/launches  → cache → LL2 launches/  │              │
│  │  /api/agencies  → cache → LL2 agencies/  │              │
│  │  /api/earth     → cache → LL2 celestial/ │              │
│  └──────────────────────────────────────────┘              │
└────────────────────────────────────────────────────────────┘
```

### Live-updating strategy

“Live” here means:
1. **Countdown timer** — runs purely client-side (`setInterval` + Svelte reactive
   `$:` statement) every second. No API call needed.
2. **Pad data** — fetched once at build/mount, cached server-side for 24 h.
3. **Launch data** — refreshed server-side every 6 h from LL2. All clients get
   fresh data within 30 s of the server-side refresh (next poll cycle).
4. **Stale-while-revalidate** — if the server cache is expired, it returns stale
   data immediately (header `X-Cache: stale`) while refreshing in the background.

This means **the frontend polls at 30 s intervals without ever hitting the LL2
rate limit** — only the server-side cache layer talks to LL2, and it does so at
most 9 requests per 6-hour window.

---

## 6. Visualization spec (detailed)

### Globe (`Globe.svelte`)

```svelte
<script>
  import { onMount, onDestroy } from 'svelte'
  import Globe from 'globe.gl'
  import { selectedPad } from '$lib/stores'
  import type { GlobePad } from '$lib/types'

  let container: HTMLDivElement
  let globe: any
  let pads: GlobePad[] = $state([])

  // Fetch enriched pads from our own caching proxy
  onMount(async () => {
    const res = await fetch('/api/pads')
    pads = await res.json()

    globe = Globe()(container)
      .globeImageUrl('https://raw.githubusercontent.com/.../earth-night.jpg')
      .backgroundColor('rgba(0, 0, 0, 0)')
      .showGraticules(false)
      .showPolarCaps(false)
      .pointsData(pads)
      .pointLat('lat')
      .pointLng('lon')
      .pointLabel(d => `${d.operators?.[0]?.abbrev || '???'} · ${d.name}`)
      .pointLabelColor(() => 'rgba(255,255,255,0.9)')
      .pointLabelAltitude(0.01)
      .pointColor(() => 'color: #ff6b35')  // company brand orange
      .pointAltitude(0.01)  // hover offset
      .pointRadius(0.55)
      .pointResolution(32)
      .pointHoverColor(() => 'color: #ffffff')
      .onPointHover(d => selectedPad.set(d))
      .onPointClick(d => {
        selectedPad.set(d)
        // also scroll to side panel on mobile
      })

    // Auto-rotate when idle
    let autoRotate = true
    container.addEventListener('pointerenter', () => autoRotate = false)
    container.addEventListener('pointerleave', () => {
      autoRotate = true
      const i = setInterval(() => {
        if (globe) globe.controls().autoRotate()
      }, 3000)
      onDestroy(() => clearInterval(i))
    })
  })
</script>

<div bind:this={container} class="globe-canvas w-full h-full" />
```

**Key Globe.gl settings:**
- `pointLat`/`pointLng` → pad coordinates
- Custom sprite: use `pointLabel` for hover text; for the **logo**, we use a
  **custom Three.js sprite material** via `pointColor` with a canvas-generated
  texture from `operator.logoUrl`. Actually, Globe.gl's `pointsData` supports a
  `pointColor` accessor that can return a hex string. To render **logo sprites**
  (not colored dots), we use Globe.gl's `customThreeObject` or `pointAltitude` +
  CSS2D markers.

**Logo rendering approach (chosen):** Use Globe.gl's **CSS2D marker layer**
(`pointLabel` + custom `labelDotColor`/`labelBackgroundColor`) with HTML `<img>`
tags. Each marker is an `<img>` element positioned absolutely over the canvas,
synced to the 3D point's screen projection. This gives us:
- Real `<img>` tags (native browser loading, caching, error fallbacks)
- CSS for hover effects, sizing, fallbacks
- No WebGL sprite texture limits

Reference: Globe.gl `pointLabel` renders an HTML `<div>` at each point. We can
inject `<img src="logoUrl">` as the label content. The `pointLabelColor` function
sets the image background. This is the cleanest approach for logo sprites.

### Side panel (`SidePanel.svelte`)

Rendered when `selectedPad` is non-null:
- Pad name, location, country
- Operator card: logo (large), name, type badge, country flag
- Next launch card:
  - Rocket name + configuration
  - Mission name + description
  - `net` date + humanized countdown (`"2 days, 3 hours, 12 min to launch"`)
  - Status badge (green "Go for Launch", red "Hold", etc.)
  - Mission patch thumbnail (if available)
  - Webcast button → opens YouTube embed in modal
  - Raw data toggle (JSON view for the "nerd" audience)

### Countdown timer

```svelte
<script>
  // Pure client-side — no API calls after initial load
  let { targetDate } = $props<{ targetDate: Date }>()
  let now = new Date()
  let interval = setInterval(() => now = new Date(), 1000)
  
  const diff = $derived(() => {
    const d = new Date(targetDate) - now
    return d > 0 ? formatDuration(d) : "Liftoff!"
  })
</script>
<span class="countdown">{diff()}</span>
```

---

## 7. File structure

```
space-dashboard/
├── PLAN.md                     ← THIS FILE (master spec)
├── README.md
├── package.json
├── svelte.config.js            ← adapter-vercel
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.cjs
├── postcss.config.cjs
├── .eslintrc.cjs
├── .prettierrc
├── .gitignore
├── static/
│   ├── favicon.ico
│   ├── earth-day.jpg           ← Earth texture (NASA CC-BY, cached locally)
│   └── earth-night.jpg         ← Night lights texture
├── src/
│   ├── app.html                 ← Root template (themed for dark)
│   ├── app.css                  ← Tailwind base + globe overrides
│   ├── routes/
│   │   ├── +layout.svelte       ← Header + dark mode toggle
│   │   ├── +page.svelte         ← Main dashboard (globe + side panel)
│   │   ├── +page.server.ts      ← Server-side data loader (initial pads)
│   │   ├── api/
│   │   │   ├── pads/+server.ts    ← Caching proxy to LL2 pads
│   │   │   ├── launches/+server.ts ← Caching proxy to LL2 launches
│   │   │   ├── agencies/+server.ts ← Caching proxy to LL2 agencies
│   │   │   └── earth/+server.ts   ← Earth texture URL
│   │   └── launch/[id]/+page.svelte ← Individual launch detail page
│   ├── lib/
│   │   ├── api/
│   │   │   ├── client.ts           ← LL2 API client (fetch + zod + retry)
│   │   │   ├── schemas.ts          ← zod schemas for all LL2 entities
│   │   │   ├── types.ts            ← TypeScript interfaces
│   │   │   ├── cache.ts            ← Server-side cache (Map + LRU)
│   │   │   ├── rateLimiter.ts      ← api-throttle probe + budget scheduler
│   │   │   └── enricher.ts         ← enrichPadsWithNextLaunch() + grouping
│   │   ├── globe/
│   │   │   ├── Globe.svelte        ← Globe.gl wrapper (onMount + CSS2D labels)
│   │   │   ├── LogoSprite.js       ← Canvas texture generator for WebGL sprites
│   │   │   └── GlobeControls.svelte ← Auto-rotate, zoom, gesture toggle
│   │   ├── components/
│   │   │   ├── SidePanel.svelte     ← Pad + launch detail side sheet
│   │   │   ├── PadMarker.svelte     ← Reusable logo badge (fallback to abbrev)
│   │   │   ├── Countdown.svelte     ← Live countdown timer (client-only)
│   │   │   ├── StatusBadge.svelte   ← Color-coded status display
│   │   │   ├── WebcastModal.svelte  ← YouTube embed
│   │   │   ├── LaunchTable.svelte   ← List view with sortable columns
│   │   │   ├── Timeline.svelte      ← Upcoming launches timeline
│   │   │   └── Roster.svelte        ← Astronauts in space
│   │   ├── stores/
│   │   │   ├── selectedPad.ts       ← Writable store for selected pad
│   │   │   ├── pads.ts              ← Pads data store (server→client)
│   │   │   └── launches.ts          ← Launches data store
│   │   └── utils/
│   │       ├── format.ts            ← Date / duration / humanize helpers
│   │       └── geo.ts               ← Coordinate / distance helpers
│   └── hooks/
│       └── handle.ts                ← SvelteKit hook (optional: CSP, logging)
├── tests/
│   ├── api/
│   │   ├── client.test.ts          ← Full pagination, 429 backoff, caching
│   │   ├── enricher.test.ts        ← Earliest-net-per-pad join logic
│   │   ├── schemas.test.ts         ← zod validation (malformed/missing logo)
│   │   └── rateLimiter.test.ts     ← Budget scheduling, throttle probing
│   ├── components/
│   │   ├── Globe.test.ts           ← Render + marker count + logo sprites
│   │   ├── SidePanel.test.ts       ← Pad detail display
│   │   ├── Countdown.test.ts       ← Time formatting + edge cases
│   │   └── StatusBadge.test.ts     ← Status → color mapping
│   ├── integration/
│   │   └── data-flow.test.ts       ← LL2 → server cache → client store → globe
│   └── e2e/
│       └── dashboard.spec.ts       ← Playwright: globe renders, click → side panel
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                  ← Lint + test + type-check on PR
│   │   └── deploy.yml              ← Deploy to Vercel on merge to main
│   └── dependabot.yml
└── vite.config.ts
```

---

## 8. Kanban card breakdown

The existing spike card (`t_02f13959`) is **done**. The repo-creation card
(`t_4d111236`) was **blocked on a typo** — the org was `endlessvoid-33-vibecoded`
but the real org is `endlessvoid-303-vibecoded`. Auth is verified working.
Repo has been created and this card can be **unblocked / completed**.

The two existing implementation cards (`t_0eafec43`, `t_87bc609c`) used
Next.js terminology. They are **updated** to reference SvelteKit and split into
finer-grained sub-cards:

### Phase 0 — Repo & scaffolding (priority 9)
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_4d111236` | **done** | manager | Create repo + protected main (✅ created, auth verified, org typo fixed) |
| `t_1c572d91` | ready | developer | Scaffold SvelteKit: `npm create svelte@latest` + TypeScript + Tailwind + ESLint + Prettier |
| `t_c70adc61` | ready | developer | Set up branch protection + GitHub Actions CI workflow |

### Phase 1 — Data layer (priority 8) `[t_0eafec43 umbrella, running]`
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_773dcad7` | todo | developer | zod schemas for all LL2 entities (pads, launches, agencies, celestial_bodies) |
| `t_5f9b70e6` | todo | developer | LL2 API client: `fetchPads()`, `fetchUpcomingLaunches()`, `fetchAgencies()` (paginated) |
| `t_a8814827` | todo | developer | Rate limiter: api-throttle probe + exponential backoff + budget scheduler |
| `t_18b6e54a` | todo | developer | Server-side cache: Map + LRU eviction + stale-while-revalidate |
| `t_c79e0a5b` | todo | developer | Enricher: `enrichPadsWithNextLaunch()` — group launches by pad, earliest net |
| `t_5a002993` | todo | developer | SvelteKit `/api/pads`, `/api/launches`, `/api/agencies`, `/api/earth` endpoints |

### Phase 2 — Globe + main view (priority 7) `[t_87bc609c umbrella, running]`
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_abe20e9d` | todo | developer | Main dashboard page: `+page.svelte` + `+page.server.ts` data loading |
| `t_ac2d2d59` | todo | developer | `Globe.svelte`: Globe.gl scene, Earth texture, auto-rotate, gestures |
| `t_a61fe877` | todo | developer | Logo sprites: CSS2D marker labels with `<img>` logos, fallback to abbrev badge |
| `t_58d0f34f` | todo | developer | Hover tooltip: pad name + operator + next launch + countdown |
| `t_b3cef645` | todo | developer | `SidePanel.svelte`: pad details, operator card, launch card, webcast modal |
| `t_14125d4c` | todo | developer | `Countdown.svelte`: live client-side countdown (no API after initial load) |
| `t_819f0302` | todo | developer | `StatusBadge.svelte`: color-coded launch status |
| `t_51145682` | todo | developer | Responsive layout: mobile side panel overlay, grid for nerd panels |

### Phase 3 — Nerdy extras (priority 6)
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_4427771d` | todo | developer | `LaunchTable.svelte`: sortable table view with countdown column |
| `t_774d7927` | todo | developer | `Timeline.svelte`: upcoming launches on a horizontal timeline |
| `t_8e7a114f` | todo | developer | `RawDataToggle`: JSON inspection panel for pad/launch objects |
| `t_39a8bbd6` | todo | developer | `/launch/[id]` detail page (deep link) |
| `t_67ff5522` | todo | developer | Rocket specs panel: rocket configuration details from LL2 |
| `t_dd3ae358` | todo | developer | Dark/light mode toggle (space theme) |
| ⏸️ **DEFERRED** | — | — | Astronaut roster (see Clarification #3) |

### Phase 4 — Testing & quality (priority 5)
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_5ee9ef10` | todo | developer | Vitest config + API client unit tests (pagination, 429 backoff, cache) |
| `t_ac9dbaf4` | todo | developer | Component tests (Globe, SidePanel, Countdown, StatusBadge) via @testing-library/svelte |
| `t_506a536a` | ready | developer | Playwright E2E: globe renders, click marker → side panel |
| `t_0ffa3b0b` | todo | developer | Pre-commit hooks: lint + type-check + test (husky + lint-staged) |

### Phase 5 — Deploy (priority 4)
| Card | Status | Owner | Description |
|---|---|---|---|
| `t_4cbbb68f` | ready | developer | Vercel deploy config: `@sveltejs/adapter-vercel` + edge cache headers |
| `t_81b39695` | ready | developer | README.md: setup instructions + architecture diagram |

### Dependencies (critical path)

```
t_4d111236 (done) → t_1c572d91 (scaffold) → t_773dcad7 (schemas) → t_5f9b70e6 (client)
  → t_a8814827 (rate limiter) → t_18b6e54a (cache) → t_c79e0a5b (enricher)
  → t_5a002993 (endpoints) → t_0eafec43 (API umbrella, running)

t_0eafec43 → t_abe20e9d (main page) → t_ac2d2d59 (Globe) → t_a61fe877 (logos)
  → t_58d0f34f (tooltip) → t_b3cef645 (SidePanel) → t_14125d4c (Countdown)
  → t_819f0302 (StatusBadge) → t_51145682 (Responsive)

Phase 3 cards (t_4427771d–t_dd3ae358) depend on t_abe20e9d (main page)
Phase 4 cards depend on Phase 1 + Phase 2 implementation
Phase 5 cards depend on Phase 4
```

---

## 9. Acceptance criteria (overall)

1. **`npm install && npm run dev`** serves a localhost dashboard with a rotating
   3D globe.
2. **≥1 company logo sprite** is correctly placed on the globe surface at the
   correct lat/lon.
3. **Hover** shows pad name + operator abbrev + next launch + countdown.
4. **Click** opens a side panel with full launch details + webcast link.
5. **Countdown timer** updates every second (client-side, no API calls).
6. **Rate limit respected** — server-side cache ensures ≤ 9 LL2 API calls per
   6-hour refresh window; 429 triggers exponential backoff.
7. **`npm test`** passes — unit tests cover pagination, enrichment, caching,
   rate-limit; E2E covers globe render + interaction.
8. **`npm run build`** produces a production bundle < 5 MB initial JS.
9. **`eslint . --max-warnings 0`** passes.
10. Deploys to Vercel on merge to `main`.

---

## 10. Clarification items (need user input)

| # | Item | Impact | Default if no answer |
|---|---|---|---|
| 1 | **Repo description**: Should the repo be listed under organization `endlessvoid-303-vibecoded` or a different one? | Naming/deployment | Using `endlessvoid-303-vibecoded` (verified org with admin access) |
| 2 | **Globe texture source**: Should we use LL2's `celestial_bodies` Earth image (CC-BY NASA) or bundle a static NASA blue-marble texture locally? | Offline support / rate budget | Bundle locally (saves 1 API call / day, always available) |
| 3 | **Astronaut/Roster features**: Phase 3 includes astronauts and rocket specs. Should these be in scope or deferred? | Scope | Included but lowest priority; can be cut if time-constrained |
| 4 | **Webcast embed policy**: When a launch has a YouTube webcast, should it auto-embed or require a click? | UX | Click-to-embed (avoids autoplay, better for mobile) |
| 5 | **Dark mode default**: Space theme dark by default with a toggle? | UX | Dark default, toggle to light |
| 6 | **Vercel deployment**: Who owns the Vercel project / should we auto-deploy from GitHub? | Deploy | Auto-deploy via GitHub Action on merge to main |
