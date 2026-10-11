# Codex implementation plan — atomic milestones, no big-bang rewrite

Codex should complete one PR-sized milestone, test it, and report actual changed files. Start from a pinned original DevilutionX fork. Milestone progression depends on tests (do not skip Phase 0).

## Phase 0 — Upstream baseline and compatibility audit
**Task P0-01**: Clone/open permitted fork, pin upstream SHA, run clean native build and original tests. Record build instructions and checksums; inspect LICENSE/asset requirements. **Do not change Gameplay.**
**Task P0-02**: Produce `docs/UPSTREAM_AUDIT.md` with real entry points for monster actions, death, native save, item use/pickup, spell book/hotkeys, multiplayer synchronization, class creation, and class-dependent stat caps. Document 1–4 player transport realities.
**Acceptance:** original single/multiplayer smoke tests successful or named blocking failure; audit cites paths/real symbols; no guessed APIs.

## Phase 1 — Domain foundations with no invasive combat changes
**Task P1-01**: Define common canonical IDs and config structure (`characterId`, `roomSessionId`, `computerSpawnId`, `skillId`, `itemId`, operationId`, `rulesetVersion`). Unit tests for IDs and retry idempotency.
**Task P1-02**: Implement pure domain types/functions for Knowledge union/30 cap, `burdenCap`, Misfire hard-capped 50%, single Momentum, UTC Loot Load with rollover, Hardcore 65% randomized equipped-only selection. Keep compile-time or fixture tests independent of actual engine.
**Acceptance:** table-driven invariants pass, especially Cap10 Burden11→10% and Burden>=15→50%; 30 distinct can be reduced by Settlement abandonment, with no separate outside-Settlement forgetting API; abandoned Skill Points are never refunded; sampling does not duplicate.

## Phase 2 — Central persistence skeleton (single service)
**Task P2-01**: Account auth, Character creation / login, public tag, status, revisioned committed save; DB migrations and ownership rules.
**Task P2-02**: Item Registry + atomic item move; Loot Load ledger on direct Computer-drop pickup; Book use as an immediate atomic item-consumption/Knowledge transaction (current Lv1 grant; OPEN-09 +1-level-per-Book proposal awaits owner confirmation). Implement minimal central API client in desktop fork.
**Task P2-03**: Native Save adapter for attributes, level, inventory, equipment, stash; build manifest/hash checkpoints (login, entering combat zone, settlement, reconnect). Clearly label security limits.
**Acceptance:** stale revision, forged item ID, duplicate save request and concurrent pickup rejected; saved character restorable; changing local Pending Knowledge cannot push it via Native Save.

## Phase 3 — Actual-use Knowledge and Battle Settlement
**Task P3-01**: Build verified action/spell → Skill ID/Level mapping table from Phase 0. Hook original monster action start/execution for **one** SkillUsedEvent per action (not per damage tick); no impact on original combat output.
**Task P3-02**: On kill aggregate `(computerSpawnId, skillId)` once and grant candidate/progress to all current active party members. Persist **only as Pending**. Unit/integration tests with 1/2/4 players and two/ten casts by same Computer.
**Task P3-03**: Follow choices, **voluntary skill abandonment**, and skill-point allocations in the same settlement UI; atomic `KNOWLEDGE_SETTLE` with abandonSkillIds, post-abandonment 30-cap validation, hotkey cleanup + idempotent status retrieval, no cancellation of Settlement, failure recovery, reset Momentum. Normal Death discards Pending; no loss of committed Knowledge except an explicitly selected and successfully committed skill abandonment.
**Acceptance:** un-followed skill not retro-credited; follow then next kill credited; abandoned skill has Lv0/progress0/follow=false and released Burden/hotkeys after Settlement; reacquisition starts fresh, and previously spent Skill Points are not refunded; no network retry duplication or stale Native Save resurrection; settlement cannot be replaced by native save; a computer not actually using a Skill produces no candidate.

## Phase 4 — Burden, Momentum, forgetting / hotkeys
**Task P4-01**: Compute union of Following/ Learned distinct skills, MAG-invested cap config; integrate one Misfire roll per legitimate learned-skill cast, charge Mana once, suppress effect.
**Task P4-02**: Global Momentum increases with configured valid combat, regenerates Mana; baseline natural Mana regen=0, outside-combat decay and settlement reset. Leave native action speed untouched.
**Task P4-03**: Hook existing memorized-Spell Hotkey slots to protect learned skills from time-based automatic forgetting; implement **Settlement-only** voluntary abandonment and clear only learned `SpellType::Spell` hotkeys/readied learned Spell on commit; preserve Scroll/Charges hotkeys with the same SpellID. Do not expose forgetting outside Settlement. Book is currently Lv1-only by KN-06; OPEN-09 proposes restoring vanilla +1 level per consumed Book, requiring owner approval and the same immediate server-atomic item/Knowledge transaction.
**Acceptance:** Burden formulas; no side effects on vanilla cast frame rate, scroll/staff (until policy decided), damage/VFX when Misfire; multi-day clock tests and Hotkey protection against automatic forgetting; 30 max, ability to abandon one at Settlement and then acquire a new skill; no restoration of abandoned data.

## Phase 5 — Multiplayer social/lobby and shared progression
**Task P5-01**: Friends, public/private/friend room lists, invites, online status, chat; original TCP/ZeroTier session bridge with tests for join-in-progress and role.
**Task P5-02**: Enforce room cap 4; Normal/Hardcore mode separation. Set server room creator lifecycle; on host termination close room, cancel/unclaim allowed HC death-loot pickups.
**Acceptance:** both existing transport modes behave as documented; no cross-mode party; active party receives same kill candidate.

## Phase 6 — Hardcore / Death Pact / Legacy
**Task P6-01**: Add Hardcore Character irreversible dead-state transition and server-chosen equipment sample; complete item destruction, legacy provenance, eligible room-member pickup after ACK.
**Task P6-02**: Implement Death Pact Potion registration-before-use; once/lifetime, server irreversible deadline, temporary stats/Mana recovery buff, no healing, certain death. Pact-only Hall of Fame records.
**Task P6-03**: Test peer death, solo death, host exit before pickup, missing death packet, duplicate death request, concurrent item pickup, forced crash, offline death deadline.
**Acceptance:** dead cannot relog with old save; exactly-once drop result; no self/late/non-HC pickup; no drops survive terminated room; Pact cannot be activated during central outage or avoided by disconnect.

## Phase 7 — Integration, QA, release guardrails
- Automated 1/2/4-player multiplayer session scripts, genuine item gen vs modified seed, bad character level/stat allocations, inventory move duplicate, UTC rollover, timeout/retry, incompatible game version.
- Regress original DevilutionX controls/UI/combat/Mana/monster pathing and network play.
- Record performance deltas compared to upstream: startup, FPS, file size, CPU/memory, network volume, backend load. Combat adapter must not trigger central API for every attack or projectile.
- Deploy only after license/asset/legal release gate addressed. Online failure and limited anti-cheat plainly documented.

## First Codex prompt (paste verbatim)
> Start **Phase 0 only** of `docs/IMPLEMENTATION_PLAN.md`. Read `AGENTS.md`, `docs/README.md`, `docs/V1_SPEC.md`, and `docs/ENGINE_INTEGRATION.md`. Inspect the actual DevilutionX repository revision and build/test setup. Create `docs/UPSTREAM_AUDIT.md` with verified source locations and symbols for Computer action execution, skill/cast IDs, monster death, native character save, spell books/hotkeys, item pickup/registry seed fields, hardcore death handling and TCP/ZeroTier synchronization. Do not implement gameplay yet; do not guess function names. Report build/test result, compatibility risks and the smallest next coding PR.

## Prompt for each subsequent milestone
> Implement **Task Pn-xx only**, following `AGENTS.md` and the relevant docs. First identify impacted existing source files and acceptance tests, then implement the smallest change, run tests, and report what passed/failed. Do not add unrelated gameplay. Where the spec says OPEN, present the unresolved choice instead of inventing a value.
