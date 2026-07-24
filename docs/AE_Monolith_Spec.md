# AE Suite Monolith — Architecture Spec
*Companion to `AE_Suite_Key_Learnings.md`. §7 of that file is the test gate for every milestone here.*
*2026-07-23: AE Suite **v1.0.0 exists** (13k-line monolith supplied by Aran). User-set parameters/preferences below are extracted from it as ground truth; per Aran, the file was used ONLY for parameters/preferences, not logic.*

---

## 0. Locked Decisions

| Decision | Choice |
|---|---|
| Architecture | Single userscript (monolith), solo use |
| First ship | Parity with all current tools; new intel surfaces (index/hover/threat/search) come after |
| Fleet split | Fleet Scanner and Fleet Monitor as **two separate modules** inside |
| Activation | Single broad `@match` + **central page-router** |
| Store entities | players, bases, fleets, sightings, tech, bookmarks, observations |
| Old data | **One-time import** from existing localStorage keys |
| Error handling | Per-module error boundaries — non-negotiable |
| Standalone HTML tools | Intercept Solver + Scout Planner stay separate (out of scope) |

## 1. Architectural Consequences of Monolith (simplifications earned)

- **GM storage is now shared by all modules** (single script = single GM namespace). The per-script-isolation constraint that forced localStorage bridging disappears. GM becomes the primary persistent store; localStorage retained only for (a) legacy import, (b) any future handoff to the standalone HTML tools.
- **Overseer becomes an internal `layout` module.** No cross-script sync, no `ae::overseer::v1` command channel between scripts, no master/mirror architecture. Arrange mode manipulates the in-memory layout state directly; persisted to GM.
- **scriptId collisions are moot** — `bookmarks2` rename can revert to a clean `bookmarks` module id.
- **Drift is impossible** — one embedded theme/layout foundation (successor to `ae_shared.js` v0.3.1), one version.
- **One fetch engine** — the primary motivation. All request-volume safety lives in one place (§5).

## 2. Boot Sequence

1. `boot()` — origin detect (`window.location.origin` = server key), page classify (router, §6), GM store open + schema-version check/migrate.
2. Theme apply (tokens → shadow roots), layout state load.
3. For each module the router activates on this page: `try { module.init(ctx) } catch (e) { boundary(e, module.id) }`.
4. Kernel tick loop; each module tick also boundary-wrapped.

**Error boundary contract:** a throwing module is disabled for the session, logged to a visible (collapsed) diagnostics panel, and MUST NOT prevent other modules from initializing or ticking. Harness assertion: a deliberately-throwing module leaves all others running (see §7-Layout in Learnings).

## 3. Module Registry + Zone Grid (ground truth from v1.0.0 REGIONS)

**Zone-grid model (supersedes the earlier corner/“8 tiles” description):**
- Two gutters (left/right), **N slots each**, numbered top→bottom: `left-1..left-N`, `right-1..right-N`.
- Slot count is **user-configurable via Overseer → Settings**, range **2–8**, baked default **5**; stored in the shared store so every module resolves the same grid. **Aran's preference: 8 slots** — at 8, `left-5` ≈ mid-left, matching the stated fleet-scanner placement.
- **`left-1` is RESERVED for the Overseer** (must stay clickable during Arrange mode).
- Zones resolve to px at **render time** → viewport-responsive; saved zones exceeding a reduced slot count are clamped on-screen.
- Layout params: `bannerHeight 30` (⚠ still an estimate), `edgeGap 8`, `gutterGap 10`, `contentSelector null` (auto-detect — unverified seam), `contentFallbackWidth 1000`, `z 2147482000`.

**REGIONS defaults (zone / width):**

| Module id | Zone | Width | Notes |
|---|---|---|---|
| `overseer` | left-1 | 460 | reserved slot |
| `report-scanner` | left-2 | 400 | |
| `bookmarks2` | left-3 | 280 | Auto-Launch + Destinations |
| `intercept-solver` | left-4 | 400 | **includes fleet monitor** |
| `fleet-scanner` | left-5 | 300 | exact compositions; mid-left at 8 slots |
| `messages-parser` | left-5 | 320 | ⚠ default-zone collision with fleet-scanner |
| `tech-harvester` | right-1 | 300 | |
| `bookmarks` | right-2 | 280 | Quick Bookmarks |
| `prebattle-predictor` | right-3 | 260 | |
| `map-intel` | right-4 | 300 | Base Intel (rival base DB) |
| `scout-killer` | right-5 | 320 | **REMOVED from scope (2026-07-23, too background-heavy)** — right-5 now free |
| `credit-optimizer` | right-5 | 340 | registered placeholder, **NOT in suite build** (matches exclusion); reassign a zone if it ever ships |
| `reports-monitor` | left-2 | 400 | legacy id kept so pre-split overrides resolve |
| `__fallback__` | right-4 | 300 | unregistered scripts |

Default-zone collisions (left-5 ×2, right-5 ×2) are baked defaults only — Overseer overrides spread them; 8 slots gives headroom. `empire-menu` remains light-DOM exempt (no panel, no zone).

**Note superseding §0:** the fleet split in v1.0.0 landed as **three** surfaces — `report-scanner` + `intercept-solver` (incl. monitor) + `fleet-scanner` — **adopted** as the target structure (Aran, 2026-07-23).

**Intentional differences vs. v1.0.0 as-built (Aran, 2026-07-23 — these SUPERSEDE v1.0.0's structure):** universal action queue (single kernel-owned queue for all fetches/side-effects), per-module boundary framework, single unified store schema (replacing federated `ae::bases/rankings/techmirror/...` domain stores), central page-router.

Modules communicate **only via the store** (write/subscribe). No direct cross-module calls except through a small event bus owned by the kernel.

## 3b. Kernel Base-Function API (what every module builds on)

| Service | API surface (sketch) | Notes |
|---|---|---|
| **Store** | `store.get/query(entity,…)` · `store.upsert(entity,id,patch,{section})` · `store.subscribe(selector,cb)` · `store.tx(fn)` | Unified schema (§4). Upsert enforces merge rules: per-section `lastScraped`, monotonic tech ratchet, no summary-clobbers-deep. Persistence behind the API: GM primary (debounced flush) + localStorage mirror w/ rev counter (HTML-tool export, cross-tab signal). |
| **Action queue** | `queue.batch(jobs,{bound,abortToken})` · `queue.resume(cursorKey)` | ALL fetches/side-effects go through it — **foreground-only: user-initiated, visible progress UI, no background traffic (§5)**. Pacing = user config (3 s mean, ±30 %+smear, 0.6 s hard floor, 250/session, stopOnErrors 3); logout → pause, passive zero-fetch re-detect; multi-tab leader election; jobs must declare a bound. |
| **Router** | `PageCtx {type, params, flags}` handed to `module.init(ctx)` | Central URL→module activation map; classifies structural variants (fogged base, non-owner trade) once. Modules never re-parse the URL. |
| **Boundary** | `boundary.wrap(moduleId, fn)` + status registry | Wraps init/tick/handlers; throw → module disabled for session, logged to diagnostics panel, others unaffected. |
| **UI / layout** | `ui.panel(moduleId)` → zone-grid host · `ui.status/toast` | Shadow-DOM chrome, theme tokens, zone grid (2–8 slots, left-1 reserved), live override re-application. |
| **Parsers** | `parsers.<pageType>(dom)` · `parsers.fleetList(text)` | One shared parser per page type + delimiter-agnostic text parsers. Modules receive parsed records, not raw DOM. |
| **Domain lib** | `domain.distance/speed/eta` · `domain.combat` · `domain.diffPresence` · `domain.dropTag` | Locked math (Learnings §3): movement, combat model, conserved-mass diffing, drop-wave tagging. |
| **Event bus** | `bus.emit/on(topic)` | Only for signals that aren't naturally store-shaped (e.g. launch-clicked → solver). Store-subscribe is the default channel. |

**Module → service/entity matrix** (⚠ = behavior assumed, not read from v1.0.0 logic):

| Module | Kernel services | Reads (entities) | Writes |
|---|---|---|---|
| report-scanner | router, parsers, store, ui, boundary | players, fleets | sightings, fleets, players |
| fleet-scanner | + queue (paced composition fetches) | fleets | fleets (exact comps), sightings |
| tech-harvester | router, parsers, domain (inversion), store, ui | fleets, players | tech, players |
| map-intel | router, **queue (heavy)**, parsers, store, ui | all | bases, players, fleets, sightings |
| messages-parser ⚠ | router, parsers, store | — | sightings, players |
| intercept-solver (+monitor) | store.subscribe, domain (movement), bus-in, ui | sightings, fleets, tech, observations | observations |
| prebattle-predictor | store, domain (combat), ui | tech, fleets, players | — (ephemeral output; **panel only — overlay popups REMOVED 2026-07-23**) |
| bookmarks | store, ui | bookmarks | bookmarks |
| bookmarks2 | store, ui, **bus-out (launch events)** | bookmarks | bookmarks |
| overseer/layout | ui, store (config) | config | config |
| empire-menu | router only (light-DOM exempt) | — | — |

## 4. Canonical Store

Persisted in GM, keyed per server origin: `ae::store::<origin-host>::v1`. In-memory object + debounced flush. Per-section `lastScraped` stamps; summary sweeps never clobber deep data (merge rule from Learnings §4).

| Entity | Key | Core fields (non-exhaustive) |
|---|---|---|
| players | playerId | name, guild, rankings {level, economy, fleet, technology, page-seen}, profile, lastSeen |
| bases | coords string | owner playerId, name, economy, structures[], commander {id, name, type, level}, fog flag, per-section stamps |
| fleets | synthetic id (owner+location+spec hash) | owner, location, composition{unit:count}, size, stationed flag, source (report/base/scan), timestamp |
| sightings | append-only log id | fleet ref or raw spec, location, time, source page — feeds diffs + future intel index |
| tech | playerId | per-tech level (monotonic ratchet), confidence, history[] |
| bookmarks | bookmark id | target, auto-launch config, destinations |
| observations | obs id | intercept-solver observations (location, time, vector) — for future in-suite solver panel + HTML-tool export |

Rules carried in: diff on conserved present-mass per player with `stationed` flag (never greedy size-bag matching); inbound/timer fleets excluded from diffs; tech monotonic; drop-wave heuristic (FT/BO ≥95% value AND <30% max observed; defense never a drop; swarm protection) tagged on sightings, not destructively applied.

## 5. Action Queue (single, kernel-owned — the ONLY fetch surface)

**FOREGROUND-ONLY POLICY (Aran, 2026-07-23 — supersedes anything below or in v1.0.0 that conflicts):** every request to the game must be **user-initiated and visually represented**. No background processes: no idle/timer crawls, no auto-fetch on page load, no silent refresh, no hover-triggered fetches. Every queue batch (a) starts from an explicit user action, (b) shows visible progress UI for its whole run (banner/progress à la map-intel scan), (c) is pausable/abortable from that UI. The universal queue is the enforcement point: a fetch outside a visible, user-started run is a spec violation.

- **User-set pacing (ground truth, v1.0.0 CRAWL_DEFAULTS):** mean delay **3 s**, jitter **±30% + sub-second smear** (never machine-regular), **hard floor 0.6 s** regardless of settings (server courtesy), **maxPerSession 250** (hard cap per run; resumable, so not a coverage limit), **stopOnErrors 3** consecutive failures aborts, tradePlunder **opt-in, default off**. Pacing is user-adjustable (Aran's call); defaults are a conservative starting point, not a ceiling. Rankings default: 4 views × pages 1–3 = 12 paced fetches (top 300).
- Sequential queue, concurrency 1; abort token per batch; hard bounded totals — no unbounded link-following.
- **Logout handling — passive, zero-fetch:** login-page detected in a response → pause the run, set shared flag, **no probe requests** (replaces v1.0.0's ~60 s probe cadence — probes are background traffic). Re-detection is passive: the next game page the user navigates to proves login via DOM check and clears the flag; the paused run offers a visible Resume. Nothing stored while logged out.
- Multi-tab: single-writer election (heartbeat key); non-leader tabs enqueue nothing.
- Any module command that triggers fetches must be consumed after execution (`removeItem` pattern) with done-id persistence — §7 delete-command class.
- Harness assertions: **zero fetches without an active user-started run**; zero fetches while paused/logged out (no probe traffic); zero stores while logged out; replayed page load re-executes a consumed command zero times; queue length never exceeds declared batch bound.

## 6. Page Router

Central URL→modules map (single source of truth), e.g. `reports.aspx → [fleet-scanner]`, `board.aspx / base.aspx → [fleet-monitor, base-intel]`, `battle report → [tech-harvester]`, `ranks.aspx → [base-intel.rankings]`, `* → [bookmarks, layout]`. Modules receive a page descriptor; they do not re-parse the URL. Known structural variants (non-owner trade page, fogged base) are classified here and passed as flags.

## 7. One-Time Import (M3)

Importer runs once per server (guarded by a done-flag), reads legacy keys, maps into store, leaves originals untouched (no deletes). Known keys to enumerate from current scripts at build time — minimum set: bookmarks (`ae_bm::…`), tech harvester mirror, base-intel mirrors, predictor mirror, `ae::overseer::v1` layout/theme overrides (import positions only if within viewport — stale-px clamp rule). Import is idempotent and boundary-wrapped; a failed import must not block boot.

## 8. Build Milestones (Claude Code)

- **M0** — Repo scaffold in `ae-tools`, port jsdom harness suites, embed theme foundation. Gate: existing 32+ assertions green against scaffold.
- **M1** — Kernel: boot, router, registry, error boundaries, store, fetch engine. Gate: new kernel test files covering §5 assertions + throwing-module isolation.
- **M2** — Module ports, one at a time, ascending risk: bookmarks → tech-harvester → pre-battle-predictor → fleet-scanner → fleet-monitor → base-intel. Each port ships only when its existing suite passes rewired against the kernel. Staged-migration rule applies (cut before insert).
- **M3** — Importer + idempotency tests.
- **M4** — Live-browser checklist (jsdom-unverifiable): pixel calibration `getContentBox`/`bannerHeight`, panel placement per region, Tech Harvester width, Trusted-Types on real CSP pages, real logged-out behavior, multi-tab leader election.

## 9. Open Items (updated 2026-07-23 against v1.0.0)

- ~~fleet-scanner placement~~ — resolved: `left-5` at Aran's 8-slot grid ≈ mid-left.
- ~~Ambiguous module inclusions~~ — resolved: credit-optimizer OUT (placeholder REGIONS entry only). v1.0.0 also carries `messages-parser` and `scout-killer`, which the earlier parity list missed.
- ~~localStorage cross-script channel~~ — **probe-verified** per v1.0.0 header (shared across userscripts on same origin; GM confirmed per-script isolated). Caveat noted in-file: game page can read the key — layout/theme prefs only, hard-namespaced.
- `bannerHeight` still an in-file estimate (30); `contentSelector` still null/auto-detect — the pixel-calibration seam remains open (live-page measurement).
- GM cross-subdomain span — still UNVERIFIED; low stakes (store keyed per origin) but worth one live test.
- 2026-07-22 chat remains unreadable; v1.0.0 itself is the best record of what was decided there.
