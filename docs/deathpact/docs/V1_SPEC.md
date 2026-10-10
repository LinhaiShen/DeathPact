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

**CHAR-02** `FROZEN` No class selection or class-restricted skill requirements. Preserve 4 native attributes **STR/MAG/DEX/VIT** and vanilla formulas/other item and attribute requirements where applicable. Audit class-dependent native loadout, stat caps, spells, character animations and creation: replace only what classlessness requires. Retain vanilla level limit until specifically changed; do NOT assume previous level 99 is V1.

**CHAR-03** `OPEN` neutral starting attributes/loadout/appearance/initial skill and numeric leveling rewards for classless characters, and release-specific level cap.

## 3. Knowledge/SkillUsedEvent
**KN-01** `FROZEN` Computer is the name for non-player enemies. DO NOT preconfigure a per-Computer teachable skill list. Preserve original Computer attack/spell functions. At the boundary when an action actually executes, emit a single canonical `SkillUsedEvent` mapped through a table keyed by verified upstream action/spell identifier and context.

Minimal conceptual event (not yet an upstream C++ type):
```
SkillUsedEvent { roomSessionId, computerSpawnId, castId,
                 engineActionKey, skillId, bearerSkillLevel,
                 occurredTick, schemaVersion }
```
- `castId` uniquely identifies an executed action (not projectile, target hit, animation subframe, or damage callback). One valid action → one event even for multi-hit/AoE/multiple targets. Mapping must not alter engine combat.
- Only mapped, actually executed and permitted actions count as Knowledge. Unmapped actions are ignored and explicitly listed in audit/report. No fabricated Skill ID/Level.
- The mapping table maps **Computer original Action/Spell ID → canonical Learnable Skill ID + deterministic Skill Level rule**. Rules for vanilla actions without level data must be declared, not guessed.

**KN-02** `FROZEN` A Computer must die for its actual observed-used skills to become Pending Knowledge. At most ONE learning credit per `(roomSessionId, computerSpawnId, canonicalSkillId)` on that kill, regardless of number of casts. Unique spawn identity must survive network replication and not collide on respawn.

**KN-03** `FROZEN` Eligible party members all receive the same Computer-derived Knowledge candidate/credit; no last-hit ownership, no damage-contribution requirement, no proximity bonus, provided they are active members of that room's party at the kill event. Each Character maintains their own Follow and progress. Kills are not historical retroactive credits for players who join later. On a new Computer kill new members may qualify.

**KN-04** `FROZEN` If skill not Followed: appear as Follow candidate at Settlement (only if actually used, and Computer died); opting into Follow does NOT retroactively give a progress point from the same past kill. If Followed before the kill: add one level-weighted credit for that unique `(Computer, Skill)` kill. Credits stay **Pending** until Settlement. New Follow selection is itself committed by Settlement. A Follow action does not retroactively process past kills.

**KN-05** `FROZEN` Higher bearer Skill Level gives stronger progress according to a configurable function. Low/equal bearer level cannot raise character's true Knowledge above the holder; the precise point curve is `OPEN`. `Skill Points` gained from original/extended progression may be spent only on Followed skills, not to create undiscovered Skill IDs. Skill-point allocations applied to Knowledge are Pending and submitted only at Settlement; abort/death cannot consume permanent points without corresponding committed progress.

**KN-06** `FROZEN` Vanilla Spell Book: if true Knowledge Lv <1, read grants **Lv1 directly**; if already Lv1+, cannot increase level. Spell Book does not automatically Follow. Preserve native Scroll/Staff Charges as temporary casts: no permanent learned skill on mere use. Validate Spell Book changes as a separate, immediate item-use/character-save mutation (`BOOK_LEARN_LV1`), **not as un-settled combat-derived progress**; never allow original book logic to raise Lv2+. The exact handling of an ineffective book and 30-skill-full book is `OPEN`, see `OPEN_DECISIONS.md`.

**KN-07** `FROZEN` At most **30 distinct** registered skills per Character, counting Following Lv0 and all Learned Lv1+ skills (union; no double count). A player may **voluntarily abandon** a Following or Learned skill **only during Battle Settlement**. On successful commit, that skill loses its Follow flag, its true Learned Level (becomes Lv0), **all stored and pending Learning Progress**, fading/protection state and associated Spell Hotkey assignments; it ceases to count toward Burden and the 30-skill limit. It cannot regain historical progress or level through replay/reconnect/Native Save. Later reacquisition starts from **Lv0 with zero progress**, through the normal discovery/Follow path or a new legal Spell Book use (which may directly grant Lv1). Abandonment cannot be undone after Settlement commits. Abandoned skill IDs cannot also be selected for Follow or Skill Point allocation within the same Settlement. No dedicated outside-Settlement manual forget action; automatic forgetting continues separately. Distinguish temporary scroll/staff spells from owned Knowledge.

**KN-08** `FROZEN` Automatic forgetting: Learned skills with no protection and no use for an extended real-time interval go Known → Fading → Forgotten; fully forgotten knowledge/progress is lost and frees burden/skill slot; Followed Lv0 candidates must have a defined cleanup policy (`OPEN`). `Slot it, preserve it`: using existing DevilutionX Spell Hotkey assignments protects Learned skills **only from automatic forgetting**, not from the player's deliberate Settlement abandonment. Hotkeys are not additional knowledge instances. On committed abandonment, remove every hotkey assignment referencing that skill and prevent a stale Native Save from restoring it. Reassign/removing the last protected slot resumes inactivity policy. Skill use refreshes recent-use timestamp; it does not itself generate learning points (those come from eligible kills/Skill Points). Missing time thresholds `OPEN`.

**KN-09** `FROZEN` At Settlement, `abandonSkillIds` denotes **existing** Following/Learned skills, not newly discovered candidate IDs. Reject any overlap with `followChoices` or `pointAllocations` in the same request. Abandoning a skill discards that skill's outstanding per-kill pending credits as well as persisted level/progress; do not award or resurrect it on the same Settlement. Lost progress remains lost after any later acquisition. Previously committed Skill Points spent on an abandoned skill are **never refunded**. The historical spent-point ledger remains unchanged; abandoning, re-learning, retrying Settlement or applying Native Save must not increase unspent Skill Points. Any pending allocation to an abandoned skill within the same Settlement is rejected. Book use after abandonment is a separate legal new acquisition, not restoration of the former skill. Test 30→29→30 capacity changes and atomic Settlement retry.

## 4. Burden, Misfire, Momentum
**COM-01** `FROZEN` `CurrentBurden = |Following ∪ Learned|`, each distinct Skill costs one, even if Followed and Learned. Default `BurdenCap=10`; ultimate Cap=30 achievable at V1 maximum level by allocating ALL available level-up distributable points to native **MAG**. Use raw *allocated* MAG points, not temporary +MAG from equipment, for the cap budget. Detailed monotone curve pending configuration; tests must prove endpoints.

**COM-02** `FROZEN` `overflow = max(0, burden - cap)`; `misfireProbability = min(0.50, overflow / cap)` (cap >=1). When Burden <= Cap → 0; at Cap10/Burden11 → 10%; 12 → 20%; 15+ → capped **50%**. On Misfire, the normal attempted spell Mana cost is consumed, the cast does not produce its intended skill effect, and original timing/action rules remain. Do not add new cooldown penalties, extra damage penalties or a zero-Mana exploit. Validate applicability to native Book-learned spells; Scroll/Staff charge behavior is `OPEN`.

**COM-03** `FROZEN` One global, character-level **Momentum** (not `Flow`, not a group). Mana has **zero automatic baseline regeneration**. Valid continued combat builds Momentum; Momentum gives configurable Mana regeneration rate `regenPerSecond = f(momentum)`, without modifying native attack speed, original spell timing or cooldown. Outside combat Momentum decays by configurable schedule; at successful Battle Settlement Momentum immediately =0. Native potions/other explicit Mana restoring effects remain valid. Trigger thresholds/curve/max are `OPEN` and must not be invented.

**COM-04** `FROZEN` Use original DevilutionX action timing, hit resolution, elemental resistances, items and original spell hotkeys. New Misfire hook must occur after ability legality checks and before release of effects; ensure exactly one Mana charge and no projectile/area/damage created when it misfires. Observe/log source engine behavior if Mana is consumed at a different point.

## 5. Battle Settlement / Native Save
**SAVE-01** `FROZEN` **Battle Settlement commits Knowledge only**, including combat-acquired pending progress, Skill Point investments, Follow choices and **skill-abandonment choices**. It never commits equipment, attributes or Loot Load. Player chooses any skill(s) to abandon **inside the Settlement UI before submission**; an accepted Settlement cannot itself be cancelled/discarded. Skill abandonment is **not** a cancellation of Settlement. The whole request (growth, abandonment, hotkey cleanup, Knowledge/Burden-capacity validation) must commit atomically, with a stable settlement ID and idempotent retries. On success reset Momentum; before success retain progress with defined recoverability. Do not use auto-save to commit Pending Knowledge or revive abandoned Knowledge.

**SAVE-02** `FROZEN` Normal Death: original DevilutionX normal death outcomes still apply; additionally clear all not-yet-committed battle Knowledge Pending. No new XP penalty invented beyond stock. If a Settlement commit was already accepted, server transaction result is definitive; reconnect resolves status by settlementId, not client's optimistic local outcome.

**SAVE-03** `FROZEN` Native Save / critical state changes synchronize committed Character attributes, level/XP, inventory/equipment/stash, item ownership with central server. Server is authoritative for the **committed copy and identity/version**, though it cannot prove live combat. Use original save hook(s) plus event-triggered sync (pickup, item transfer, equip, character stats, logout). Avoid periodically persisting Pending Knowledge through vanilla serialized Spell Level fields: separate or mask knowledge while making native persistence compatible.

**SAVE-04** `FROZEN` Every mutation has stable idempotency key; compare Character revision; reject stale/invalid values; preserve an isolated recovery state for interrupted settlement and interrupted equipment mutations; never duplicate an item or consume Loot Load twice on retry. High-value asset operations cannot claim success if central ledger is unreachable.

## 6. Loot and Item Registry
**LOOT-01** `FROZEN` Original Computer loot generation, item qualities Normal/Magic/Unique, affixes, durability, equipment requirements, 2D inventory and free pickup retained. No Personal Loot or trade UI. Other players can pick up a player-dropped item; ordinary free-pickup race is an accepted V1 property.

**LOOT-02** `FROZEN` Daily Character-scoped, rarity-weighted **Loot Load**, counting **direct pickups of Computer-origin ground items**. Initial balancing values: Normal 1; Magic 3; Unique 10; daily limit 200. Server UTC 00:00 reset, pickup belongs to UTC period at occurrence. On pickup: reserve/deduct Load and register/transfer item **as one idempotent server mutation**. If insufficient, deny pickup (do not hide/erase item for others). Dropping, destroying or donating never refunds Load; a player-dropped item acquired by someone else consumes zero additional direct monster Loot Load. Multiple accounts/characters are not prevented; each Character has separate limit. Learning/combat continue at quota limit.

**LOOT-03** `FROZEN` Official Item Registry assigns global `itemId`, stores origin/type/creation seed or parameters, registration revision, mode, current owner/location, state. No item in two places at once. Server checks generated affix legality, level/requirements where verifiable, and single ownership; this does not prove that non-authoritative combat actually produced the claimed drop. Use direct item event sync, not Knowledge Settlement, for changes.

**LOOT-04** `OPEN` Peer ground pickup ordering under latency and transient central server failure; choose fail-closed for duplicate/over-quota item acquisition, with explicit retry UI. Global inventory replacement/rollback must not create client-visible clones.

## 7. Hardcore, Death Pact, PvP
**HC-01** `FROZEN` Normal/Hardcore combat rooms and loot economies separated; no mixed PvE or PvP. PvP damage allowed only in designated original/extended PvP areas. HC PvP death permanent. Character cannot be revived after server-committed `DEAD` state.

**HC-02** `FROZEN` Hardcore death event automatically reported to central server. Server locks character, uses most recently accepted equipped-item snapshot, randomly chooses approximately **65% of the actually equipped item count** (not 65% of slots) as legacy drops, irreversibly destroys remainder and inventory/stash per final death rule, commits exactly one death/asset result and returns selected item IDs. Use **stochastic rounding** of `0.65 × equippedCount` to preserve expected fraction; sample without replacement. No client-chosen dropped equipment.

**HC-03** `FROZEN` Only other Hardcore players already present in the same room **at the instant of death** are eligible to pick death drops. Ground loot is created after server reply and carries original owner `Name#Tag`, death level/time; on subsequent owner death it keeps original Legacy provenance. Solo death yields no collectible items. When host leaves/room ends, all unclaimed death drops disappear permanently; no future stash/retrieval. Late joiners not eligible.

**HC-04** `FROZEN` **Death Pact**: HC-only special Potion; usable once in lifetime; must FIRST successfully register activation with central server (durable non-reversible `PACT_ACTIVE` and server deadline), only THEN apply temporary large STR/MAG/DEX/VIT buffs + Mana Recovery. Healing disabled while pact active; continuous inevitable Life loss until death, even after disconnect; server deadline guarantees eventually `DEAD` even if client does not send a later combat death event. Must not be reversible by healing, Mana Shield, alt-F4, relog, or changing equipment. Exact buff/drain/deadline parameters `OPEN`.

**HC-05** `FROZEN` HC deaths via pact only enter permanent Hall of Fame, sorted death level DESC, time of death ASC. Death lock and payout/drop transactions are idempotent. Death reports from non-authoritative clients can be suppressed or falsified; V1 integrity checkpoints do NOT solve this completely. Mark risk explicitly.

## 8. Multiplayer / online / anti-cheat
**NET-01** `FROZEN` 1–4 player sessions. Account service handles login/auth; Character, friends, online status, public/friend/private room listings, invites, chat. Support join-in-progress using original DevilutionX synchronization after auditing actual transport/state conditions. V1 retains original TCP/ZeroTier as applicable; the room's creator is a lifecycle owner, NOT an authoritative combat server. No host migration. On room owner termination server closes room and invalidates unclaimed HC death loot, even if a peer transport could otherwise continue; define client termination handshake.

**SEC-01** `FROZEN` Lightweight anti-cheat: signed/hashed approved build manifest covering executable, critical shared libraries and rule/config files at login, entry into combat areas, settlement and reconnect; client reports version and file hashes. Central server checks legit Character attributes against available allocation and known affix generation constraints, known item IDs, ownership and revisions, loot limits, knowledge claim bounds. Do not claim file hashes prove runtime process integrity or real kills. No kernel anti-cheat or full event-by-event authoritative simulation.

**SEC-02** `FROZEN` Use authenticated TLS for central services, native transport appropriate to pinned engine. Reject cross-account Character spoofing, old revisions, duplicate mutation IDs, HC/Normal cross-room join, invalid Skill IDs and forged Item Instance ownership. Record audit reason codes with rate limits; do not auto-ban from client-hash alone.

## 9. V1 acceptance flows (minimum)
1. Fresh account creates neutral Classless Character; 4 native stats visible; login restores last committed save.
2. In 2-player room Computer actually casts mapped spell twice; it dies; both players have ONE kill-based credit/candidate for that `(Computer, skill)`; not two. Unused mapped skills earn zero.
3. Unfollowed candidate can be Followed at Settlement, but cannot retroactively gain that kill's progress. Followed skill receives level-weighted pending progress; skill-point allocation persists only after Settlement. User may mark an existing Followed/Learned skill for abandonment at Settlement; after commit it is Lv0, 0 progress, not Following, not hotkey-assigned, and no longer counted toward Burden or the 30-skill limit. Reacquisition never restores the old level/progress.
4. Pending Knowledge lost on Normal Death; native equipment save remains consistent. Settlement cannot cancel; retry with same ID changes progress once; Momentum resets only on success.
5. Burden Cap10: Burden10 0%; 11 10%; 12 20%; 15 and above 50% maximum; Misfire consumes Mana and produces no skill effect.
6. Starting Momentum 0 yields no passive Mana regen; sustained valid combat causes recovery; Settlement resets to 0. Regular Mana Potion still works.
7. Spell Book on unknown skill grants Lv1; on Lv1+ cannot grant Lv2. Up to 30 owned/followed skill types; voluntary abandonment is allowed **only in Settlement**, while un-slotted inactive Knowledge is automatically forgotten; hotkeys protect only against the automatic path. If the 30-slot cap is full, abandoning one skill in the same Settlement permits a new legal Follow choice after atomic capacity validation.
8. Free loot pickups use Character-specific quota with UTC 00:00 rollover; simultaneous requests cannot overspend or duplicate item IDs; manually dropped items do not charge receiver's Loot Load.
9. Hardcore death auto-reports; server selects ~65% of actually equipped assets, returns result, locks dead Character; only co-present HC peers can pick; unclaimed drop vanishes if host quits; duplicate death reports not duplicate gear.
10. Death Pact cannot activate offline without server acknowledgement; after registered, forced death even on disconnection; Hall of Fame entry once for Death Pact death only.

## 10. Explicitly deferred and accepted limits
- No **outside-Settlement** manual forgetting, no custom 10-slot hotbar, no custom CD/attack timing, no CD groups, no Flow/Momentum groups, no new resists/gear slot systems, no rifts, no browser/client 3D, no replay, no economy/trading system, no server combat authority, no aggressive anti-cheat.
- Accepted limitations: non-authoritative combat can be falsified; in-room racing for free loot is possible; file integrity checks detect simple tampering, not memory patching; no host migration; room loss causes unclaimed HC death drops to disappear; uncommitted Knowledge is at risk.
- See `OPEN_DECISIONS.md` for implementation constants and edge cases that must NOT be guessed.
