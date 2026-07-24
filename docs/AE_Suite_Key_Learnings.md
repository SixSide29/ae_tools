# AE Suite — Key Learnings & Direction File
*Distilled from project chat history (June–July 2026). Intended as reusable context for future script work.*
*Note: the most recent chat (2026-07-22) was unavailable for review; decisions made there are not reflected here.*

---

## 1. High-Level Goals

- **A unified intel platform, not a pile of scripts.** Every tool (Fleet Monitor, Tech Harvester, Base Intel, Pre-Battle Predictor, Bookmarks2, Intercept Solver, Scout Planner) feeds or reads a shared data layer. The trajectory is from siloed panels → shared stores → derived intel surfaces (hover intel, search palette, threat board) that are **read-only projections over data already collected — no new scraping surface per feature**.
- **Decision support over raw data.** The end state of each pipeline is an actionable answer: "who can hit me, with what, and when" (threat/ETA), "which target galaxy" (intercept solver), "what tech do they have" (harvester inversion), "send to BXX; scrap BYY" (scout planner).
- **Server-agnostic by construction.** No hardcoded server values; origin derived from `window.location.origin`; storage keyed per-server/per-origin.
- **Low detectability footprint.** Rate-limited fetches with jitter, hover surfaces that inject nothing at rest (delegated listener, tooltip built on hover, destroyed on leave).
- **FOREGROUND-ONLY FETCHING (locked 2026-07-23, supersedes older probe/auto-resume patterns anywhere in this file).** Every request to the game is user-initiated and visually represented — visible progress UI, pausable/abortable. No background processes: no idle/timer crawls, no auto-fetch on page load, no silent refresh, no hover-triggered fetches, no logout probe polling (logout re-detection is passive via DOM check on the user's next navigation). The universal action queue is the single enforcement point. Consequence of the same principle: scout-killer module and the pre-battle predictor's overlay popups were cut as too background-heavy.
- **Everything headlessly validated before delivery.** No script ships without `node --check` + jsdom assertion harnesses against real game-DOM samples.

## 2. UX Design Principles (recurring, confirmed choices)

**Visual identity**
- Deep-space sensor-console aesthetic: cyan primary `#3fb6c4`, amber secondary `#f0b541`, system font stacks, mono + tabular-nums for data. All colors via `var(--ae-*)` tokens, never hardcoded.
- Standalone HTML tools (Intercept Solver, Scout Planner) share the same dark "tactical console" look even though they're outside AEShared.

**Layout & chrome**
- Fixed-region corner tiling via AEShared + Overseer, not per-script drag chrome. Old drag handles, launcher pills, FABs, and saved-position logic get **removed** during retrofit; replaced by a position-neutral collapse-to-header toggle.
- Shadow DOM for all injected panels; script-specific structural styles as a second shadow-root `<style>` block so they survive live re-adoption.
- Exception pattern: tools meant to *look native* (Empire Menu Everywhere) stay light-DOM and inherit game CSS — retrofitting those to panels breaks their purpose.

**Interaction patterns Aran consistently prefers**
- **Rank-by-drag over numeric scores** (reliability list with movable FREE/CONTIG divider replaced a 0–10 score).
- **Paste-in parsing** as a first-class input method; parsers must be **delimiter-agnostic** (anchor on structural tokens like `[GUILD]` and trailing size, tokenize on any whitespace, tolerate emoji names / multi-word players / dash rows).
- **Settings as in-page shadow-DOM modal** sharing GM storage directly — not separate pages.
- **State visibility over silence**: hidden observations are dimmed + tagged with an active count; "all hidden" shows an explicit explanation rather than an empty-looking panel; localStorage failure degrades to "local save unavailable — use Export" instead of erroring.
- **Density control**: dense visualizations get a tiering control (timeline clock-marks: hours / halves / quarters / all / hide) rather than being fixed.
- **Color semantics for status grids**: green OK, orange new, blue changed, red missing — per-*item* (galaxy), not per-player.
- **Uncertainty must be shown, not hidden**: where a computed value depends on unknown inputs (e.g. ETA without confirmed drive/JG tech), the UI shows a range/badge, never a confident wrong number.
- **Amalgamated action output**: results end in imperative, grouped instructions ("Send to BXX; Scrap BYY"), plus BBCode export in AEU-legal markup (`[list]/[color]`; `[code]` + nbsp padding; **no `[table]`**).
- Wire tool-to-tool handoffs through the receiving tool's own API (departures panel calls the solver's `addObsFromLaunch()`), not clipboard/paste-box hacks.

## 3. Logic Pillars (recurring algorithmic patterns)

**Movement math (locked)**
- Galaxy distance: `c=floor(g/10)`, `p=g%10`; same-cluster `|p1−p2|×200`; cross-cluster `min((p1+p2)×200+1000, ((9−p1)+(9−p2))×200+1000)`; max 2800.
- Speed: `base × (1 + 0.05·drive) × (1 + 0.70·JG)`; fleet moves at slowest ship's base speed.
- Travel: `time = distance/speed`; arrival `= departure + travel×(1 − 0.01·logistics)`.
- Anchor on **timezone-free countdown fields**, never displayed clock times (which reflect the poster's AE profile timezone).

**Combat model (locked from calculator ports)**
- Shield-vs-attack breakpoints; ion bypass 50% (vs 1% standard passthrough); overkill water-fill targeting; damage factor 0.848; drop-then-finish wave logic.
- Tempo insight: splitting BO-heavy fleets raises hit volume; tempo inversion vs defender multiplier is the lever (×5 splits @1.5x, ×10 @2.0x). Support fleets (RC/SS/OS/CA/FC) excluded from crash modeling.

**Diff/state-tracking discipline**
- **Diff on conserved quantities per identity**, not greedy set-matching: the phantom departed/arrived bug came from size-bag matching across fleets; the fix tracked only present-mass delta per player with a `stationed` flag, ignoring inbound (timer) fleets.
- Snapshots must carry enough state flags to disambiguate repartitioning from movement.

**Allocation engines**
- Demand-first sort (remaining desc → score desc), **hard cap at demand** (never assign past remaining==0), window scoring by *sum of remaining demand* (never min-score tiebreaks that favor low-value regions), keep-cap on existing assets (retain runners only up to demand, most-reliable kept, extras become scraps freeing capacity).
- Gap-fill principle: running assets stay locked; churn only from demand edits or redundant over-coverage.

**Classification heuristics**
- Drop-wave detection: FT/BO ≥95% of value AND <30% of player's max observed force; defense-side never a drop; genuine swarms protected (largest attack == max value).
- Monotonic stores for inferred tech (levels only ratchet up as evidence accrues); confidence badges on inferred values.

**Command channels & sync**
- Commands in localStorage must be **consumed** (`removeItem`) after execution, with `lastCmdId` seeded from a persistent DONE key — otherwise re-execution on every navigation.
- GM storage is per-script isolated; **localStorage is the cross-script/cross-tab bridge** (per-origin = per-server). Shared logged-out flag lets any tab pause the whole suite; probe cadence ~60s while logged out, auto-resume on login.

## 4. Repeated Data Links (the shared data web)

- **Central layer convention**: localStorage mirrors keyed by `playerId` — shared by Tech Harvester, Base Intel, Pre-Battle Predictor.
- **Override channel**: `ae::overseer::v1` (per-origin) for theme/layout; live-update via storage event + 1.5s rev-counter poll. Overseer master in GM (`ae::overseer::master`) synced to per-server localStorage on load. Cross-server GM sync is **UNVERIFIED** (needs 2-server test).
- **Reports page as discovery index**: cheap Stage-1 index → rate-limited Stage-2 base-page fetches. Reports page is a guild scanner-network aggregator (positional + fleet size), not a report inbox.
- **Game UI as free oracle**: build previews / `base_ajax.aspx` expose cost/time deltas without executing; `#map_base` shows economy even when the base page is fogged.
- **Rankings merge**: four views (`ply_level|economy|fleet|technology`), 100/page, each carries only its own stat → must merge; reject the outer wrapper table (require ≥2 distinct header columns).
- **Merge rule for scraped records**: sweeps update only fetched sections with per-section `lastScraped` stamps; summary sweeps never clobber deep data.
- Solver ← Monitor (departures→observations); Predictor ← Harvester (tech levels); planned hover layer ← intel index derived purely from existing logs.

## 5. Engineering & Validation Discipline

- File assembly: **Python `open(...,'w')` only** — bash heredoc fails silently here.
- Staged migrations: **cut old code before inserting new** (reverse order deletes the new block).
- Template-literal regex trap: test with `eval` of the real literal, never faked escape processing.
- jsdom: no layout engine (pixel placement always flagged unverified); `runScripts:'dangerously'` + injected `<script>` (not `window.eval`); `await` a ~80ms tick after DOMContentLoaded registration; `global.navigator` read-only in Node 22.
- Trusted-Types CSP: never `innerHTML`; `adopt()` + `placeHost()` path, all chrome via DOM nodes; harnesses include an innerHTML-throws trap.
- module_build_test strips comments before scanning for undefined calls.
- Canonical-vs-embedded drift is a real failure mode: verify embedded `ae_shared.js` is byte-identical to canonical; update canonical + overseer copy + script copy together (3 places).

## 6. Direction Candidates (previously proposed, ordered by value/effort)

1. **Intel index** (pure derivation from existing logs — foundation, zero new footprint)
2. **Hover intel layer** (player/coord/fleet tooltips; inject-nothing-at-rest design)
3. **Threat board with ETA** (movement math + tech DB → time-to-reach my bases; show ranges where tech unknown)
4. **Search palette (Ctrl+K)** across all stores with actions
5. Smaller: tech sparklines (unused `history[]`), galaxy activity map, movement timeline

Open items carried forward: Base Intel drop-calc hover overlay (mid-implementation), Fleet Monitor→Scanner/Monitor split (2 decisions pending), pixel calibration of `getContentBox`/`bannerHeight`, 2-server GM sync test, Tech Harvester width 300→~340 check, browser-mode live theme for Harvester host.

---

## 7. Regression Checklist — Bug Classes That Have Actually Occurred

*Every rebuild/refactor must re-test these. Each entry: the bug, and the assertion that catches it.*

### Fetch / request-volume safety (highest severity)
- **Cascading / background explosion of page loads.** Risk vectors seen or designed against: (a) logged-out state — fetch returns the login page and a naive loop hammers it; MUST detect login-page response, set the shared logout flag, and stop fetching entirely (**no probe polling** — re-detection is passive via DOM check on the user's next navigation; supersedes the old ~60s probe cadence). Assert: zero fetches while paused/logged out; no snapshot stored while logged out; other tabs honor the shared flag. (b) Multiple tabs each running their own scan loop against the same locations — shared flags/coordination must dedupe, and scanner tabs on non-game pages must signal logout and store nothing. (c) Report-index → base-page pipelines: strictly sequential queue (concurrency default 1), delay + jitter (`interval + floor(random()*(jitter+1))`, defaults 2000/500ms), exponential backoff on failure, abort token, and a hard progress-bounded total — never unbounded follow-the-links. (d) Any fetch not tied to a visible, user-started run — forbidden outright under the foreground-only policy; assert zero fetches without an active run.
- **Command re-execution on every navigation** (delete-command bug). Commands must be consumed (`removeItem(CMD_KEY)`) after execution; `lastCmdId` seeded from persistent `CMD_DONE_KEY`. Assert: replaying a page load after a command executes it zero additional times — a repeating *fetch-bearing* command is a page-load explosion.
- **Scheduler math**: after resume, next-attempt time returns to normal cadence (test with a short interval so probe ≠ normal cadence is distinguishable — a prior test false-failed because default 120s cadence exceeded the 45s check).

### Diff / state-tracking
- **Phantom departed+arrived pairs** from greedy size-bag matching when fleets repartition (conserved-sum pattern). Fix: `stationed` flag in snapshots; diff only present-mass delta per player; ignore inbound (timer) fleets. Assert with a repartition fixture: no departure/arrival emitted when total present mass per player is unchanged.
- **Monotonic tech store**: inferred levels never regress on weaker evidence.

### Parsers (test against real-DOM fixtures, not synthetic)
- **Silent row drop on delimiter change**: paste arrived space-separated instead of tab and every row died at a null check. Parsers must be delimiter-agnostic (anchor on `[GUILD]` + trailing size, any-whitespace tokenize) and be tested with: spaces, tabs, emoji fleet names, multi-word players, dash rows, moving fleets with countdowns.
- **Outer wrapper table trap** (rankings): single header cell containing both "player" and the stat word — reject unless ≥2 distinct header columns.
- **Non-owner trade page** structural variant: plain tbody header row, no sorttable, plain-text distance.
- **Fogged base pages**: economy/capacity block absent — parser must not throw; `#map_base` fallback carries economy.
- **Template-literal regex**: `\\d` in source vs `\d` in browser. Test by `eval` of the real literal; never fake escape processing (gave false green before).

### Layout / overseer / UI
- **Stale px override pushing panels off-screen** (`ae::overseer::v1` pinning left=1632 on a 1365px viewport). Assert: overrides beyond viewport are clamped or clearable; provide a targeted-clear path.
- **Duplicate scriptId collision**: two scripts sharing `'bookmarks'` merged into one uncontrollable overseer row. Assert: detected host list has unique scriptIds; every host's id exists in REGIONS.
- **Canonical/embedded drift**: embedded `ae_shared.js` must be byte-identical to canonical (compare length + hash); local additions (e.g. `--ae-orange`) live in the script's own `:host` CSS.
- **Trusted-Types trap**: harness stubs `innerHTML` to throw; whole boot path must survive. Never `createPanel()` on CSP pages — `adopt()`+`placeHost()` with DOM nodes only.
- **Live override re-application**: theme/layout changes apply via StorageEvent AND the 1.5s rev poll without reload; host re-anchors on layout override.

### Allocation engine
- **Over-assignment to low-tier / starvation of high-override galaxies**: root causes were sort ignoring override demand, min-score CONTIG tiebreak, and uncapped runners. Regression fixture: real savefile must reach assigned == demand exactly (162=162), zero shortfall, zero over-assignment; runners kept only up to demand.

### Build / harness environment traps (false-positive & false-negative sources)
- Bash heredoc redirection fails **silently** → Python `open(...,'w')` only; verify output with `wc -l` / size checks.
- Staged migration ordering: cut old before inserting new, or later cuts match the new block's signatures and delete it.
- jsdom: `global.navigator` read-only (Node 22); `window.eval` doesn't resolve mocked bare globals — inject `<script>` with `runScripts:'dangerously'`; await ~80ms tick before asserting post-DOMContentLoaded state; `window.atob` stricter than `Buffer.from`.
- module_build_test must strip comments before scanning for undefined function calls (prose parentheses false-positive).
- localStorage may throw in sandboxed contexts: autosave must no-op gracefully with a visible "use Export" notice — assert no uncaught exception.
- **No pixel/layout assertions in jsdom** — anything depending on `getContentBox`/`bannerHeight` is permanently flagged unverified until measured on the live page; don't let a green harness imply visual correctness.
