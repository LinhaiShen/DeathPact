# Central service contract — V1 implementation proposal

This is a **proposed** transport/domain contract; route names are not existing DevilutionX APIs and may be adapted to selected server framework. Prefer a single backend app + PostgreSQL, but do not claim the server technology has been chosen. All authenticated operations use Character membership, build/ruleset compatibility, unique `requestId`, compare-and-swap `expectedRevision`, and return server-assigned revision/result. Transactional consistency matters more than endpoint naming.

## 1. Authority and durability matrix

| Domain | Live handler | Committed source | Write boundary |
|---|---|---|---|
| Movement/monster/combat | DevilutionX peers | no combat server authority | original networking |
| Account, Character identity, lobby, mode | Central | DB | authenticated calls |
| Stats, level, XP, inventory, equipment, stash | Original client during play | Central committed save+ledger | native Save + high-value mutation events |
| Spell Book level result (currently Lv1-only, OPEN-09 proposal to use native +1 per Book) | Original item-use + sync adapter | DB Knowledge and item state | independent immediate atomic Book-consumption + level transaction; no combat Settlement |
| Kill-based skill discovery/progress and allocated Skill Points | Local Pending | DB committed Knowledge | **Battle Settlement only** |
| Following choices and deliberate skill abandonment | Settlement UI/Pending | DB committed Follow / Knowledge | **Battle Settlement only** |
| Momentum / combat-time Mana recovery | local live engine | no independent DB requirement | reset on Settlement; no mutation authority |
| Monster-origin Loot Load | Client request and central quota check | DB UTC character ledger | pickup reservation/commit |
| Item registry and ownership | client sync after action | DB registry | atomic, versioned movement |
| Hardcore death / Pact | local report/potion attempt | DB irrevocable flags | server death/Pact transaction |

Do not allow a native save containing changed spell fields to silently overwrite committed Knowledge, **restore voluntarily abandoned skills**, or reapply cleared hotkey assignments. Reconstruct the playable spell state by merging `committedKnowledge` + **volatile pending progress used only for UI**, then apply vanilla effective spell/item buffs as required. Server recognizes no skill level increase from local automatic saves.

## 2. Suggested entities and invariants

```
Account(id, emailHashOrAddress, authFields, createdAt)
Character(id, accountId, name, publicTag, mode[Normal|Hardcore], status[Alive|PactActive|Dead],
          level, xp, str, mag, dex, vit, allocatedMagPoints,
          saveRevision, knowledgeRevision, equipmentSnapshotRevision, roomId, buildVersion)
KnowledgeSkill(characterId, skillId, learnedLevel, following, learningPoints,
               lastUsedAtUtc, lastSlotSeenAtUtc, isFading, updatedAt)
SpellHotkey(characterId, hotkeyIndex, spellId, spellType[Spell|Skill|Scroll|Charges])
SkillPointLedger(characterId, awarded, committedSpent)
Room(id, creatorAccountId, mode, visibility, status, lastHeartbeatAt)
RoomMember(roomId, characterId, joinedAt, leftAt, role)
PendingSession(characterId, roomSessionId, ephemeralClaimDigest, localState)
Settlement(id, characterId, roomSessionId, requestId, preKnowledgeRevision,
           state[Received|Committed|Rejected|Recovering], requestDigest, resultDigest, createdAt)
ItemRegistry(itemId, itemType, generationParameters, mode, originType, originContext,
             itemState, currentCharacterId, roomId, registeredAt, revision, legacyOwnerTag,
             legacyDeathLevel, legacyDeathTime)
ItemOperation(requestId, itemId, sourceState, destState, preRevision, result, occurredAt)
LootUsage(characterId, utcDate, used)
LootOperation(requestId, characterId, itemId, originType, rarity, cost, utcDate,
              state[Reserved|Committed|Denied], result)
HardcoreDeath(deathId, characterId, roomId, deathSnapshotRevision,
              aliveToDeadRevision, equippedCount, dropItemIds, destroyedItemIds,
              allowedRecipientCharacterIds, deathAtUtc, cause, outcomeHash)
PactActivation(characterId UNIQUE, requestId, activatedAtUtc, inevitableDeathAtUtc, outcome)
AuditEvent(id, actionType, subjectId, reasonCode, observedAtUtc, sanitizedMetadata)
```

Use relational foreign keys, unique `(characterId, skillId)` and unique mutation IDs; one registered item can have exactly one active placement/owner. The game may need more vanilla fields than shown: keep versioned opaque, validated native-save payload **in addition to normalized key fields**, not instead of original data.

`Committed Character` (and equipment snapshot) is authoritative when logging in; a local cache is never a new source of truth.

## 3. Proposed boundary commands (conceptual JSON)

### 3.1 Native character save
```json
{
  "type": "CHARACTER_SAVE",
  "requestId": "uuid",
  "characterId": "character-id",
  "expectedRevision": 12,
  "buildVersion": "approved-build",
  "rulesetVersion": "v1-data",
  "nativeSaveSchema": 1,
  "snapshot": {"level": 17, "xp": 123456, "str": 40, "mag": 65, "dex": 35, "vit": 30,
                "equipmentItemIds": ["item-a"], "inventoryItemIds": ["item-b"],
                "stashItemIds": []}
}
```
Server recomputes legal attribute-point budget and derived equipment stats (rather than trusting a displayed final STR), validates item IDs/positions/attributes, prevents state rollback. Reject or queue a conflict for reconciliation; never simply last-write-wins.

### 3.2 Knowledge settlement
```json
{
  "type": "KNOWLEDGE_SETTLE",
  "requestId": "uuid",
  "settlementId": "uuid",
  "characterId": "character-id",
  "roomSessionId": "session-id",
  "expectedKnowledgeRevision": 9,
  "claims": [
    {"computerSpawnId": "monster-instance-a", "skillId": "firebolt", "bearerSkillLevel": 3,
     "killEventId": "kill-101"}
  ],
  "followChoices": ["firebolt"],
  "abandonSkillIds": ["old-skill-id"],
  "pointAllocations": [{"skillId": "firebolt", "points": 2}]
}
```
Required validation: signed-in Character, correct room/mode and known session; dedup `(roomSessionId, computerSpawnId, skillId)`; skill mapping exists; sane levels and bounds; current Follow/skill capacity; available unspent Skill Points; expected revision and duplicate IDs. `abandonSkillIds` must be distinct IDs already Following or Learned, and cannot overlap with `followChoices` or `pointAllocations`. **In the same atomic Knowledge transaction**, delete abandoned skills' Follow/level/committed and pending progress, clear Fading state and memorized learned-Spell (`spellType=Spell`) Hotkey references / active readied learned Spell only (preserve same-spell Scroll/Charges hotkeys), and remove them from Burden/30-cap counting; discard any associated credit claims from that Settlement. Recalculate the 30-skill limit using the post-abandonment candidate set, so 30→29→30 is valid. Never refund any previously committed Skill Points spent on abandoned skills (`FROZEN`); preserve historical spend ledger and unspent-point balance across Settlement retries, Native Save, and re-learning. On retry, return exactly the previously committed outcome and never resurrect dropped progress/hotkeys. Server **cannot prove kills were real**: explicitly record low trust in audit; optional same-room peer digests add weak cross-checks only.

Server result includes `settlementId`, `CommittedKnowledgeRevision`, committed Follow/level/progress, **abandonedSkillIds, clearedHotkeyIndices**, skill points consumed, and `resetMomentum: true`. Record transaction before acknowledgement. If client crashes after send, `GET_SETTLEMENT_STATUS(settlementId)` returns durable result; never allow cancel after acceptance. If rejected, return reason and deterministic retry policy.

### 3.3 Direct loot pickup
```json
{
  "type": "PICKUP_MONSTER_DROP",
  "requestId": "uuid",
  "characterId": "character-id",
  "roomId": "room-id",
  "groundItemToken": "room-item-token",
  "rarity": "Unique",
  "claimedPickupAtUtc": "2026-10-10T14:22:00Z",
  "generationParameters": {"engineItemId": 123, "seed": 456, "createInfo": 789}
}
```
Server timestamp/UTC period is definitive, **NOT `claimedPickupAtUtc`**. For a late-arriving event at a UTC boundary, use a server-issued event token/timestamp/reservation (otherwise take receipt time); never accept arbitrary backdating by client. Check same ground token is not already assigned, validate generation parameters as far as possible, verify character/day quota, atomically commit ledger cost plus unique global item ID and ownership; reject on insufficient quota. During server outage fail closed for new direct-monster pickups that would mint committed items.

### 3.4 Player drops / pickup of player-dropped item
Use distinct `MOVE_ITEM` requests, with server-tracked `itemId` and source/destination owner/location. Cost=0 for receiving another player's **already registered, voluntarily dropped** item. Native game may show the object before central acknowledgement; do not finalize owner UI until ack. Concurrent claims serialized by item revision and row lock.

### 3.5 Hardcore death
```json
{
  "type": "HARDCORE_DEATH_REPORT",
  "requestId": "uuid",
  "deathId": "stable-once-per-death-uuid",
  "characterId": "hc-character-id",
  "roomId": "hc-room-id",
  "equippedSnapshotRevision": 25,
  "cause": "ComputerOrPvPOrPact"
}
```
Server status moves to irreversible `Dead`, rejects further gameplay/save, and generates one deterministic-persisted result using server RNG: `n=equippedCount`, `exact=0.65*n`, `dropCount=floor(exact)+Bernoulli(frac(exact))`; sample `dropCount` distinct equipped item IDs. Freeze allowed recipients from room membership **at death** (best available server recorded membership); exclude dead character. Remaining equipped items + inventory/stash destroyed. Only after DB commit return result for floor drop spawning and handling; on no recipients, mark selected drops destroyed too. If creator leaves/room ends, unclaimed dropped item IDs become destroyed immediately. Deduplicate by `characterId` / `deathId` — never roll twice.

### 3.6 Death Pact
```json
{"type":"PACT_ACTIVATE","requestId":"uuid","characterId":"hc-character-id", "roomId":"room-id"}
```
Server first persists one-time Pact state + unavoidable server-side death deadline (configured) and returns `PACT_ACTIVATED`; only then client applies potion buff and Life drain. Background server transaction kills PactActive Character at deadline even if no client reports normal combat death. If a natural death report arrives sooner, same death path applies once. Do not allow activation without connectivity.

## 4. Consistency, ordering, errors
- Separate monotone `saveRevision` and `knowledgeRevision`, plus item revisions. Save and Settlement must not overwrite each other's domains. However operations across these domains that share spell book consumption or death must use one transaction/lock or an explicit coordinator; **no accidental partial mutations**.
- Device local file timestamps are not authoritative. UTC daily Loot Load credited via server receipt/issued reservation token. Retry returns stored exact result for the same `requestId`; same ID + changed payload is rejected.
- All item operations use unique one-location invariant and DB locks/serializable claim, not just mutable in-memory JS state.
- On network failure: client clearly shows Pending/Not Committed. No fake success. Death Pact, settlement finalization, registered loot minting and item ownership transfer need server acknowledgements.
- To avoid leaking personal email, use public `CharacterName#Tag` and opaque IDs outside authenticated account admin features.

## 5. Security capability statement
Server can validate **schema, level/stat budget, item generation plausibility, ownership uniqueness, quotas, correct published ruleset, allowed transitions and idempotency**. With player-hosted non-authoritative combat, it **cannot verify** true monster kill, drop provenance, actual Hardcore death, or that an unmodified process ran between hash checkpoints. This is V1 risk acceptance. Do not imply file checksum, multi-peer signatures, deterministic-looking RNG seeds or a local replay are reliable anti-cheat proofs.
