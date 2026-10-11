# DevilutionX code integration/audit plan

## IMPORTANT
All paths below are **public upstream candidate files verified to exist in the public `diasurgical/DevilutionX` repository**, NOT a claim of stable functions or suitable insertion points. Codex MUST pin and inspect the actual revision before touching code. Current upstream changes over time.

Upstream browse:
- https://github.com/diasurgical/DevilutionX/blob/master/Source/monster.cpp — monster behavior/attacks; locate the **single semantic action execution** boundary (avoid multi-hit duplicate events)
- https://github.com/diasurgical/DevilutionX/blob/master/Source/player.cpp — player model, attributes, spell timing, class assumptions
- https://github.com/diasurgical/DevilutionX/blob/master/Source/items.cpp — generation, spell book and consumable handling, ground item generation
- https://github.com/diasurgical/DevilutionX/blob/master/Source/items.h — item serialization/misc IDs/requirements
- https://github.com/diasurgical/DevilutionX/blob/master/Source/inv.cpp — inventory interaction/loot pickup/use and equipment changes
- https://github.com/diasurgical/DevilutionX/blob/master/Source/pfile.cpp — original character-save entry points, serialization; **exclude pending Knowledge from auto-save**
- https://github.com/diasurgical/DevilutionX/blob/master/Source/sync.cpp — multiplayer state synchronization; inspect ownership and event IDs
- https://github.com/diasurgical/DevilutionX/blob/master/Source/dead.cpp — death presentation/handling; actual Player death transition may live elsewhere
- https://github.com/diasurgical/DevilutionX/blob/master/Source/diablo.cpp — loop/input/game lifecycle; refer only if actual search confirms hooks

## Phase 0 required audit report (`docs/UPSTREAM_AUDIT.md`, produced by Codex)
1. Upstream Git SHA, release tag, build platform/toolchain, license notices; successful clean build/test.
2. Actual Class/starting stats/max attribute cap/skill levels/level cap and character serialization fields; list confirmed symbols with file path and line ranges.
3. Actual monster attack/cast boundaries; demonstrate a test Computer action where exactly one event can be added without changing original hit/damage/missiles.
4. Engine spell/skill/Book/Scroll/Staff IDs and vanilla effective spell level calculation; identify which native skills have no grade and need mapping policy.
5. How native multiplayer assigns monster spawn identity and syncs death/actions; determine whether source action events are emitted identically on multiple peers and select canonical dedup identity.
6. Native save cadence and function(s), storage format and checksums; check whether auto-save writes spell hotkeys/levels and whether stash is shared, and prevent pending skill save leakage.
7. Native gear generation/seed/createInfo, rarity detection, item pickup and drop flow, data ownership, native item record dedup; note where central item registry can attach.
8. Vanilla multiplayer transport differences TCP vs ZeroTier, room owner exit vs peers continuing, join-in-progress support; no blanket assumptions.
9. Death detection, normal vs Hardcore behavior, item drop event ordering and existing resurrection, tests for late disconnect/host exit.
10. Record each `OPEN` setting and recommended low-intrusion defaults as **proposals** requiring approval if they change Gameplay.

## Preferred extension shape (proposal)
Add a distinct domain module and thin adapters rather than mixing entire service logic into monster/player code:
```
Source/arpg/knowledge/        // skill mapping, events, pending, following, forgetting
Source/arpg/combat/           // burden, misfire, single momentum/mana recovery
Source/arpg/persistence/      // save adapter, version/migrations, central REST/RPC client
Source/arpg/hardcore/         // death pact, death reporting, result-applier
Source/arpg/online/           // accounts/lobby/friends session adapter
Server/                      // one app, database migrations and integration tests
```
These directories are target suggestions, not currently confirmed upstream paths. Choose names consistent with repo conventions; minimize diff to upstream.

## Skill event mapping table
Define data in a versioned table, e.g. `config/skill_action_mapping.csv`, with columns:

`engine_action_key,engine_spell_or_attack_id,canonical_skill_id,skill_level_rule,knowledge_eligible,notes`

Map an **observed action execution**, NOT a monster type. If action has native level, capture native verified value; otherwise map action ID + game difficulty / monster level using an explicitly reviewed rule. Version mapping in ruleset. Include at least one spell and one ordinary attack only if a valid learnable player skill counterpart exists. Do **not** add blanket mappings from monster sprite name to skills or fill unknown IDs with fabricated data.

Two-step aggregation:
```
executed monster action -> SkillUsedEvent (unique castId)
  -> set of skill IDs used per specific computerSpawnId
  -> on ComputerDeath, ONE knowledge credit per skill in the set
  -> append same eligible credit/candidate to each current room party member's Pending
```
Replicated peers might see the same action; dedup by `(roomSessionId, computerSpawnId, castId)` before adding to per-kill set, then by `(roomSessionId, computerSpawnId, canonicalSkillId)` when awarding. Do NOT use monster type ID alone for computerSpawnId.

## Misfire / Book / Momentum interception notes
- Locate original Mana charge and spell release boundary; Misfire must be sampled once per attempted skill, Mana charged once even on failure, and original cooldown/action-lock timing unchanged. No double-roll on multiplayer echo or delayed animation callbacks.
- Locate Book item-use handler. **Current frozen contract:** Lv1-only grant with immediate server persistence. **Under review, not yet approved:** preserve vanilla successful Book consumption and +1 true level per book (even at Lv1+). Either policy must commit item consumption and Knowledge atomically, outside combat Pending/Settlement; reject over-cap/full-30 actions safely.
- Mana recovery tick must be deterministic/robust to delta time; base natural rate stays zero; recovery only from Momentum or existing explicit effects. No intervention in speed frames / attacks.
- Spell hotkey assignments act as **automatic** forget protection only; voluntary abandonment in Battle Settlement overrides protection. On acknowledged abandonment, clear memorized `SpellType::Spell` hotkeys and readied permanent learned-Spell reference only; **do not clear same-ID `SpellType::Scroll` / `SpellType::Charges` hotkeys**. Other spell-source shortcuts do not count as protection against automatic forgetting. Refresh spells/UI and prevent stale native save from restoring lost true level/progress/memorized Spell hotkeys. Capture hotkey change, actual skill use and UTC time safely; verify spell UI can show Following below Lv1 even if not castable.

## Save integration notes
- Keep raw vanilla character save as backwards-compatible as feasible; store extended metadata with explicit schema version.
- Maintain separate Knowledge committed state in central DB and separate volatile pending. Native save request must explicitly strip/mask pending skill gains, never merge pending progress or Follow into committed fields.
- Domain changes to attributes/equipment may trigger native Save; Knowledge itself must not save from ordinary autosave, death or logout.
- List all original places that can write character/spell state; add regression tests for each to avoid accidental settlement bypass.
- New asset grants, pickup or loss are audited against Item Registry, not just static stats recomputed from a client-provided snapshot.

## Engineering guardrails
- Retain upstream gameplay feel, animations, hit calculation, input, UI textures and networking. Instrumentation/event collection should be minimal and isolated.
- Tests run headless on gameplay domain logic where possible; compare original-engine behavior before/after hooks.
- Do not implement attack speed, spell cooldown or new inventory UI just to make Momentum/Burden work.
- Code review every delta touching multiplayer serialization: mismatched client versions must not silently corrupt player save or item packets.
