# Cosmic Law (CK3) — Implementation Plan

**Mod name:** Cosmic Law  
**Concept:** Vedic astrology mechanics in Crusader Kings III  
**Repository folder:** `CK3_Cosmic_Law`

**Reference locations:**
- **Sample mods:** `c:\Users\PARTH\Documents\Paradox Interactive\Crusader Kings III\Mod\` (e.g. Cosmic_Law, Invite Debutantes).
- **Game files:** `d:\Games\Crusader Kings 3\` (e.g. `game\common\religion\religions\00_hinduism.txt`, traits, events).

---

## 1. Workspace & CK3 Conventions (Summary)

- **Sample mods** in the Paradox Mod folder and **vanilla game** under `d:\Games\Crusader Kings 3\game` are used for structure, scripting patterns, and triggers (e.g. `religion = religion:hinduism_religion`).
- **CK3 mod layout:** `.mod` file + mod folder; `common/` (traits, scripted_effects, on_actions, game_rules, scripted_variables), `events/`, `gfx/`, `localization/`.
- **Key hooks:** `on_birth` / `on_birth_child` (child scope), `on_game_start` / `on_game_start_after_lobby` (init); `on_quarterly_playable_pulse` or similar for periodic transit; `on_post_birth` (baby scope).
- **Traits:** `common/traits/`; scripted assignment via on_actions; use `religion = religion:hinduism_religion` for Hindu-only scope when game rule is Religious Focus.
- **Game rules:** `common/game_rules/`; all mod logic checks `has_game_rule` (e.g. `cosmic_law_scope_off`, `cosmic_law_scope_cultural`, `cosmic_law_scope_global`).

---

## 2. Design Decisions (Locked)

1. **Rashi = Moon sign**  
   In Vedic astrology the birth sign used for the character is the **Moon sign (Rashi)**, not the Sun sign. The trait represents the character’s **Janma Rashi** (Moon sign at birth).

2. **Rashi assignment at birth**  
   **Purely random** (1/12). No date-based calculation to avoid heavy operations.

3. **Planet transit timing**  
   - **At new game start:** All 9 planets are placed in a **random Rashi (1–12)**. This state is stored in **global variables** (or equivalent) so it **does not change when the game is loaded again** (persistent per save).
   - **After start:** Each planet has a **fixed transit interval in days**. Transits are implemented in **days** (not months/years) for simplicity. Approximate reference intervals:
     - **Chandra (Moon):** ~2.25–2.5 days → use **3 days** per Rashi.
     - **Surya (Sun):** ~1 month → **30 days**.
     - **Mangal (Mars):** ~45 days (use **45**).
     - **Budh (Mercury):** ~1 month → **30 days**.
     - **Guru (Jupiter):** ~1 year → **360 days**.
     - **Shukra (Venus):** ~1 month → **30 days**.
     - **Shani (Saturn):** ~2.5 years per sign → **912 days** (~2.5 × 365).
     - **Rahu / Ketu:** ~1.5 years each → **548 days** (or 540).

4. **Sade Sati**  
   **2.5 years per phase** (~912 days per phase). Use **days** internally for consistency with transits.

5. **Hindu / Religious Focus**  
   Use **`religion = religion:hinduism_religion`** to identify Hindu characters. The religion key is defined in `game/common/religion/religions/00_hinduism.txt` as `hinduism_religion` (faiths under this religion group are Hindu).

---

## 3. Architecture Overview

- **Single namespace** for events (e.g. `cosmic_law` or `cl`).
- **Central scripted effects** for: “should astrology apply to this character” (game rule + religion if Religious Focus), “assign Rashi at birth”, “apply planet-in-rashi modifiers”, “Sade Sati phase”.
- **Variables / flags:** Store per-character Rashi (trait); store per-game planetary positions (e.g. global flags or saved scope); store Sade Sati phase on character (flag or modifier).
- **Mod options (game rules)** drive all behavior; no astrology when Disabled; Religious Focus = Hindu-only; Cosmic Law = all characters.

---

## 4. Data Structure Design

### 4.1 Rashis (12 signs) — Moon sign (Janma Rashi)

- **Semantic:** Each character’s Rashi trait is their **Moon sign (Janma Rashi)** in Vedic terms, not the Sun sign.
- **Internal IDs:** e.g. `rashi_mesha`, `rashi_vrishabha`, … `rashi_meena`.
- **Data:** One trait per Rashi; optional `common/scripted_values` or static script lists for Rashi lord, Mitra, Shatru, neutral (see 4.3).
- **Rashi lord (Swami) mapping:**  
  Mesha→Mars, Vrishabha→Venus, Mithuna→Mercury, Karka→Moon, Simha→Sun, Kanya→Mercury, Tula→Venus, Vrishchika→Mars, Dhanu→Jupiter, Makara→Saturn, Kumbha→Saturn, Meena→Jupiter.

### 4.2 Navagraha (9 planets)

- **IDs:** `surya`, `chandra`, `mangal`, `budh`, `guru`, `shukra`, `shani`, `rahu`, `ketu`.
- **Storage:** Global variables (e.g. `global_<planet>_rashi` = 1–12, `global_<planet>_days` = days spent in current Rashi). Stored in save, so **positions do not change on reload**.
- **Transit (day-based):** Each planet has a **fixed transit length in days** per Rashi. On a periodic pulse (e.g. quarterly), add pulse length in days to each planet’s “days in current Rashi”; when `days >= transit_days`, advance Rashi (1–12 wrap) and reset (or carry over) days. Fast movers (e.g. Moon) may advance multiple times per pulse.

**Transit days per planet (reference):**

| Planet   | Approx. real | In-game days |
|----------|--------------|--------------|
| Chandra  | ~2.5 days    | 3            |
| Surya    | ~30 days     | 30           |
| Mangal   | ~45 days     | 45           |
| Budh     | ~30 days     | 30           |
| Guru     | ~1 year      | 360          |
| Shukra   | ~30 days     | 30           |
| Shani    | ~2.5 years   | 912          |
| Rahu     | ~1.5 years   | 548          |
| Ketu     | ~1.5 years   | 548          |

### 4.3 Planet–Rashi relations (extensible)

- **Tables (in script or in data):**
  - **Sign lord:** Which planet rules which Rashi (above).
  - **Mitra (friend) / Shatru (enemy) / neutral:** Per planet, list of Rashis as friend/enemy/neutral (or list of planets that are friend/enemy to current sign lord).
- **Modifier keys:** e.g. `cosmic_planet_in_own_sign`, `cosmic_planet_in_friend_sign`, `cosmic_planet_in_enemy_sign`, `cosmic_planet_in_neutral_sign` (optional: per-planet modifiers for future use).
- **Extensibility:** Same pattern can later host Neech/Uchcha (debilitation/exaltation) by adding more modifier types and trigger conditions.

---

## 5. Trait System Plan (Zodiac / Rashi = Moon sign)

- **12 traits:** One per Rashi (Moon sign / Janma Rashi); category e.g. `lifestyle` or custom; `genetic = no`; not education.
- **Assignment:**
  - **At birth:** `on_birth` / `on_birth_child` effect that checks game rule (and, if Religious Focus, `religion = religion:hinduism_religion`); if in scope and character has no Rashi trait, assign **one random** Rashi (1/12).
  - **At game start:** Same effect run for every living character (e.g. from `on_game_start`); only assigns if no Rashi yet and scope applies.
- **Visibility:** Traits appear in character view; use standard trait localization and icons in `gfx/interface/icons/traits/`.
- **Effects per Rashi (examples; tune in implementation):**
  - Small stat modifiers (e.g. `martial`, `diplomacy`, `stewardship`, `intrigue`, `learning`) and/or `opinion_modifier` with same-faith / same-Rashi.
  - Optional: `triggered_opinion` toward other characters (e.g. same Rashi = small positive; optional opposite element = slight negative).
- **Personality:** No hard link to personality traits; optional events or modifiers that reference both Rashi and personality for flavor (e.g. “Wrathful + Mesha” flavor text).

---

## 6. Planet System Plan (Navagraha)

- **Initialization:** On **game start only** (`on_game_start` / `on_game_start_after_lobby`), one effect that:
  - Runs only when game rule is not Off (global or cultural).
  - For each of the 9 planets: set `global_<planet>_rashi` to a random 1–12 and `global_<planet>_days` to 0. **Do not** re-run this on load; global variables persist in the save, so positions stay fixed per playthrough.
- **Transit (day-based, periodic):**
  - Use the **shortest available global pulse** (e.g. `on_quarterly_playable_pulse` ≈ 91 days). Each tick: add the pulse length in days to each planet’s `global_<planet>_days`.
  - For each planet, when `days >= transit_days`: advance `global_<planet>_rashi` (12 → 1), subtract `transit_days` from days (or set to remainder). For fast movers (e.g. Moon, 3 days), repeat advance/subtract up to a safe cap per tick (e.g. Moon: up to 30 advances per 91-day pulse).
  - Single lightweight effect; no per-character iteration in the transit tick.
- **Transit intervals:** Defined in script as days per planet (see table in 4.2). Sade Sati uses 912 days (~2.5 years) per phase.

---

## 7. Planet–Rashi Interaction (Modifier & Event Framework)

- **When a planet is in a Rashi:** For each character in scope (respecting game rule and religion):
  - Determine “relationship” of that planet to the character’s Rashi: Swami (lord), Mitra, Shatru, neutral.
  - Apply **one modifier per active planet** (or one combined “cosmic influence” modifier) from a small set: e.g. `cosmic_planet_friendly`, `cosmic_planet_enemy`, `cosmic_planet_neutral`, `cosmic_planet_lord`.
- **Application strategy:**
  - **Option A (recommended):** On **yearly pulse**, for each character (or only rulers/playable), run a scripted effect that (1) reads current planet positions, (2) computes net effect per character (e.g. count of friendly/enemy planets), (3) applies or refreshes one combined modifier. Avoids 9 modifiers per character.
  - **Option B:** Apply up to 9 small modifiers per character (one per planet); more granular but noisier.
- **Opinion and events:** Same scripted logic can add temporary opinion modifiers (e.g. “Shani in enemy sign” with vassals) and feed into **random_events** or targeted events (e.g. “Jupiter in friend sign” positive event). Keep event weights low to avoid spam.
- **Extensibility:** Same “planet in Rashi + relation to character’s Rashi” pipeline can later feed Neech/Uchcha, Drishti, and Bhava by adding more conditions and modifier types.

---

## 8. Kundali (Birth Chart) UI

- **Data to show:** Character’s Rashi (trait); current positions of 9 planets (read from global state); optional Sade Sati phase for this character.
- **CK3 GUI:** Use the existing **Interface** system (`.gui`/`.gfx` in `gfx/interface/`). Add a **widget/panel** that can be toggled (e.g. from character window or a decision).
- **Implementation approach:**
  - **Window:** New window or sub-view (e.g. “Kundali” button on character sheet) that opens a layout: character portrait, list of Rashi + 9 planets with current Rashi name (and optional icon).
  - **Data binding:** Use **scripted gui** or **scripted triggers** that pass character scope and read variables/flags for planet positions; display via localized text (e.g. “Surya in Simha”).
  - **Mod option:** “Show Kundali” game rule or a separate “Kundali panel” option that hides the button or the entire window when off.
- **Placeholder:** First deliverable can be a **decision** “View my Kundali” that opens a **character event** with description listing Rashi and planet positions (no custom .gui yet); full GUI in a later branch.

---

## 9. Sade Sati Event Chain

- **Condition:** Saturn’s current Rashi is 12th, 1st, or 2nd from the character’s **Moon Rashi**. The character’s **Rashi trait is their Moon sign** (Janma Rashi), so we use that directly—no separate Moon sign.
- **Phases (2.5 years ≈ 912 days each):**
  - **Phase 1 (12th from Moon):** Modifier (e.g. stress gain, small gold/health malus); optional event at phase start and one random event during phase.
  - **Phase 2 (1st from Moon):** Stronger modifier (e.g. stress, prestige/opinion malus); one or two events (e.g. “obstacles at court”).
  - **Phase 3 (2nd from Moon):** Recovery modifier (e.g. small stress loss over time); one positive-flavor event.
- **Detection:** In the same periodic logic that applies planet–rashi effects, for each character compute Saturn’s Rashi vs character’s Rashi (mod 12); if Saturn is in 12th, 1st, or 2nd from character’s Rashi, set Sade Sati phase (1/2/3) and apply the corresponding modifier; otherwise remove it.
- **Duration:** Saturn’s transit is **912 days per sign** (day-based); phase changes when Saturn moves. Time-based and dynamic.

---

## 10. Mod Options (Game Rules)

- **Location:** `common/game_rules/` (e.g. `claw_game_rules.txt`).
- **Rule: “Cosmic Law scope”** (e.g. `claw_scope`):
  - **Disabled** (`cosmic_law_scope_off`): No Rashi assignment, no planet init, no transit, no Sade Sati, no Kundali.
  - **Religious Focus** (`cosmic_law_scope_cultural`): Only characters with **`religion = religion:hinduism_religion`** get traits, modifiers, events, and Kundali. This covers all Hindu faiths (e.g. vanilla Hindu faiths under `hinduism_religion`).
  - **Cosmic Law (Global)** (`cosmic_law_scope_global`): All characters regardless of religion.
- **Optional second rule:** “Show Kundali panel” (yes/no) for UI.
- **Implementation:** Every scripted effect and event checks the game rule; when scope is Cultural, also require `religion = religion:hinduism_religion` on the character.

---

## 11. Event Framework Structure

- **Namespaces:** One, e.g. `cosmic_law` or `cl`.
- **Files (suggested):**
  - `events/cosmic_law_birth_startup.txt` — on_birth (assign Rashi), on_startup (assign Rashi for existing, init planets).
  - `events/cosmic_law_yearly.txt` — yearly pulse: update planet positions; apply planet–rashi modifiers; Sade Sati detection and phase/removal.
  - `events/cosmic_law_sade_sati.txt` — Sade Sati flavor events (phase 1/2/3).
  - `events/cosmic_law_planet_flavor.txt` — optional random events for planet–rashi combinations (low weight).
- **Scripted effects:** `common/scripted_effects/cosmic_law_effects.txt` (or split): assign_rashi, should_apply_cosmic_law, get_planet_rashi_relation, apply_planet_modifiers, update_planets, sade_sati_phase_effect.
- **Scripted triggers (optional):** `common/scripted_triggers/cosmic_law_triggers.txt`: cosmic_law_enabled_for_character, is_in_sade_sati, etc.

---

## 12. GUI Implementation Strategy

- **Phase 1:** Decision “View my Kundali” → character event with text listing Rashi + 9 planets (no new .gui).
- **Phase 2:** Custom window in `gfx/interface/`: new .gui with a simple layout (character + list of 10 lines: Rashi, then 9 planets with current sign). Use scripted gui to fill text. Toggle visibility via Mod Option (game rule or option).
- **Icons:** Reuse or add small icons for 12 Rashis and 9 planets in `gfx/interface/icons/` (e.g. traits or custom folder).

---

## 13. Performance Considerations

- **Avoid:** Global pulses that loop over every character every tick; avoid hundreds of modifiers per character.
- **Do:** Use **on_yearly_pulse** (or less frequent) for planet transit and for applying/refreshing modifiers; limit “who gets checked” (e.g. only rulers, or only characters with a Rashi trait); use one combined “cosmic influence” modifier per character where possible; keep Sade Sati and planet–rashi checks in one place per year.
- **Planet update:** One event per year that only updates 9 positions (flags/variables); no character iteration in that event. Separate logic for “apply to characters” can run in same or another yearly event with a reasonable scope (e.g. all characters with Rashi trait, or only courtiers in player’s court + AI rulers).

---

## 14. Future Extensibility (Designed In)

- **Guru Chandal Yog:** Add trigger “Jupiter + Rahu in same Rashi (or specific house)” and a modifier/event; store positions per planet so “same Rashi” is a simple comparison.
- **Neech/Uchcha:** Extend planet–rashi table with “debilitation” and “exaltation” Rashis; in modifier application, check for these and apply stronger debuff/buff.
- **Mahadasha:** Add a “current dasha” variable per character (or per save); advance with game time; modifiers/events keyed off dasha lord and position.
- **Raj Yog:** Define combinations (e.g. lord of 1st + lord of 10th in same sign); check from stored positions and Rashi lords; apply special modifier.
- **Drishti (aspects):** Define which Rashis “aspect” which (e.g. 7th from planet); from planet positions compute aspects and add aspect-based modifiers.
- **Bhava (houses):** If we add “ascendant” or “house 1” per character, same position storage can drive house-based logic; keep planet positions in a single source of truth.

All of the above can be added without reworking the core: same global planet state, same per-character Rashi (and optional Moon Rashi), same yearly application pipeline.

---

## 15. Suggested Additional Features

- **Rashi compatibility in marriage:** Spouse opinion (Mitra/Shatru/same Rashi) is applied yearly for married couples when Cosmic Law is enabled. To affect **marriage acceptance** (AI willingness to accept a proposal), the game’s `marriage_ai_accept_modifier` would need to be overridden in full and a modifier block added that uses scripted triggers `claw_scope_has_mitra_rashi_to_root` (add acceptance) and `claw_scope_has_shatru_rashi_to_root` (subtract acceptance) with `scope:secondary_actor` and `scope:secondary_recipient` as the couple; the mod does not override that file by default.
- **Astrology council task:** “Consult the stars” giving a temporary bonus or event based on current transits.
- **Festival/ritual events** on “auspicious” planetary combinations (e.g. Jupiter in exaltation).
- **Nicknames or titles** tied to Sade Sati or strong planetary influence (e.g. “Shani’s child” after Sade Sati).

---

## 16. Deliverables Order (Feature Branches / PRs)

**Debugging:** A game rule **Cosmic Law Debug** (e.g. `claw_debug`) is added in the foundation. When enabled, in-game debugging is available (decisions/events showing current Rashi, planet positions, modifiers) so you can verify behaviour without console.

| Order | Branch / PR | Content |
|-------|-------------|--------|
| 1 | `mod-foundation` | Mod skeleton: `.mod`, descriptor, folder structure; game rules (Cosmic Law scope + optional Kundali toggle); scripted triggers “cosmic_law_enabled_for_character”; no traits yet. |
| 2 | `traits-rashi` | 12 Rashi traits (definition, localization, icons); on_birth + on_startup assignment (random); respect game rule and Religious Focus. |
| 3 | `planets-navagraha` | Planet position storage (flags/variables); on_startup init (random); one yearly event to advance transits (single speed); no character effects yet. |
| 4 | `planet-rashi-interactions` | Rashi lord + Mitra/Shatru data; scripted effects to compute relation and apply one combined modifier per character; hook into yearly pulse; optional low-weight flavor events. |
| 5 | `rashi-relationships` | Mitra/Shatru triggers; Rashi lord (planet in own sign); Rashi compatibility opinion (spouse yearly); marriage spouse opinion; marriage acceptance override doc. |
| 6 | `kundali-view` | **Initial:** Character view only (e.g. decision “View my Kundali” + event, or panel in character window). Later: optional custom .gui and Mod Option to show/hide. |
| 7 | `sade-sati` | Sade Sati phase detection (Saturn 12th/1st/2nd from Moon Rashi); 3 modifiers + 3 phase events; stress/recovery effects; time-based. |
| 8 | `polish-localization` | Full localization pass; balance numbers; Rashi compatibility in marriage (opinion/acceptance modifier); doc/README update. |

---

## 17. File and Folder Layout (Target)

```
CK3_Cosmic_Law/
├── descriptor.mod
├── Cosmic_Law.mod          # or same name as folder
├── common/
│   ├── game_rules/         # 00_cosmic_law_rules.txt
│   ├── scripted_effects/   # cosmic_law_*.txt
│   ├── scripted_triggers/  # cosmic_law_triggers.txt (optional)
│   ├── on_actions/         # 00_cosmic_law_on_actions.txt
│   ├── traits/             # cosmic_law_rashi_traits.txt
│   ├── modifiers/          # (if needed) cosmic_law_modifiers.txt
│   └── character_interactions/  # (optional) e.g. View Kundali
├── events/
│   ├── cosmic_law_birth_startup.txt
│   ├── cosmic_law_yearly.txt
│   ├── cosmic_law_sade_sati.txt
│   └── cosmic_law_planet_flavor.txt
├── gfx/
│   └── interface/
│       ├── icons/          # traits, maybe planets
│       └── (kundali .gui in Phase 2)
├── localization/
│   └── english/
│       └── cosmic_law_l_english.yml
└── docs/
    └── IMPLEMENTATION_PLAN.md  (this file)
```

---

## 18. Next Steps

1. **You:** Answer the clarifying questions in **Section 2**.
2. **You:** Confirm or adjust the plan (e.g. Sade Sati using birth Rashi as Moon Rashi, transit speeds, Hindu faith list).
3. **After approval:** Implement in order of **Section 16**, each major feature in its own branch and PR.
4. **Repo name:** `CK3_Cosmic_Law` (corrected).

---

*End of Implementation Plan*
