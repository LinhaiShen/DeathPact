# Codex implementation plan — atomic milestones, no big-bang rewrite

Codex should complete one PR-sized milestone, test it, and report actual changed files. Start from a pinned original DevilutionX fork. Milestone progression depends on tests (do not skip Phase 0).

## Phase 0 — Upstream baseline and compatibility audit
**Task P0-01**: Clone/open permitted fork, pin upstream SHA, run clean native build and original tests. Record build instructions and checksums; inspect LICENSE/asset requirements. **Do not change Gameplay.**
**Task P0-02**: Produce `docs/UPSTREAM_AUDIT.md` with real entry points for monster actions, death, native save, item use/pickup, spell book/hotkeys, multiplayer synchronization, class creation, and class-dependent stat caps. Document 1–4 player transport realities.
**Acceptance:** original single/multiplayer smoke tests successful or named blocking failure; audit cites paths/real symbols; no guessed APIs.

## Phase 1 — Domain foundations with no invasive combat changes
**Task P1-01**: Define common canonical IDs and config structure (`characterId`, `roomSessionId`, `computerSpawnId`, `skillId`, `itemId`, operationId`, `rulesetVersion`). Unit tests for IDs and retry idempotency.
**Task P1-02**: Pure domain tests for Computer observed Skill ID set and fixed Learning +1 on kill, Follow candidate, no hard 30-Skill limit (BurdenCap<=30; CurrentBurden>30 legal and invokes capped Misfire), uniform classless gameplay, vanilla Book +1/consumption, Momentum, Loot Load, HC drops.
**Acceptance:** repeated Computer cast 1/10/100 times yields Learning +1 on kill, only for pre-Following Character; CurBurden 30->31->32 legal Over-Burden/Misfire capped50%; Book may acquire new Skill at Burden30+. Internal HeroClasses use identical rules; abandonment does not refund Points.

## Phase 2 — Central persistence skeleton (single service)
**Task P2-01**: Account auth and **globally unique public Account `Name#Tag`** (`publicName`, `publicTag`, initial random three-digit Tag with collision retry); private per-Character names/selection, committed Character save, account ownership, status/revisions and DB migrations. No public Character tag. APIs distinguish own-character details from public Account identity.
**Task P2-02**: Item Registry and atomic transfers, UTC Loot Load, native Book consume-one/true Level+1 as immediate server transaction; other/new Skills allowed at Burden>=30. Deny ineffective already-maxed same-Spell book only.
**Task P2-03**: Native Save adapter for attributes, level, inventory, equipment, stash; build manifest/hash checkpoints (login, entering combat zone, settlement, reconnect). Clearly label security limits.
**Acceptance:** stale revision, forged item ID, duplicate save request and concurrent pickup rejected; saved character restorable; changing local Pending Knowledge cannot push it via Native Save.

## Phase 3 — Actual-use Knowledge and Battle Settlement
**Task P3-01**: Verify actual executed Computer actions; map action context to canonical Skill ID and add to per-spawn usedSkillIds SET. Never count casts, missiles, hits or bearer level. Combat untouched.
**Task P3-02**: On Computer death each observed mapped Skill gives exactly Pending Learning +1 to pre-Following eligible Character, candidate-only otherwise; dedup by room+spawn+Skill. Tests 1/2/4 players and 1/10/100 casts.
**Task P3-03**: Settlement Follow choices, voluntary abandonment and Skill Point spending atomically/idempotently; recompute Burden with no hard 30-Skill validation, higher Burden invokes Misfire. Clear only learned-Spell hotkeys and never refund spent Skill Points.
**Acceptance:** no retroactive Follow credit, next eligible kill +1; no 30-Skill cap, abandonment lowers Burden but not required for next Skill. No Knowledge/save replay/refund or network retry duplicates.

## Phase 4 — Burden, Momentum, forgetting / hotkeys
**Task P4-01**: Compute union of Following/ Learned distinct skills, MAG-invested cap config; integrate one Misfire roll per legitimate learned-skill cast, charge Mana once, suppress effect.
**Task P4-02**: Global Momentum increases with configured valid combat, regenerates Mana; baseline natural Mana regen=0, outside-combat decay and settlement reset. Leave native action speed untouched.
**Task P4-03**: Memorized Spell hotkeys protect from automatic forgetting; Settlement abandonment clears learned-Spell shortcuts only, preserves Scroll/Charges. Native Book consumption/+1 Level requires server atomic commit; allow other/new Skills at Burden>=30.
**Acceptance:** Burden/Misfire soft cap despite CurrentBurden>30, no native attack timing changes, Hotkey protection and Scroll/Charges independence, Book/Follow acquisition at Burden30+, no refund after abandonment.

## Phase 5 — Multiplayer social/lobby and shared progression
**Task P5-01**: Account-based Friends, public/private/friend room list, invites, online status and chat; every peer-facing label uses **Account `Name#Tag`**, including native room/party/in-game overlays and Hardcore Legacy owner/Hall of Fame. Character names remain owner-only. Audit native DevilutionX hero name serialization/UI and test 2/4-player sessions for private-name leakage; keep stable `characterId` internally.
**Task P5-02**: Enforce room cap 4; Normal/Hardcore mode separation. Set server room creator lifecycle; on host termination close room, cancel/unclaim allowed HC death-loot pickups.
**Acceptance:** both existing TCP/ZeroTier transports behave as documented, no cross-mode party and correct kill credit; each Account's own character selector shows its Character privateName, while other Accounts see only unique public Account `Name#Tag` in roster, game, chat, invitations, legacy provenance and public rankings. Character names never leak through peer-visible name fields/messages.

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
