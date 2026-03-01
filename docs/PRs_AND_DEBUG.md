# Cosmic Law — PRs and in-game debugging

## Branch order (merge in this order)

1. **mod-foundation** — Mod skeleton, game rules (scope + Kundali + Debug), scripted triggers.
2. **traits-rashi** — 12 Rashi traits, birth/game-start assignment, debug Rashi decision.
3. **planets-navagraha** — Planet variables, game-start init, day-based transit, debug planets decision.
4. **planet-rashi-interactions** — Modifiers, yearly refresh, debug modifier decision.
5. **rashi-relationships** — Mitra/Shatru triggers, Rashi lord (planet in own sign), Rashi compatibility opinion (spouse yearly), marriage compatibility (spouse opinion; acceptance requires override — see IMPLEMENTATION_PLAN).
6. **sade-sati** (separate) — Sade Sati phase detection and events.

Create PRs from each branch into `main` and merge in order 1 → 2 → 3 → 4 → 5 (then 6).

## In-game debugging

1. **Game rules (at new game):**
   - **Vedic Astrology Scope:** set to **Cosmic Law (Global)** or **Religious Focus (Hindu Only)**.
   - **Cosmic Law Debug:** set to **On**.

2. **Decisions (when Debug is On):**
   - **Debug: Show my Rashi (Moon sign)** — Opens an event showing your character’s Rashi trait (or “No Rashi” if out of scope).
   - **Debug: Show planet positions** — Confirms transits are running; describes day-based positions.
   - **Debug: Show my cosmic modifier** — Shows whether you have the “Cosmic Planet Awareness” modifier (refreshed yearly for characters with a Rashi).

3. **What to check:**
   - New game: all living characters get a random Rashi (if scope is Global or you are Hindu).
   - New births: child gets a random Rashi (if in scope).
   - After a few in-game years: take “Debug: Show my cosmic modifier” — you should have the modifier if you have a Rashi.
   - Planet positions are fixed at game start and do not change on reload; they advance every quarter (day-based).

## Running the mod

Copy the repo folder (e.g. `CK3_Cosmic_Law`) to your mods directory and ensure the `.mod` file points to it. Example:

- Folder: `Documents\Paradox Interactive\Crusader Kings III\Mod\CK3_Cosmic_Law`
- File: `Documents\Paradox Interactive\Crusader Kings III\Mod\Cosmic_Law.mod` with `path="mod/CK3_Cosmic_Law"` (or match your folder name).

Trait icons (e.g. `rashi_mesha.dds`) go in `gfx/interface/icons/traits/`; see that folder’s README.
