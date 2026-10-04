# Mission Headquarters
### The Control Plane for GitHub Sub-domain Mission-Based Games

> **One domain. Dozens of repos. One shared brain.**

Mission Headquarters is an opinionated framework for building rich, cross-repo games and interactive experiences on `*.github.io`. It treats your GitHub Pages as a galaxy of levels, and IndexedDB as the shared physics that binds them.

```
https://yourname.github.io/mission-hq/      <- HQ (this repo)
https://yourname.github.io/repo-a/          <- Level / Mechanic A
https://yourname.github.io/repo-b/          <- Level / Mechanic B
https://yourname.github.io/repo-c/          <- Level / Mechanic C
```

![Architecture Overview](https://raw.githubusercontent.com/EricEisaman/mission-headquarters/main/assets/images/01-architecture-overview.webp)

---

## Why This Architecture?

Most portfolio / open-source projects die as islands. This pattern makes them **symbiotic**:

1.  **Preserve Purity:** Each repo keeps its original purpose, tech stack, and clean architecture. We don't rewrite it — we *hijack* it minimally.
2.  **Shared Persistence Without a Server:** IndexedDB on `yourname.github.io` origin is shared across all sub-paths. It can store 100s of MBs of structured data, blobs, HTML/CSS, and application state.
3.  **Declarative Game Design:** A single `mission.json` drives the entire player experience. No imperative spaghetti across repos.
4.  **Composability:** One repo can be part of many missions. One mission can include any open-source Pages site you control.

## Core Opinionated Principles

**1. HQ Owns the Truth, Repos Own the Fun**
Mission Headquarters is the single source of truth for mission state, progression, and assets. Repos are dumb mechanics that emit events.

**2. Two-Line Hijack Rule**
If integrating a repo requires more than ~20 lines or pollutes its core, you're doing it wrong. The bridge must be an isolated module.

**3. IndexedDB is the OS**
We standardize the DB so every repo speaks the same protocol. No ad-hoc localStorage keys.

**4. Everything is a Blob**
HTML, CSS, SVG, JSON, PNG, WASM — if it can be a Blob, it can be a shared mission asset cached in IndexedDB and hot-swapped across repos.

## The Declarative Mission Data Structure: `mission.json`

This is the heart of the design. A mission is pure data. The engine is generic.

![Mission JSON Blueprint](https://raw.githubusercontent.com/EricEisaman/mission-headquarters/main/assets/images/02-mission-json-blueprint.webp)

```jsonc
{
  "mhq": "1.0.0",
  "mission": {
    "id": "symbiotic-archipelago-v1",
    "version": "1.0.0",
    "title": "The Symbiotic Archipelago",
    "description": "Recover fragments of your past projects scattered across the Pages galaxy.",
    "author": "yourname",
    "entryUrl": "https://yourname.github.io/mission-hq/",
    "mode": "sequential", // sequential | open-world | branching
    "origin": "https://yourname.github.io"
  },

  "indexedDB": {
    "dbName": "mhq_symbiotic-archipelago-v1",
    "version": 1,
    "stores": ["meta", "state", "events", "assets", "repo_registry", "checkpoints"]
  },

  "repos": [
    {
      "id": "portfolio-gallery",
      "url": "https://yourname.github.io/repo-a/",
      "role": "collector",
      "primaryPurpose": "My photo portfolio (React + Vite)",
      "hijack": {
        "moduleUrl": "./bridges/repo-a-bridge.js",
        "mountSelector": "#mhq-overlay",
        "permissions": ["state:read", "state:write", "events:emit", "assets:read"]
      },
      "provides": ["pointer-input", "gallery-items"],
      "requires": []
    },
    {
      "id": "shader-lab",
      "url": "https://yourname.github.io/repo-b/",
      "role": "puzzle",
      "primaryPurpose": "WebGL shader experiments",
      "hijack": {
        "moduleUrl": "./bridges/repo-b-bridge.js",
        "mountSelector": "#mhq-overlay",
        "permissions": ["state:read", "events:emit"]
      }
    }
  ],

  "state": {
    "schema": {
      "type": "object",
      "properties": {
        "fragments": { "type": "array", "maxItems": 12 },
        "inventory": { "type": "array" },
        "player": { "type": "object" }
      }
    },
    "initial": {
      "fragments": [],
      "inventory": [],
      "player": { "level": 1, "visited": [] },
      "progress": 0
    }
  },

  "objectives": [
    {
      "id": "obj-find-prism",
      "title": "Find the Prism",
      "type": "interact", // collect | interact | create | solve | explore | craft
      "repo": "portfolio-gallery",
      "description": "Click the hidden prism in your 2019 gallery",
      "trigger": { "on": "repo:mounted", "in": "portfolio-gallery" },
      "validation": {
        "type": "statePredicate",
        "predicate": "state.fragments.includes('prism')"
      },
      "rewards": [
        { "type": "statePatch", "patch": { "fragments": ["$append:prism"] } },
        { "type": "assetUnlock", "assetId": "theme-nebula-css" },
        { "type": "eventEmit", "event": "fragment:found", "payload": { "id": "prism" } }
      ]
    }
  ],

  "stages": [
    {
      "id": "awakening",
      "title": "Awakening",
      "requires": [],
      "unlocks": ["portfolio-gallery"],
      "completion": { "allOf": ["obj-find-prism"] }
    },
    {
      "id": "refraction",
      "title": "Refraction",
      "requires": ["awakening"],
      "unlocks": ["shader-lab"],
      "completion": { "anyOf": ["obj-solve-shader"] }
    }
  ],

  "ui": {
    "hud": {
      "componentUrl": "./ui/hud.js",
      "position": "top-right",
      "themeAssetId": "theme-nebula-css"
    },
    "map": {
      "enabled": true,
      "type": "archipelago",
      "assetId": "map-svg"
    },
    "transitions": "portal" // portal | fade | slide
  },

  "assets": [
    { "id": "theme-nebula-css", "type": "text/css", "src": "./assets/nebula.css", "cache": "eager" },
    { "id": "success-sfx", "type": "audio/wav", "src": "./assets/success.wav", "cache": "lazy" },
    { "id": "map-svg", "type": "image/svg+xml", "src": "./assets/map.svg", "cache": "eager" }
  ]
}
```

### Design Notes on the Structure

- `mode` enforces player freedom. `sequential` is curated, `open-world` lets HQ map show all repos.
- `validation` is never in repo code. Repos just emit `mhq:objective:attempt`. HQ validates via predicate.
- `rewards` are declarative side-effects. No repo writes state directly except via allowed patch.
- `$append:` is a tiny DSL for immutable patches to keep state logic declarative.

## The IndexedDB Standard: The Shared Brain

All Pages under `https://yourname.github.io/*` share the **same origin**, so they share the same IndexedDB. We formalize it.

![IndexedDB Core Internals](https://raw.githubusercontent.com/EricEisaman/mission-headquarters/main/assets/images/03-indexeddb-core-internals.webp)

### DB Naming Convention

```
mhq_<github-username>_<mission-id>_<major-version>
Example: mhq_alice_symbiotic-archipelago-v1_1
```

This prevents collision between your missions and lets you run multiple missions in parallel.

### Standard Stores (v1)

| Store | Key | Value | Purpose |
| :--- | :--- | :--- | :--- |
| `meta` | `missionId` | `mission.json` + hash + installedAt | Cache manifest, version check |
| `state` | `"current"` | Validated player state (JSON) | Single source of truth. Patched only via `MissionBridge.patch()` |
| `events` | autoIncrement | `{ ts, type, repo, payload }` | Append-only log for analytics, replay, and objective validation |
| `assets` | `assetId` | `{ blob, mime, src, cachedAt, version }` | HTML/CSS/JS/WASM/Images stored as Blobs. The magic sauce. |
| `repo_registry` | `repoId` | `{ url, lastSeen, status, provides }` | Which repos are part of this mission and their health |
| `checkpoints` | `stageId` | `{ stateSnapshot, ts }` | Save points for branching missions |

### The MissionBridge SDK (What repos actually import)

```js
// In any repo: 2 lines, no pollution
import { MissionBridge } from 'https://yourname.github.io/mission-hq/sdk/bridge.js';
const mhq = await MissionBridge.connect('symbiotic-archipelago-v1');

// Repo never touches IndexedDB directly
mhq.onMount(async ({ state, assets }) => {
  // Inject shared theme if unlocked
  if (assets.has('theme-nebula-css')) {
    const css = await assets.getText('theme-nebula-css');
    document.adoptedStyleSheets = [new CSSStyleSheet()];
    document.adoptedStyleSheets[0].replaceSync(css);
  }
  showHiddenPrismIfNeeded(state);
});

canvas.addEventListener('prism-found', () => {
  // Don't mutate state. Emit intent. HQ validates.
  mhq.emit('objective:attempt', { objectiveId: 'obj-find-prism', payload: { fragments: ['prism'] } });
  mhq.emit('fragment:found', { id: 'prism' });
});

// Listen for progression
mhq.on('state:changed', (newState) => updateProgressBar(newState.progress));
mhq.on('stage:unlocked', ({ stage }) => showPortalTo(stage.unlocks[0]));
```

**Rules Enforced by Bridge:**
- `permissions` from `mission.json` are enforced. A puzzle repo can't write inventory.
- All writes go through JSON Schema validation against `state.schema`.
- Blobs >5MB are chunked automatically.
- Events are rate-limited and sanitized.

### Asset Flow: How HTML/CSS Moves Between Repos

1.  HQ pre-caches `assets` marked `eager` into `assets` store on mission start.
2.  Any repo can `mhq.assets.get('theme-nebula-css')` -> returns Blob -> convert to text/objectURL.
3.  A repo can contribute assets: `mhq.assets.put('player-screenshot', canvasBlob)` — now available to all other repos.
4.  This lets repo-b's generative art become repo-a's gallery texture without a server.

## Player Lifecycle

![Player Journey](https://raw.githubusercontent.com/EricEisaman/mission-headquarters/main/assets/images/04-player-journey.webp)

```
1. Land on HQ -> MissionBridge creates/opens DB, caches mission.json + eager assets
2. HUD mounts -> shows map of all registered repos, locked/unlocked states
3. Player clicks portal to repo-a -> navigates to https://yourname.github.io/repo-a/
4. repo-a's bridge.js mounts -> connects to same DB, reads state, emits events
5. Objective validated in HQ logic (running in shared worker) -> state patched -> checkpoint
6. Player returns to HQ or jumps directly to next repo -> progress persists
7. Mission complete -> final state + event log exported as JSON blob for sharing
```

## Minimal Repo Integration Guide

For each repo you want to hijack:

**1. Add `/public/mhq-bridge.json`**
```json
{
  "missionEnabled": true,
  "allowedMissions": ["symbiotic-archipelago-v1", "*"],
  "bridge": "./mhq/bridge.js"
}
```

**2. Add `/mhq/bridge.js` (20 lines template)**

We provide a CLI to scaffold it:

```bash
npx mhq add repo-a --template react
# or vanilla, svelte, vue, etc.
```

This file is the ONLY change to your repo. It doesn't import your app code — your app code optionally imports it.

**3. Deploy as usual to GitHub Pages.** No build step change.

The separation is absolute: your original repo still works 100% without HQ. If IndexedDB is empty, bridge no-ops.

## Project Structure of This Repo (Mission Headquarters)

```
/
├── mission.json              # Your declarative mission (the game)
├── sdk/
│   ├── bridge.js             # The 2-line import for child repos
│   ├── hq-engine.js          # Validation, state machine, asset manager
│   └── worker.js             # SharedWorker for cross-tab state sync
├── bridges/                  # Reference implementations of repo bridges
│   ├── repo-a-bridge.js
│   └── repo-b-bridge.js
├── ui/
│   ├── hud.js                # Web Component <mhq-hud>
│   └── map.js                # <mhq-map type="archipelago">
├── assets/                   # Shared CSS, SFX, SVGs cached as blobs
└── images/                   # Architecture diagrams for this README
```

## Creating Your First Mission

```bash
git clone https://github.com/yourname/mission-hq
cd mission-hq
npm create mhq@latest my-first-mission
# Answer: title, repos to include, mode
npm run dev # local HQ with mocked IndexedDB
```

Edit `mission.json`, add objectives. Test by opening `http://localhost:5173` and your local repo-a.

## Philosophy

This architecture bolsters the creative developer mindset: constraints breed creativity. By limiting cross-repo communication to a single, well-typed, offline-first database and a declarative manifest, you are forced to think in terms of **mechanics as verbs** (collect, solve, create) rather than tightly coupled features.

It's not a framework for building a game *in* a repo. It's a framework for building a game *out of* repos.

You already built the levels. Now build the mission that connects them.

---

## License & Attribution

MIT. Build missions, share them, include open-source Pages as levels (respect their licenses).

If you ship a mission, add it to the `MISSIONS.md` gallery.

**Reference:** [IndexedDB API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)

> Designed for `yourname.github.io/*` — where every repo is a portal.
