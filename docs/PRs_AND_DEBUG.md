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
   - **Debug: Show my Rashi (Moon sign)** — Opens an event showing your character's Rashi trait (or "No Rashi" if out of scope).
   - **Debug: Show planet positions** — Confirms transits are running; describes day-based positions.
   - **Debug: Show my cosmic modifier** — Shows whether you have the "Cosmic Planet Awareness" modifier (refreshed yearly for characters with a Rashi).

3. **What to check:**
   - New game: all living characters get a random Rashi (if scope is Global or you are Hindu).
   - New births: child gets a random Rashi (if in scope).
   - After a few in-game years: take "Debug: Show my cosmic modifier" — you should have the modifier if you have a Rashi.
   - Planet positions are fixed at game start and do not change on reload; they advance every quarter (day-based).

## Running the mod

Copy the repo folder (e.g. `CK3_Cosmic_Law`) to your mods directory and ensure the `.mod` file points to it. Example:

- Folder: `Documents\Paradox Interactive\Crusader Kings III\Mod\CK3_Cosmic_Law`
- File: `Documents\Paradox Interactive\Crusader Kings III\Mod\Cosmic_Law.mod` with `path="mod/CK3_Cosmic_Law"` (or match your folder name).

Trait icons (e.g. `rashi_mesha.dds`) go in `gfx/interface/icons/traits/`; see that folder's README.

## Localization encoding

**CK3 expects localization `.yml` files to be saved with UTF-8 BOM encoding.** If files are plain UTF-8 (no BOM), the game may show raw keys instead of the translated text (e.g. `trait_rashi_simha` instead of "Simha Rashi (Leo)").

When editing files in `localization/english/`, save as **UTF-8 with BOM** (e.g. in VS Code: save, then use "Save with Encoding" and choose "UTF-8 with BOM").

## Debug output (console vs messages)

**Script cannot log to the game console.** CK3 has no script effect that writes to the debug console or to `game.log`; the engine only logs its own errors (e.g. missing localization, parse errors). So debug output has to use in-game UI.

**Why toasts often don't show:** `send_interface_toast` from a **quarterly pulse** frequently does not appear. Likely causes: the pulse runs in a context where toasts are not delivered to the UI, or the toast type/scope is ignored when triggered from this on_action. Vanilla uses toasts mainly from events, decisions, or character-scoped effects, not from global/time-based pulses.

**What the mod does instead:** When Cosmic Law Debug is **On**, the mod uses **`send_interface_message`** so transit toasts appear in the **message feed** (top-right of the screen). You get one message per planet transit, e.g. "Surya moved to Mesha". When a **friendly** planet (Chandra, Budh, Guru, Shukra) transits into **your** Rashi, the toast uses the good type and plays positive music; when an **enemy** planet (Mangal, Shani, Rahu, Ketu) enters your Rashi, the toast uses the bad type and plays suspense music.

Check the message list (envelope icon) if you don't see them immediately.

## Planet timers (transit)

Planet positions advance **once per game quarter** (not on a 91-day timer). The mod runs when the quarterly pulse fires with a **quarter guard**: it only runs if the current quarter (1–4, from `scope:quarter`) has not already been processed this year (globals `claw_transit_ran_q1`–`q4`, 1-year duration). So when the pulse fires repeatedly (e.g. while paused), we skip; when a new quarter starts, we run once and add 91 days to all planet day-counters, then each planet advances by its own rule (e.g. Surya ~30 days per rashi, Chandra ~3). They only run when:

1. **Vedic Astrology Scope** is **not** "Disabled" (use "Cosmic Law (Global)" or "Cultural Focus (Hindu Only)").
2. You **started a new game** with that setting (or loaded a save that had it): init runs only at game start; loaded games keep saved positions.
3. The mod hooks into the correct pulse: `quarterly_playable_pulse` (the game's name; not `on_quarterly_playable_pulse`).

To verify: turn **Cosmic Law Debug** **On**, start a new game, run a few quarters, and use **Debug: Show planet positions**. Positions should change over time (e.g. Chandra moves every ~3 days, Surya every ~30). You should also get a "Cosmic Law debug active" message each quarter in the message feed.
