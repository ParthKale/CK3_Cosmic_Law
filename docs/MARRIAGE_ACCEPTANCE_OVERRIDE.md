# Rashi compatibility and marriage acceptance

Spouse **opinion** (Mitra/Shatru/same Rashi) is applied **yearly** for married couples when Cosmic Law is enabled. No file override is required.

To also affect **marriage acceptance** (AI willingness to accept a proposal), the game’s scripted modifier `marriage_ai_accept_modifier` must be overridden. The game only loads one definition per name, so you must copy the full vanilla file and add the following modifier blocks inside `marriage_ai_accept_modifier`.

**Where:** `common/scripted_modifiers/00_marriage_scripted_modifiers.txt` (copy from game, then add the blocks below inside the `marriage_ai_accept_modifier = { ... }` block).

**Modifier blocks to add (e.g. after the “Same language” modifier):**

```
	# Cosmic Law: Rashi compatibility (Mitra/Shatru)
	modifier = {
		add = 15
		desc = claw_rashi_mitra_marriage_acceptance
		scope:secondary_actor = {
			scope = scope:secondary_recipient
			claw_scope_has_mitra_rashi_to_root = yes
		}
	}
	modifier = {
		add = -15
		desc = claw_rashi_shatru_marriage_acceptance
		scope:secondary_actor = {
			scope = scope:secondary_recipient
			claw_scope_has_shatru_rashi_to_root = yes
		}
	}
```

Add localization keys `claw_rashi_mitra_marriage_acceptance` and `claw_rashi_shatru_marriage_acceptance` for the `desc` strings. The mod does **not** override the marriage scripted modifier by default, so marriage acceptance is unchanged unless you add this override.
