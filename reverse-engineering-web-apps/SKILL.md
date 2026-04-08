---
name: reverse-engineering-web-apps
description: Use when mapping features, data models, or APIs of a live web app without access to source code — especially SPAs, Flutter Web, or React apps behind auth walls. Symptoms include: need to clone/replicate a product, competitive analysis, security audit, or building compatible tooling.
---

# Reverse Engineering Web Apps

## Overview

Live apps leak their own internals: network requests expose DB schema, localStorage reveals client state, static assets expose data models, and API keys are visible in request headers. The goal is to pivot quickly past rendering barriers toward data extraction.

**Core principle:** Don't fight the renderer — go around it. DOM manipulation fails on CanvasKit/Flutter/canvas-heavy apps. Network and storage always work.

## When to Use

- Building a product based on competitor/reference app
- Security audit of client-side exposure
- Building import/export compatibility
- Mapping an app you own but lost source code for

**Not for:** Apps where you have source access (just read the code).

---

## The Workflow

```
Static files → Screenshot UI → Try DOM → Pivot to Network → localStorage → API calls → Compile
```

### Phase 1 — Static Recon (no browser needed)

```bash
# If you have extracted files (bundle, APK, electron app):
rg -o '"[a-z_]+\.[a-z]+\.co/[^"]*"' bundle.js   # extract URLs
rg -o 'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+' bundle.js  # JWTs/keys
```

Fetch known public assets from the live app:
```js
// In browser evaluate or via curl:
fetch('/assets/AssetManifest.bin.json').then(r=>r.json())  // Flutter
fetch('/asset-manifest.json').then(r=>r.json())             // CRA/Vite
```

### Phase 2 — Screenshot to Map UI

Take a full-page screenshot first. Read it carefully before touching DOM:
- Map nav sections, sidebar categories, modals
- Note bottom bars, floating panels, CTAs
- Record exact button labels (needed for DOM queries later)

```js
// Playwright:
await page.screenshot({ fullPage: true, path: 'map.png' })
```

### Phase 3 — DOM Interaction (expect failure on canvas apps)

Try standard accessibility snapshot + clicks. **Flutter CanvasKit / PixiJS / Three.js apps will fail here** — the flt-glass-pane or canvas intercepts events. Diagnosis:

```js
// In evaluate:
document.elementFromPoint(x, y).tagName
// Returns: FLUTTER-VIEW or CANVAS → pivot to Phase 4
// Returns: BUTTON, DIV, A → proceed normally
```

For Flutter specifically:
- `flutter-view` receives events but ignores `dispatchEvent` (uses internal Dart event loop)
- Accessibility tree only shows `flt-semantic-node-*` with 0-1 elements
- **Wheel events, pointer events, keyboard events all silently fail**

### Phase 4 — Network Interception (primary for canvas apps)

Start capturing all non-static requests as soon as the app loads:

```js
// Playwright MCP:
browser_network_requests({ static: false, requestHeaders: true, filter: 'api|rest|graphql|supabase|firebase' })
```

From network requests, extract:
- **API base URL** + all endpoint paths
- **Auth scheme** (Bearer, Cookie, API key header)
- **apikey / x-api-key** from request headers — these are often hardcoded anon keys
- **Query params** that reveal table/column names (e.g. `?select=id,title,user_id`)
- **RPC/function names** (e.g. `/rpc/record_daily_visit`)
- **GraphQL operations** (operation names reveal business logic)

### Phase 5 — localStorage Extraction

Client-side state in localStorage is the richest source of **data model** info:

```js
// Dump all keys and parse JSON values:
Object.fromEntries(
  Object.keys(localStorage).map(k => {
    try { return [k, JSON.parse(localStorage.getItem(k))] }
    catch { return [k, localStorage.getItem(k)] }
  })
)
```

Look for:
- Auth tokens (`sb-*-auth-token`, `firebase:authUser`, `__clerk_db_jwt`)
- Cached API responses (reveal full object schema)
- Feature flags, user config, design state
- Session/analytics metadata

### Phase 6 — Direct API Calls with Real Credentials

Use the real auth token + apikey from Steps 4-5 to call the API directly:

```js
// Supabase example (common pattern):
const session = JSON.parse(localStorage.getItem('sb-PROJECT-auth-token'));
const token = session.access_token;
const apikey = '<from request headers>';
const h = { Authorization: `Bearer ${token}`, apikey };

// Probe schema by requesting all columns:
fetch(`${base}/tablename?select=*&limit=1`, { headers: h }).then(r => r.json())
// Column-not-found errors reveal actual column names
```

For REST APIs, probe unknown endpoints systematically:
```js
// Try common table names:
['users','profiles','designs','subscriptions','projects'].forEach(async t => {
  const r = await fetch(`${base}/${t}?limit=1`, { headers: h });
  console.log(t, r.status); // 200=exists, 404=not found, 400=wrong params
})
```

### Phase 7 — Compile the Map

Structure the output as:
1. **UI Layout** — nav, sidebars, canvas, modals
2. **Component/entity types** — all types visible in sidebar/dropdowns
3. **Data model** — full field schema per entity (from localStorage state)
4. **API schema** — tables, RPCs, Edge Functions, endpoints
5. **Business logic** — subscription tiers, credit system, feature flags
6. **AI/automation features** — prompts, schemas, models used
7. **Unknowns** — server-side logic you couldn't observe

---

## Quick Reference

| Signal | Where to look | What it reveals |
|--------|--------------|-----------------|
| Column names | Network URL query params | DB schema |
| API key | Request headers (apikey, x-api-key) | Auth for direct calls |
| Full object schema | localStorage cached state | All entity fields |
| Feature flags | localStorage / network response | Subscription gating |
| Prompts | Extracted bundle strings | AI system design |
| Canvas schema | Static JSON assets (/assets/*.json) | Data model |
| RPC names | POST /rpc/* endpoints | Business logic functions |
| Analytics events | GA/Mixpanel collect calls | User journey map |

---

## Common Mistakes

**Spending too long on DOM interaction for canvas apps**
→ Diagnose with `document.elementFromPoint` first. If `FLUTTER-VIEW` or `CANVAS`, skip to Phase 4 immediately.

**Using wrong apikey**
→ Don't guess or decode from bundle. Extract from actual request headers (`requestHeaders: true`).

**Missing localStorage on first load**
→ State is populated after auth. Wait for network activity to settle before dumping localStorage.

**Querying with wrong column names**
→ The error message `column X does not exist` is useful — it tells you the column isn't there. Try `select=*` to get all available columns.

**Forgetting static assets**
→ SPAs often ship full config JSON files (templates, schemas, problem definitions) as public assets. Check `/assets/` and the asset manifest first.
