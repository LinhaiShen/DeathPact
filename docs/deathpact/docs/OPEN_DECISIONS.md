# OPEN / not frozen — Codex must not invent rules

This document distinguishes `FROZEN`, `PROPOSED`, and `OPEN`. Implement tests of frozen invariants now; expose missing values as config or make small scaffolds until product decisions are made. Do not let placeholders become silent design decisions.

## Frozen numbers and choices
| Key | Value/status | Explanation |
|---|---|---|
| `knowledge.max_distinct_skills` | **30 FROZEN** | Following union Learned; one burden/skill |
| `burden.initial_cap` | **10 FROZEN (tentative numeric)** | initial V1 balance, config key |
| `burden.endgame_cap` | **30 FROZEN target** | achievable by allocating all possible level-up stat points to MAG at max level |
| `burden.cost_per_skill` | **1 FROZEN** | no double count for Follow+Learned |
| `burden.misfire_max_probability` | **0.50 FROZEN** | hard cap even at huge overflow |
| `burden.misfire_probability` | **min(0.50, max(0, burden-cap)/cap)** | zero if <= cap; Mana consumed on misfire |
| `momentum.instances_per_character` | **1 FROZEN** | no group variation |
| `mana.base_natural_regen_per_second` | **0 FROZEN** | native potions can still recover |
| `knowledge.max_credit_per_computer_skill_kill` | **1 FROZEN** | effective cast events deduped |
| `knowledge.group_reward` | **all active party members FROZEN** | same kill candidate, own Follow state |
| `spellbook.max_level_granted` | **Lv1 FROZEN** | no level gain at Lv1+ |
| `forget.manual_only_in_settlement` | **true FROZEN** | voluntary abandonment only inside Battle Settlement, never a standalone forget action |
| `forget.abandon_resets_to_level` | **0 FROZEN** | remove Follow, true Learned Level and all learning progress; clear hotkeys and Burden |
| `loot.daily_reset` | **UTC 00:00 FROZEN** | character level |
| `loot.limit`, `loot.weights` | **200; Normal1 Magic3 Unique10** | explicitly *initial values to balance* |
| `hardcore.equipped_drop_fraction` | **0.65 FROZEN** | random round; server chooses; eligible co-present HC only |
| `pact.once_per_character` | **true FROZEN** | activation registered before effect; certain death |

## Open Gameplay/technical values
| ID | Missing decision | Safe Codex action |
|---|---|---|
| OPEN-01 | `BurdenCap(MAG allocated)` exact progression; vanilla maximum level/total points and classless point caps | Implement monotone configurable mapping with 10→30 endpoints; confirm actual max level from pinned engine and ask before locking curve |
| OPEN-02 | Per-kill skill level credit amount, player skill-level thresholds, teacher-vs-student restriction, max true skill level | Versioned data-driven tables; tests for no retroactive Follow + one credit per killed Computer+Skill |
| OPEN-03 | Computer action/spell ID → canonical Skill ID and Level | Audit engine code, fill table only from verified IDs; unmapped ignored |
| OPEN-04 | Whether a regular physical attack is teachable as a Skill (if there is no true player-equivalent Skill) | Leave unmapped until confirmed; no blanket mapping |
| OPEN-05 | Skill Point accrual from leveling; whether native XP cap unchanged | Wire allocator and budget invariant; do not change base XP curve without decision |
| OPEN-06 | Auto-forgetting thresholds Fading/Forgotten, following Lv0 cleanup, exact server-time update semantics | Keep durations config; no default that silently erases user skills |

| OPEN-07 | Momentum gain trigger, max, decay delay/shape, Mana Recovery curve | Config hooks; test zero baseline and settlement reset; no arbitrary temporary rates shipped as final |
| OPEN-08 | Misfire applicability to Scroll, Staff Charges, original innate/class skills | Intercept permanent learned Knowledge casting only as safe default; document original behavior until reviewed |
| OPEN-09 | Book consumed or retained when already Lv1+; full-30 Book action and immediate online commit recovery | Block level increase; use transactional item+knowledge change; UI must not silently waste a book without explicit product decision |
| OPEN-10 | Classless character starting stats/visual, starting loadout/spell, original profession-dependent quirks | Audit and present smallest change proposal; do not invent starting class |
| OPEN-11 | Hardcore death drop asset count rounding policy | Proposed stochastic rounding (expectation 65%); needs sample-value tests; note 65% exact is impossible for small counts |
| OPEN-12 | Death Pact buffs, Life drain rate, irrevocable server deadline, interaction with immunity/shields | Keep config/fixtures; always enforce death once Pact registered |
| OPEN-13 | Runtime handling of a server outage/forced disconnection during play | Fail closed on irreversible mutations; document possible local Pending loss; no silent rollback of server-final state |
| OPEN-14 | PvP zone detection, Hardcore ranking scope/format, original class-to-knowledge interaction | Audit current game modes; new combat rules only when requested |
| OPEN-15 | Persistent vs ephemeral state when Host quits, players fail to reconnect and TCP vs ZeroTier lifecycle differences | Server room owner is authoritative for *lobby lifecycle*; peers may behave differently, test this |

## Preserved product decisions that are NOT open
- **Voluntary skill abandonment is allowed only inside Battle Settlement, and previously spent Skill Points are permanently spent (no refund).** Players can deliberately free skill slots at 30 by abandoning an existing Following/Learned skill; on committed abandonment it becomes Lv0 with zero progress and no Follow or hotkey, and every later acquisition starts fresh. `Slot it, preserve it` prevents only **automatic** forgetting. No standalone Forget API/UI outside Settlement. Abandoning a skill does not cancel the Settlement itself.
- No custom CD/Momentum group; Momentum restores Mana only.
- Loot is free pickup with per-character rarity-weighted Loot Load; no new player-specific allocation system.
- Battle Settlement is not a general save, and Native Save is not a Knowledge Settlement.
- No browser prototype, replay, camera/animation rewrites, or superimposed 10-slot bar.

## Practical design-change procedure
When work hits an OPEN field: (1) add exact source/code observation, (2) propose minimum 1–2 alternatives with gameplay/performance impact, (3) ask owner for the narrow decision or preserve a feature flag/config, (4) then add acceptance tests. Do not fill gaps with stock Diablo assumptions if they violate `V1_SPEC.md`.
