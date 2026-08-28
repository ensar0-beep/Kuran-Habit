# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Turkish-language PWA for tracking a daily Quran-reading habit ("Kuran Takip"). The entire app — markup, CSS, and JS — lives in one file: [kindex.html](kindex.html). There is no build step, no package manager, no bundler, and no test suite. There is also no `index.html` — the entry file is literally named `kindex.html`.

## Running / developing

There are no commands to build, lint, or test. To work on the app:
- Open [kindex.html](kindex.html) directly in a browser (or serve the directory with any static file server) and reload after edits.
- All persistence and reads happen over `fetch` to public internet APIs (see below), so the app needs real internet access even during local development — there is no mock/offline backend.
- Verify changes manually in the browser; there is no automated test to run.

## Architecture

### One file, three sections
[kindex.html](kindex.html) is organized as `<style>` (lines ~14-364) → static markup for the auth screen, three app tabs, and modals (lines ~366-628) → a single `<script>` (line 629 to end) containing all app logic. When making changes, find the relevant section with grep rather than assuming file layout.

### Data flow: localStorage-first, Firebase-synced
- `S` is the single global in-memory state object (see `freshState()`): `{name, levelIndex, streak, totalDays, history[], todayLogged, todayDate, currentPage, groupCode}`.
- `S` is mirrored to `localStorage` under key `kuran_v6` (`saveLocal`/`loadLocal`) and to a Firebase Realtime Database instance over plain REST — `fbGet`/`fbSet` just `fetch` `https://kuran-takip-default-rtdb.firebaseio.com/<path>.json`. There is no Firebase SDK.
- On every mutation the pattern is: mutate `S` → `saveLocal()` → `syncToFb()` (writes to `/members/{safeKey(name)}`) → re-render. Local storage is the source of truth for the current session; Firebase is for cross-device/group sync.
- A separate `localStorage` key, `kuran_session_v1`, stores just the logged-in username so `tryAutoLogin()` can restore a session on load.

### Auth is bespoke, not a real backend
There's no server. "Accounts" are records at `/users/{safeKey(username)}` in Firebase holding a SHA-256 password hash (`crypto.subtle.digest`, see `hashPassword`). `safeKey()` lowercases and strips non-alphanumerics to form Firebase-safe keys. This is app-level obfuscation only, not real auth security — don't treat it as a model for anything sensitive.

### History replay is the source of truth for progression
`S.history` (array of `{date, pages, success}`) is the only durable record of behavior. `S.levelIndex`, `S.streak`, and `S.totalDays` are **derived**, not stored authoritatively — `recalcAll()` replays the entire `history` day-by-day from the first entry to today every time it's called (after every log, edit, or data load) to recompute those three fields deterministically. Any code that edits history (`applyReading()`, used by both today's log and the calendar-editing modal) must call `recalcAll()` afterward rather than incrementing counters directly.

The progression rule set lives in the `LEVELS` array: escalating daily page targets (1 → 3 → 5 → 10 → 20 pages/day, then a 20×30 "Aylık Hatim" stage), each requiring a run of consecutive successful days (`days`) before advancing. Missing a day's target drops the level back by one stage and resets the streak (see the replay loop in `recalcAll`).

### Quran position/reference data
- `SURELER`: static table of `[sureName, startingPage]` across the 604-page standard mushaf layout; `getSure(page)` does a linear lookup.
- `JUZ_STARTS`: static `[sureNo, ayetNo]` start of each of the 30 juz; `getJuzFromVerse()` maps a verse to its juz.
- For precise sure/ayet-at-top-of-page (used by the "Kuran'da Yerim" card and the ayet modal), `fetchPageData()` fetches per-page layout JSON from `raw.githubusercontent.com/zonetecde/mushaf-layout` and caches results in `pageDataCache`. The static `SURELER`/`getSure()` tables are only an instant local estimate shown before that fetch resolves.
- `openAyetModal()` then fetches the Arabic text and Diyanet Turkish translation of that verse from `api.alquran.cloud`.
- Hijri date display (`hijriDate()`) is computed manually from a fixed anchor (1 Muharrem 1448 = 16 June 2026 UTC) plus a tabular intercalation rule, rather than relying on `Intl`'s Islamic calendar — this keeps it aligned with Diyanet's calendar regardless of device locale.

### Three tabs, one state object
`switchTab()` toggles the three `.pane` sections (`#tab-me`, `#tab-yol`, `#tab-grup`) and lazily renders each on activation. A touch-based swipe handler (`initSwipe`, using `TAB_ORDER`) also drives tab switching on mobile.
- **Bugün** ("Today"): log today's pages read via slider/quick buttons, shows current level target, and the "Kuran'da Yerim" position card/slider.
- **Yolculuk** ("Journey"): the level-progression chain (`renderChain`/`renderLevels`) and a monthly calendar (`renderCal`) for viewing/editing past days (`openEdit`/`saveEdit`, both funnel through `applyReading`).
- **Grup** ("Group"): create or join a group via a random 6-character code (`generateGroupCode`) stored at `/groups/{code}`; membership is a `{safeKey: displayName}` map on the group record, and the member list is rendered by fetching each member's `/members/{safeKey}` record and sorting by level/streak.

### UI conventions
All user-facing strings are Turkish. Styling is plain CSS custom properties (no framework) with a dark theme defined in `:root`. Per-stage accent colors (`STAGE_COLORS`, `CHAIN_COLORS`) drive slider thumb color, chain segment color, and button highlighting together — when adding a new level/stage, colors need to be added in both places to stay in sync.
