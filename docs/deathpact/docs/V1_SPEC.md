# DevilutionX ARPG — V1 Product & Gameplay Specification

**Status:** V1 design baseline, 2026-10-10. **Goal:** 1–4 player native desktop cooperative, classless Diablo-like ARPG built by extending DevilutionX, with Knowledge/Settlement progression and modest central online services. This is NOT the earlier browser/Three.js architecture.

## 0. Requirement status and source of truth
- `FROZEN`: explicitly selected V1 rule. Only change by recorded design decision.
- `OPEN`: not determined; may be configurable but Codex must not silently finalize it.
- `DEFERRED`: not in V1. Do not implement without approval.
- `ORIGINAL`: retain behavior of the pinned DevilutionX fork, except explicit changes listed here.
- **Server authority** is for accounts, committed saves, registered items/loot ledger, settlement operations, Hardcore death registration; **NOT** for the real-time combat simulation. Client/peer integrity checks provide limited protection, not proof.

## 1. V1 product boundary
`FROZEN` Native DevilutionX graphics, world generation, enemies and their original behavior, original STR/MAG/DEX/VIT, Mana, attack/cast sequencing, original spells, item rarity/type/affixes/resistances/durability/identification, equip slots and inventory footprints, mouse/keyboard, hotkeys, PvE networking primitives. Preserve core engine rather than transplanting prior design.

`FROZEN` 1–4 players; player-hosted room/peer engine and central account, character, friends, lobby, chat, save, knowledge/settlement, loot ledger, item identity, and Hardcore services. Single backend application plus relational DB; no microservices.

`DEFERRED/EXCLUDED` browser/WebGPU/Three.js, original custom 10-slot UI, Attack Move/Shift+A control scheme, custom Cooldown system/group, dual-wield redesign, new armor slots, Rare quality/runes, numeric elemental immunity redesign, Rift, auction house, equipment trading product UI, replays, host migration, authoritative game server, heavy anti-cheat, PvP outside designated zones.

## 2. Character and classless
**CHAR-01** `FROZEN` Account has multiple Characters. Friends belong to Account; private email is not public. Character has unique public `Name#Tag` (initial tag random three digits, retry collisions; storage supports future longer tag). Character identity, stats, inventory, stash, Knowledge, Burden, Loot Load, Hardcore flag, save revision are individual.

**CHAR-02** `FROZEN` Keep valid internal Player::_pClass/HeroClass for engine, graphics, animation, save/network ONLY; do NOT remove it. **No class-specific gameplay effects whatsoever**: identical starting stats/skills/loadout, caps/growth, Life/Mana, damage/defense, weapon/handedness privileges, item/Skill eligibility for all Characters. Keep original STR/MAG/DEX/VIT and ordinary equipment/Book stat requirements. Shared classless numeric choices/cosmetic mapping remain OPEN-10 and apply equally to everyone.

**CHAR-03** `OPEN` Pick one neutral shared starting stats/loadout/innate-skill policy, common caps and HP/Mana/damage/weapon coefficients, and valid internal HeroClass/sprite mapping; never select a native profession with hidden mechanical advantage. Native max character level remains until separately changed.

## 3. Knowledge/SkillUsedEvent
**KN-01** `FROZEN` Computer = non-player enemy. Do not preconfigure a teachable Skill list. When a native Computer action actually executes, map its verified action context to canonical learnable Skill ID and add it to that Computer spawn's **usedSkillIds set**. Repeated executions/missiles do not count or increase credit. Unmapped and only-planned actions do nothing.

Conceptual observation: ComputerSkillObserved(roomSessionId, computerSpawnId, canonicalSkillId, rulesetVersion). Store only SET membership per specific spawn, not castId/cast count, bearer level or per-projectile events. Peer observations should merge idempotently; preserve original combat.

**KN-02** `FROZEN` On actual Computer death, for each distinct mapped Skill in its usedSkillIds set, make exactly one kill opportunity keyed by (roomSessionId,computerSpawnId,skillId). Every eligible Character who was Following before kill receives **Learning +1 exactly**, independent of 1/100 casts, projectiles or teacher skill level. Stable unique spawn IDs must prevent repeat credit across peers/respawn.

**KN-03** `FROZEN` Eligible party members all receive the same Computer-derived Knowledge candidate/credit; no last-hit ownership, no damage-contribution requirement, no proximity bonus, provided they are active members of that room's party at the kill event. Each Character maintains their own Follow and progress. Kills are not historical retroactive credits for players who join later. On a new Computer kill new members may qualify.

**KN-04** `FROZEN` If Following BEFORE Computer kill, receive Pending Learning +1 for each mapped actually used Skill. Otherwise the skill appears as Follow candidate at Settlement but **no retroactive learning**; later kills grant +1. No monster-level weighting or cast counting.

**KN-05** `FROZEN` Computer-derived credit is fixed Learning +1 per dead Computer/distinct used mapped Skill/pre-Following eligible Character. No bearer-level scaling, teacher-student ceiling, or cast frequency bonus. Learning threshold to gain TRUE Skill Level remains OPEN-02. Skill Points may be spent only on Following skills, as part of Battle Settlement.

**KN-06** `FROZEN` Preserve native DevilutionX Spell Book: each successful reading consumes one Book and immediately increases **true** Spell Level +1, including Lv1->Lv2, up to native MaxSpellLevel. Commit Book consumption and level in one immediate idempotent server transaction, NOT combat Pending/Settlement; Book does not auto-Follow. Avoid consuming a Book for that SAME Spell once maxed; **other/new Skills are still acquirable even with CurrentBurden>=30**. New 31st+ Skill increases Burden/Over-Burden/Misfire, never blocked by 30.

**KN-07** `FROZEN` No 30-distinct-Skill hard cap: CurrentBurden=|Following union Learned| may exceed 30; only BurdenCap tops out at 30 and excess invokes Misfire rather than preventing learning. Valid Skill IDs alone bound finite types. Count each Skill once in Following union Learned (including Following Lv0); there is no need to abandon a skill before learning #31. Manual abandonment ONLY at Settlement clears true level to0, Follow, pending/committed Learning, fading state and learned-Spell hotkeys/readied Spell; preserve Scroll/Charges hotkeys. Burden falls. Spent Skill Points NEVER refund, reacquisition starts afresh.

**KN-08** `FROZEN` Automatic forgetting: Learned skills with no protection and no use for an extended real-time interval go Known → Fading → Forgotten; fully forgotten Knowledge/progress is lost and reduces CurrentBurden (there is no hard skill slot); Followed Lv0 candidates must have a defined cleanup policy (`OPEN`). `Slot it, preserve it`: using existing DevilutionX Spell Hotkey assignments protects Learned skills **only from automatic forgetting**, not from the player's deliberate Settlement abandonment. Hotkeys are not additional knowledge instances. On committed abandonment, remove every **memorized `SpellType::Spell` hotkey** for that skill and any readied **learned-Spell** reference, but keep the same `SpellID`'s `SpellType::Scroll` and `SpellType::Charges` hotkeys when their items remain usable; prevent a stale Native Save from restoring permanently learned spell state. A Scroll/Charges shortcut does not protect learned Knowledge against automatic forgetting. Reassign/removing the last protected slot resumes inactivity policy. Skill use refreshes recent-use timestamp; it does not itself generate learning points (those come from eligible kills/Skill Points). Missing time thresholds `OPEN`.

**KN-09** `FROZEN` Settlement abandonment targets only existing Following/Learned Skills; reject overlap with same Settlement's Follow/Skill Point allocations. Permanently erase Knowledge/progress, but never refund committed Skill Points. Abandonment lowers Burden; Follow and Book acquisition DO NOT have a hard 30-Skill cap. No resurrection through Native Save or retry; Book after abandonment may directly grant Lv1.

## 4. Burden, Misfire, Momentum
**COM-01** `FROZEN` No 30-distinct-Skill hard cap: CurrentBurden=|Following union Learned| may exceed 30; only BurdenCap tops out at 30 and excess invokes Misfire rather than preventing learning. BurdenCap starts 10 and can grow to max30 via investing all level-up points in native MAG (not equipment MAG); exact growth curve OPEN-01. CurrentBurden may exceed 30 if native valid Skills allow it.

**COM-02** `FROZEN` `overflow = max(0, burden - cap)`; `misfireProbability = min(0.50, overflow / cap)` (cap >=1). When Burden <= Cap → 0; at Cap10/Burden11 → 10%; 12 → 20%; 15+ → capped **50%**. On Misfire, the normal attempted spell Mana cost is consumed, the cast does not produce its intended skill effect, and original timing/action rules remain. Do not add new cooldown penalties, extra damage penalties or a zero-Mana exploit. Validate applicability to native Book-learned spells; Scroll/Staff charge behavior is `OPEN`.

**COM-03** `FROZEN` One global, character-level **Momentum** (not `Flow`, not a group). Mana has **zero automatic baseline regeneration**. Valid continued combat builds Momentum; Momentum gives configurable Mana regeneration rate `regenPerSecond = f(momentum)`, without modifying native attack speed, original spell timing or cooldown. Outside combat Momentum decays by configurable schedule; at successful Battle Settlement Momentum immediately =0. Native potions/other explicit Mana restoring effects remain valid. Trigger thresholds/curve/max are `OPEN` and must not be invented.

**COM-04** `FROZEN` Use original DevilutionX action timing, hit resolution, elemental resistances, items and original spell hotkeys. New Misfire hook must occur after ability legality checks and before release of effects; ensure exactly one Mana charge and no projectile/area/damage created when it misfires. Observe/log source engine behavior if Mana is consumed at a different point.

## 5. Battle Settlement / Native Save
**SAVE-01** `FROZEN` Battle Settlement commits ONLY Pending Learning, Follow choices, Skill Point spends and voluntary abandonment as one atomic/idempotent revisioned transaction. It never commits gear/stats/Loot Load. Accepted Settlement cannot be canceled, status recoverable on retry; reset Momentum on success. Recalculate Burden but NEVER validate a hard 30-Skill slot count. Native Save must not commit combat Pending or restore abandoned Skills.

**SAVE-02** `FROZEN` Normal Death: **retain vanilla DevilutionX death/drop behavior unchanged**, including its conditional equipped-item drops, half-gold drops, held-item handling, PvP Ear, and original no-drop exceptions; additionally clear all not-yet-committed battle Knowledge Pending. No new XP penalty invented beyond stock. If a Settlement commit was already accepted, server transaction result is definitive; reconnect resolves status by settlementId, not client's optimistic local outcome.

**SAVE-03** `FROZEN` Native Save / critical state changes synchronize committed Character attributes, level/XP, inventory/equipment/stash, item ownership with central server. Server is authoritative for the **committed copy and identity/version**, though it cannot prove live combat. Use original save hook(s) plus event-triggered sync (pickup, item transfer, equip, character stats, logout). Avoid periodically persisting Pending Knowledge through vanilla serialized Spell Level fields: separate or mask knowledge while making native persistence compatible.

**SAVE-04** `FROZEN` Every mutation has stable idempotency key; compare Character revision; reject stale/invalid values; preserve an isolated recovery state for interrupted settlement and interrupted equipment mutations; never duplicate an item or consume Loot Load twice on retry. High-value asset operations cannot claim success if central ledger is unreachable.

## 6. Loot and Item Registry
**LOOT-01** `FROZEN` Original Computer loot generation, item qualities Normal/Magic/Unique, affixes, durability, equipment requirements, 2D inventory and free pickup retained. No Personal Loot or trade UI. Other players can pick up a player-dropped item; ordinary free-pickup race is an accepted V1 property.

**LOOT-02** `FROZEN` Daily Character-scoped, rarity-weighted **Loot Load**, counting **direct pickups of Computer-origin ground items**. Initial balancing values: Normal 1; Magic 3; Unique 10; daily limit 200. Server UTC 00:00 reset, pickup belongs to UTC period at occurrence. On pickup: reserve/deduct Load and register/transfer item **as one idempotent server mutation**. If insufficient, deny pickup (do not hide/erase item for others). Dropping, destroying or donating never refunds Load; a player-dropped item acquired by someone else consumes zero additional direct monster Loot Load. Multiple accounts/characters are not prevented; each Character has separate limit. Learning/combat continue at quota limit.

**LOOT-03** `FROZEN` Official Item Registry assigns global `itemId`, stores origin/type/creation seed or parameters, registration revision, mode, current owner/location, state. No item in two places at once. Server checks generated affix legality, level/requirements where verifiable, and single ownership; this does not prove that non-authoritative combat actually produced the claimed drop. Use direct item event sync, not Knowledge Settlement, for changes.

**LOOT-04** `OPEN` Peer ground pickup ordering under latency and transient central server failure; choose fail-closed for duplicate/over-quota item acquisition, with explicit retry UI. Global inventory replacement/rollback must not create client-visible clones.

## 7. Hardcore, Death Pact, PvP
**HC-01** `FROZEN` Normal/Hardcore combat rooms and loot economies separated; no mixed PvE or PvP. PvP damage allowed only in designated original/extended PvP areas. HC PvP death permanent. Character cannot be revived after server-committed `DEAD` state.

**HC-02** `FROZEN` Hardcore death **bypasses all vanilla death item/gold/ear drops**; do not emit the normal all-equipped-items drop or carried-item/gold/Ear drop on Hardcore death. Hardcore death event automatically reported to central server. Server locks character, uses most recently accepted equipped-item snapshot, randomly chooses approximately **65% of the actually equipped item count** (not 65% of slots) as legacy drops, irreversibly destroys remainder and inventory/stash per final death rule, commits exactly one death/asset result and returns selected item IDs. Use **stochastic rounding** of `0.65 × equippedCount` to preserve expected fraction; sample without replacement. No client-chosen dropped equipment.

**HC-03** `FROZEN` Only other Hardcore players already present in the same room **at the instant of death** are eligible to pick death drops. Ground loot is created after server reply and carries original owner `Name#Tag`, death level/time; on subsequent owner death it keeps original Legacy provenance. Solo death yields no collectible items. When host leaves/room ends, all unclaimed death drops disappear permanently; no future stash/retrieval. Late joiners not eligible.

**HC-04** `FROZEN` **Death Pact**: HC-only special Potion; usable once in lifetime; must FIRST successfully register activation with central server (durable non-reversible `PACT_ACTIVE` and server deadline), only THEN apply temporary large STR/MAG/DEX/VIT buffs + Mana Recovery. Healing disabled while pact active; continuous inevitable Life loss until death, even after disconnect; server deadline guarantees eventually `DEAD` even if client does not send a later combat death event. Must not be reversible by healing, Mana Shield, alt-F4, relog, or changing equipment. Exact buff/drain/deadline parameters `OPEN`.

**HC-05** `FROZEN` HC deaths via pact only enter permanent Hall of Fame, sorted death level DESC, time of death ASC. Death lock and payout/drop transactions are idempotent. Death reports from non-authoritative clients can be suppressed or falsified; V1 integrity checkpoints do NOT solve this completely. Mark risk explicitly.

## 8. Multiplayer / online / anti-cheat
**NET-01** `FROZEN` 1–4 player sessions. Account service handles login/auth; Character, friends, online status, public/friend/private room listings, invites, chat. Support join-in-progress using original DevilutionX synchronization after auditing actual transport/state conditions. V1 retains original TCP/ZeroTier as applicable; the room's creator is a lifecycle owner, NOT an authoritative combat server. No host migration. On room owner termination server closes room and invalidates unclaimed HC death loot, even if a peer transport could otherwise continue; define client termination handshake.

**SEC-01** `FROZEN` Lightweight anti-cheat: signed/hashed approved build manifest covering executable, critical shared libraries and rule/config files at login, entry into combat areas, settlement and reconnect; client reports version and file hashes. Central server checks legit Character attributes against available allocation and known affix generation constraints, known item IDs, ownership and revisions, loot limits, knowledge claim bounds. Do not claim file hashes prove runtime process integrity or real kills. No kernel anti-cheat or full event-by-event authoritative simulation.

**SEC-02** `FROZEN` Use authenticated TLS for central services, native transport appropriate to pinned engine. Reject cross-account Character spoofing, old revisions, duplicate mutation IDs, HC/Normal cross-room join, invalid Skill IDs and forged Item Instance ownership. Record audit reason codes with rate limits; do not auto-ban from client-hash alone.

## 9. V1 acceptance flows (minimum)
1. Fresh account creates neutral classless Character; valid internal HeroClass solely controls graphics/engine/save, NOT gameplay; every Character gets same stats/growth/weapon/Skills/abilities.
2. A Computer executes mapped Skill once, twice or 100 times then dies: each eligible pre-Following Character gets exactly Learning +1, non-Following Character sees candidate only. Unused skills grant 0; extra missiles and repeat death messages do not add credit.
3. Follow at Settlement is not retroactive: next eligible kill grants Learning +1. Abandonment at Settlement erases Level/progress/Following and learned-Spell hotkey, reduces Burden, never refunds spent points; a new Skill can always be acquired even at Burden>=30.
4. Pending Knowledge lost on Normal Death; native equipment save remains consistent. Settlement cannot cancel; retry with same ID changes progress once; Momentum resets only on success.
5. Burden Cap10: Burden10 0%; 11 10%; 12 20%; 15 and above 50% maximum; Misfire consumes Mana and produces no skill effect.
6. Starting Momentum 0 yields no passive Mana regen; sustained valid combat causes recovery; Settlement resets to 0. Regular Mana Potion still works.
7. Spell Book consumes one and grants true Spell Level+1 on valid reading up to native MaxSpellLevel; no wasted already-maxed same-Spell book. Other/new Skills can be acquired even at Burden 30+, becoming Over-Burden and potentially Misfiring. No hard 30-Skill limit; native learned Spell hotkeys protect only from automatic forgetting.
8. Free loot pickups use Character-specific quota with UTC 00:00 rollover; simultaneous requests cannot overspend or duplicate item IDs; manually dropped items do not charge receiver's Loot Load.
9. Hardcore death auto-reports; server selects ~65% of actually equipped assets, returns result, locks dead Character; only co-present HC peers can pick; unclaimed drop vanishes if host quits; duplicate death reports not duplicate gear.
10. Death Pact cannot activate offline without server acknowledgement; after registered, forced death even on disconnection; Hall of Fame entry once for Death Pact death only.

## 10. Explicitly deferred and accepted limits
- No **outside-Settlement** manual forgetting, no custom 10-slot hotbar, no custom CD/attack timing, no CD groups, no Flow/Momentum groups, no new resists/gear slot systems, no rifts, no browser/client 3D, no replay, no economy/trading system, no server combat authority, no aggressive anti-cheat.
- Accepted limitations: non-authoritative combat can be falsified; in-room racing for free loot is possible; file integrity checks detect simple tampering, not memory patching; no host migration; room loss causes unclaimed HC death drops to disappear; uncommitted Knowledge is at risk.
- See `OPEN_DECISIONS.md` for implementation constants and edge cases that must NOT be guessed.
