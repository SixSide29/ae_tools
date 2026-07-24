# CLAUDE.md — AE Suite monolith

## Project
Tampermonkey monolith userscript for AstroEmpires intel/decision-support. Solo-use tool.
Authoritative docs (read before any structural work):
- `docs/AE_Monolith_Spec.md` — architecture, kernel API, module matrix, milestones M0–M4
- `docs/AE_Suite_Key_Learnings.md` — design principles, locked math, §7 regression checklist (the test gate)
Reference sources for ports: `reference/AE_Suite-1_0_0.user.js` (v1.0.0 as-built), `reference/legacy/*.user.js` (per-tool validated scripts), `reference/context/*` (game-data docs).
Conflict rule: chat-locked structural choices in the spec SUPERSEDE v1.0.0 as-built (unified store, universal action queue, boundary framework, central router, foreground-only fetching).

## Commands
- Syntax gate: `node --check dist/ae_suite.user.js`
- Test suite: `node tests/run_all.mjs` (jsdom harnesses; must pass before any commit)
- Build: `node build.mjs` (assembles dist from src modules; no external build deps)

## Hard rules (violations = rejected work)
1. FOREGROUND-ONLY FETCHING. No fetch outside a visible, user-started queue run. No idle/timer crawls, no auto-fetch on page load, no silent refresh, no hover-triggered fetches, no logout probe polling (logout re-detect is passive DOM check on next navigation). Every batch: user-initiated, visible progress UI, pausable/abortable, declared bound.
2. All fetches/side-effects go through the kernel action queue. Pacing: 3 s mean, ±30% jitter + sub-second smear, 0.6 s hard floor, 250/session cap, abort after 3 consecutive errors.
3. Every module entry point wrapped in the boundary framework. A throwing module is disabled for the session; others keep running. There is a test for this.
4. One unified store behind `store.*` API (entities: players, bases, fleets, sightings, tech, bookmarks, observations). GM primary per server, localStorage mirror + rev. Modules never touch GM/localStorage directly.
5. Central page-router only. Modules receive PageCtx; they never parse the URL or re-detect page variants.
6. Never `innerHTML` anywhere in panel chrome (Trusted-Types CSP on game pages). DOM nodes only; `adopt()`/`placeHost()` path.
7. Staged migrations: cut old code BEFORE inserting replacements (reverse order deletes the new block when signatures match).
8. Every §7 regression class in the Learnings file has a corresponding test. New bug fixed = new test added in the same commit.
9. Locked math (movement, combat, diff-by-conserved-mass, drop-wave heuristic, allocation) is ported verbatim from validated sources, never re-derived.

## Environment notes
- These Learnings items are claude.ai-sandbox-specific and DO NOT apply here: "bash heredoc fails silently / use Python open()" (heredocs work locally), `global.navigator` read-only workaround only matters on Node 22+.
- These jsdom constraints DO still apply: no layout engine (never assert pixel positions — anything touching getContentBox/bannerHeight stays flagged UNVERIFIED for manual M4), `runScripts:'dangerously'` + injected `<script>` (not `window.eval`), ~80 ms tick before asserting post-DOMContentLoaded state, `window.atob` stricter than `Buffer.from`.
- Fixtures in `tests/fixtures/` are the only game HTML available. Never attempt to fetch astroempires.com. If a needed fixture is missing (page type or variant), STOP and ask Aran to capture it — do not synthesize game HTML for parser tests.
- module_build check: strip comments before scanning for undefined function calls (prose parentheses false-positive).

## Working style (Aran)
- Terse-directive. Propose options with tradeoffs; wait for the decision; execute cleanly.
- Flag every unverified assumption explicitly. State impossibilities plainly. No overconfident claims.
- On correction: minimal redo, no lengthy error post-mortems.
- Visual standard: deep-space sensor console — cyan `#3fb6c4` primary, amber `#f0b541` secondary, system font stacks, mono + tabular-nums for data, `.ae-*` classes, all colors via theme tokens.
- Zone grid: left/right gutters, slots 2–8 (Aran runs 8), left-1 reserved for Overseer. In-game nomenclature in comments/UI (FT = fighter, BO = bomber, etc.).

## Milestones (from spec §8 — one commit series each, tests green at every gate)
M0 scaffold + harness port → M1 kernel (router, queue, store, boundary, ui, parsers, domain, bus) → M2 module ports ascending risk (bookmarks → tech-harvester → predictor → report-scanner → fleet-scanner → intercept-solver → map-intel → messages-parser) → M3 one-time importer (idempotent, legacy keys left intact, exporter-shim for GM-isolated legacy data) → M4 manual live-browser checklist (Aran executes; produce the checklist file).
Removed from scope: scout-killer, credit-optimizer, predictor overlay popups.
