# DeathPact — Phase 0 upstream source audit (partial; native test gates pending)

**Audit date:** 2026-10-11  
**Repository:** `LinhaiShen/DeathPact`, branch `master`  
**Upstream/native engine baseline:** `bf574c9b74f1d718b0bcee15d648b20a8bfc927c` (first parent of DeathPact's specification-import commit `b689c68e9f5edd7c4ad1da900551bc13f9005f41`).  
**Source evidence:** All source links below are pinned to that baseline, not floating `master`.  
**Scope:** Static inspection of native DevilutionX source through the connected GitHub repository. No game behavior, build, CI, end-to-end multiplayer session or native save round-trip was executed in this environment. Do not interpret this as a completed Phase 0 acceptance.

## 0. Completion and provenance

| Gate | Status | Evidence / next step |
| --- | --- | --- |
| Identify original engine commit | **VERIFIED** | Native base: `bf574c9b74f1d718b0bcee15d648b20a8bfc927c`; current fork at audit start: `b689c68e9f5edd7c4ad1da900551bc13f9005f41` |
| Confirm native source is unmodified relative to baseline | **VERIFIED** | Git compare native base → imported-spec commit lists only ten new `docs/deathpact/**` files; no engine file changed |
| Locate monster attack, monster death and player death functions | **VERIFIED (source)** | Sections 1 and 5 |
| Locate item/Book/hotkey/native save/class/multiplayer entry points | **VERIFIED (source)** | Sections 2–8 |
| Build unmodified native game | **NOT RUN** | Must run on developer Windows/WSL or connected CI; see §10 |
| Run native tests | **NOT RUN** | Record command, output and pass/fail with actual environment |
| Test solo and 2/4-player native sessions | **NOT RUN** | Must include host closure, reconnect, combat, item ownership |
| Gameplay/Knowledge/Hardcore implementation | **NOT STARTED** | Phase 0 must not make gameplay changes |
| Map every existing Computer action to a canonical Skill | **NOT COMPLETE** | Needs case-by-case mapping audit, not blanket translation |
| LICENSE and data prerequisites | **VERIFIED (source)** | §9; commercial use is constrained |

## 1. Computer execution events and stable death attribution

- Monster behavior runs from `ProcessMonsters()` and `AiProc[monster.ai]`, with actions progressed in `UpdateModeStance()`. See [`Source/monster.cpp:4257-4331`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L4257-L4331) and [`Source/monster.cpp:3164-3204`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L3164-L3204).
- `StartAttack`, `StartRangedAttack`, `StartRangedSpecialAttack` **schedule attack modes**; this is not proof that an attack's effect has executed. See [`Source/monster.cpp:821-860`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L821-L860).
- `MonsterAttack`, `MonsterRangedAttack`, and `MonsterRangedSpecialAttack` implement actual attack frames and projectiles. Ranged `ChargedBolt` creates **three missiles within one executed action**; never count each missile as its own Learning event. See [`Source/monster.cpp:1270-1360`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L1270-L1360).
- The AI-to-projectile mapping `GetMissileType(MonsterAIID)` maps monster AI kinds to **`MissileID`**, not canonical `SpellID` or skill level. A static monster type or `MonsterAIID` does **not** prove actual skill use. See [`Source/monster.cpp:1940-1977`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L1940-L1977) and [`Source/spells.cpp:208-230`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/spells.cpp#L208-L230).
- `MonsterDeath(Monster&, Direction, bool)` performs experience, loot, state and quest transitions; `M_StartKill`/`M_SyncStartKill` and monster-on-monster paths can enter death handling. The separate `MonsterDeath(Monster&)` is a **death-animation-mode handler**; do not mistake it for the one-time gameplay kill event. See [`Source/monster.cpp:3996-4066`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L3996-L4066), [`Source/monster.cpp:1498-1519`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L1498-L1519), [`Source/monster.cpp:999-1014`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L999-L1014).
- `Monster::getId()` is documented as **an index into the `Monsters` array**, not a cross-room unique identity. Create a separate per-spawn `computerSpawnId` scoped by stable room session / level instance / spawn generation; test ID reuse when loading levels and after respawns. See [`Source/monster.h:417-424`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.h#L417-L424).

**Frozen Knowledge decision:** actual executed mapped Computer Skill marks its ID present in the per-spawn usedSkillIds SET. Upon death eligible pre-Following Character gains precisely Learning +1 per Skill; all cast/projectile/monster skill level counts ignored. Idempotent by room+spawn+Skill.

## 2. Spell usage, Mana and the Misfire hook

- `StartSpell()` checks the legality of queued Skill/Spell/Scroll/Charges and prepares `executedSpell`; `CheckSpell` validates learned-level and Mana for ordinary Spell. See [`Source/player.cpp:248-290`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L248-L290) and [`Source/spells.cpp:182-207`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/spells.cpp#L182-L207).
- `CastSpell()` constructs one or more missiles **before** `ConsumeSpell()`, and consumes only when `fizzled == false`; for `ChargedBolt` this means additional projectiles. See [`Source/spells.cpp:208-231`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/spells.cpp#L208-L231).
- `ConsumeSpell()` behaves differently for `Skill`, `Scroll`, `Charges`, and learned `Spell`; learned Spell subtracts both `_pMana` and `_pManaBase`. It also has certain spell-specific HP consequences, so suppression must define whether such side effects apply on a Misfire instead of accidentally inheriting them. See [`Source/spells.cpp:144-175`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/spells.cpp#L144-L175).
- `Player::GetSpellLevel()` includes equipment bonus `_pISplLvlAdd`; it is **effective spell level**, not the DeathPact's authoritative true Knowledge level. See [`Source/player.h:638-647`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.h#L638-L647).
  
**Design consequence:** a Misfire must charge Mana exactly once *without spawning spell effects*. Simply invoking an early return from `CastSpell()` would skip Vanilla's normal Mana deduction, violating the V1 rule. Check Scroll/Charge/innate Skill applicability against OPEN-08; do not silently apply Misfire to all four sources.

## 3. Books, skill hotkeys and forced forgetting

- Inventory `UseInvItem()` invokes `UseItem()` then removes a used non-scroll item from its inventory or belt slot. See [`Source/inv.cpp:2194-2230`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/inv.cpp#L2194-L2230).
- Native IMISC_BOOK increases _pSplLvl up to MaxSpellLevel. DeathPact preserves consume-one Book / true Level+1 via atomic server item+Knowledge mutation; allow Books for new/other Skills at Burden30+, don't waste already maxed same-Spell Book. See Source/items.cpp near lines 4289-4310.
- The book's Magic requirement changes with the current spell level; avoid leaving the vanilla increase-based requirements accidentally active when introducing V1 Knowledge logic. See [`Source/items.cpp:4828-4844`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/items.cpp#L4828-L4844).
- Hotkeys are represented by paired `_pSplHotKey[]` and `_pSplTHotKey[]`, and distinguish memorized `Spell`, innate `Skill`, `Scroll` and `Charges`. See [`Source/player.h:320-339`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.h#L320-L339) and [`Source/panels/spell_list.cpp:270-333`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/panels/spell_list.cpp#L270-L333).

**Frozen interpretation:** Settlement abandonment removes authoritative skill level/progress and memorized learned-Spell (`SpellType::Spell`) hotkeys/readied learned Spell only. Keep same-`SpellID` Scroll and Staff-Charges hotkeys, because those reference independent consumable/equipment casting sources; they do not protect permanent Knowledge from automatic forgetting. No Skill Points refund.

## 4. Native save is not a Knowledge commit

- Native hero save `pfile_write_hero()` calls `PackPlayer`, `EncodeHero`, `SaveHotkeys` and `SaveHeroItems` in the appropriate mode. See [`Source/pfile.cpp:527-543`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/pfile.cpp#L527-L543) and [`Source/pfile.cpp:145-181`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/pfile.cpp#L145-L181).
- Actual serialization writes `_pSplLvl`, `_pMemSpells`, `_pAblSpells`, hotkey slots; native read restores these fields. See [`Source/loadsave.cpp:445-481`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/loadsave.cpp#L445-L481) and [`Source/loadsave.cpp:1317-1339`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/loadsave.cpp#L1317-L1339). Extra hotkeys are handled separately; `LoadHotkeys`/`SaveHotkeys` exist in [`Source/loadsave.cpp:2333-2390`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/loadsave.cpp#L2333-L2390).
- `pfile_update(forceSave)` autosaves **multiplayer** hero/stash approximately once per minute or on forced flush and is invoked in `GameLogic()`. It explicitly does **not** run for single-player. Other explicit save hooks still exist. See [`Source/pfile.cpp:827-842`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/pfile.cpp#L827-L842) and [`Source/diablo.cpp:1520-1568`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/diablo.cpp#L1520-L1568).
- `InitSingle()` and `InitMulti()` take distinct paths but both load locally saved hero data. See [`Source/multi.cpp:519-573`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/multi.cpp#L519-L573) and [`Source/pfile.cpp:785-805`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/pfile.cpp#L785-L805).

**Design consequence:** do not let native serialized spell fields overwrite the server's committed Knowledge, undo abandonment, or persist combat Pending before Settlement. Build a versioned adapter that distinguishes `Native Save` (vanilla character/item state) from `Knowledge Ledger` (authoritative committed skill state) and `Pending Combat Credit`; reconcile all three on login and after a successfully accepted settlement. Test both single and multiplayer paths; no “just add fields to PlayerPack” shortcut until compatibility is proven.

## 5. Player death, resurrect, Normal and Hardcore: product decision

**FROZEN 2026-10-11:**

- **Normal** keeps *all applicable vanilla DevilutionX death-drop behavior*, including equipment, half gold, held items and PvP Ear as conditioned by vanilla location/death cause. **Only the DeathPact-specific not-yet-committed Pending Knowledge is lost** (subject to already accepted server settlements).
- **Hardcore** suppresses vanilla death-drop generation, including its unconditional held item handling, half-gold and full-equipment branches, and PvP Ear branch. Client may display a death animation locally, but **only a centrally committed, idempotent death result** can choose and release the roughly 65%-sampled *last server-committed equipped items* as Legacy Gear to *other previously co-present Hardcore room players*. No non-equipment items are additional collectible death drops under this design. Character is permanently locked; other items destroyed according to V1.

**Source evidence and risks:**
- `StartPlayerKill()` sends the peer death command (`CMD_PLRDEAD`) and independently decides `dropGold`, `dropItems`, `dropEar`; `DeadItem` creates ground items, and the equipment-drop loop `item.pop()` empties the equipped slots. See [`Source/player.cpp:2685-2792`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L2685-L2792) and [`Source/player.cpp:291-326`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L291-L326).
- Native `ApplyResurrect()` directly resets local death state, HP/Mana and character mode. Hardcore must guard **all revival paths**, not just reject logins for `DEAD`, and reject stale peer resurrection commands. See [`Source/spells.cpp:245-275`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/spells.cpp#L245-L275) and the `CMD_RESURRECT` command in [`Source/msg.cpp:133-153`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/msg.cpp#L133-L153).
- The original engine has **no central permanent death ledger** or delayed item-release ACK. Add the centrally versioned state machine and suppress Hardcore item mutations **before** native drop, including concurrent/duplicate death reports. On central outage, display unresolved/pending death; do not invent success or allow relog rollback. Exact live error UX needs tests.

**Tests:** Normal Monster/Trap death, Normal PvP, arena/level16 exceptions, Normal multiplayer remote item sync; Hardcore Monster/Trap/PvP/Pact, solo and 2/4 peer rooms, mid-death disconnect, duplicate notifications, offline/reconnect, resurrection while DEAD, host exit before claimed Legacy, and invalid old save.

## 6. Items, quota and ownership

- `SpawnLoot()` uses original drop logic and ultimately `SpawnItem()`; it must remain untouched for Normal/Hardcore Computer generation. See [`Source/monster.cpp:907-950`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/monster.cpp#L907-L950) and [`Source/items.cpp:3422-3481`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/items.cpp#L3422-L3481).
- Vanilla item reconstruction/validation uses seed, creation info and item index. Multiplayer `OnRequestGetItem()` performs local active-level owner checks and sends `CMD_GETITEM`; `IsPItemValid` checks structural item flags. See [`Source/msg.cpp:1273-1322`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/msg.cpp#L1273-L1322), [`Source/msg.cpp:1429-1469`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/msg.cpp#L1429-L1469) and [`Source/items/validation.cpp:158-195`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/items/validation.cpp#L158-L195).
- `GetItemRecord()` deduplication has a **6-second expiry**, including explicit commentary on same-seed quest-item collisions; it is a transient transport defense, **not global item identity**. See [`Source/items.cpp:4753-4799`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/items.cpp#L4753-L4799).
- `Player` stores equipment, backpack, belt and held item separately. See [`Source/player.h:210-234`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.h#L210-L234).

**Design consequence:** server-side `itemId` + provenance + mode + one-owner ledger are new; inspect and gate *every* relevant `InvGetItem`, `AutoGetItem`, `OnRequestGetItem`, `OnGetItem`, `CMD_PUTITEM` and item-use path. Reserve/deduct direct-Computer-pickup Loot Load atomically before accepting pickup. A player-dropped item's later pickup should cost 0, but still transfer exclusive ownership. Native six-second item record does not enforce daily quota or prevent forged server state.

## 7. Classless character is not an empty HeroClass

- Character selection currently lists `HeroClass` values. [`Source/DiabloUI/hero/selhero.cpp:171-208`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/DiabloUI/hero/selhero.cpp#L171-L208).
- Class data loads `baseStr/Mag/Dex/Vit`, `maxStr/Mag/Dex/Vit`, Life/Mana growth, attack and animation-relevant parameters. [`Source/tables/playerdat.cpp:169-207`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/tables/playerdat.cpp#L169-L207); sprite/animation logic refers to `_pClass` in [`Source/player.cpp:2135-2210`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L2135-L2210).
- `GetMaximumAttributeValue()` applies **class-specific** caps; `NextPlrLevel()` gives up to five point allocations (subject to caps) and class-dependent Life/Mana gains. [`Source/player.cpp:1650-1662`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L1650-L1662) and [`Source/player.cpp:2370-2421`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/player.cpp#L2370-L2421).
- Maximum level is data-driven via `GetMaximumCharacterLevel()`, not safely assumed from old Diablo design or a fixed value. [`Source/tables/playerdat.cpp:409-419`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/tables/playerdat.cpp#L409-L419).
- `IsNetPlayerValid()` rejects invalid `_pClass`. See [`Source/players/validation.cpp:14-27`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/players/validation.cpp#L14-L27).

**Frozen design:** valid internal HeroClass remains only for engine/graphic/save/network compatibility. Normalize ALL class-specific gameplay including stats/caps/growth, HP/Mana, combat, weapon handling, innate abilities and item/Skill eligibility. Shared numerical rules OPEN-10; valid native class field is NOT a gameplay profession.

## 8. Multiplayer, public Account identity and room lifecycle

- `InitMulti()` sizes player state for `MAX_PLRS` and enters original provider/game-selection and sync flow. See [`Source/multi.cpp:544-573`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/multi.cpp#L544-L573).
- In TCP transport, `tcp_client::create()` creates a **local `tcp_server`** before joining; `IsGameHost()` checks its existence. It is a transport relay, not the proposed DeathPact central account/asset service. See [`Source/dvlnet/tcp_client.cpp:27-35`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/dvlnet/tcp_client.cpp#L27-L35) and [`Source/dvlnet/tcp_client.cpp:129-144`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/dvlnet/tcp_client.cpp#L129-L144).
- `tcp_server::HandleReceiveNewPlayer()` admits peer connections and `SendPacket()` fans out/forwards messages; it does not verify DeathPact account authority, permanent death or a daily loot ledger. See [`Source/dvlnet/tcp_server.cpp:122-180`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/dvlnet/tcp_server.cpp#L122-L180).
- Host `tcp_client::SNetLeaveGame()` shuts down its local TCP server. See [`Source/dvlnet/tcp_client.cpp:243-254`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/dvlnet/tcp_client.cpp#L243-L254). ZeroTier implementation exists under `Source/dvlnet/protocol_zt.cpp`, but **its host departure/reconnection behavior was not demonstrated by this source audit**.
- Monster/item state sync is partly local-client owned; `SyncMonster`, `DeltaGetItem`, `OnGetItem` and other handlers must be audited before cross-peer Knowledge grant and central item registry are integrated. See [`Source/sync.cpp:157-210`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/sync.cpp#L157-L210) and [`Source/msg.cpp:1429-1532`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/Source/msg.cpp#L1429-L1532).

**New frozen identity requirement (design; NOT implemented upstream):** Account owns unique public `Name#Tag`; Character's private name is visible ONLY to its owning Account. Before Phase 5 implementation, inspect how native player/hero names are serialized, synchronized and rendered; ensure peers see Account `Name#Tag` and cannot receive the private Character name in peer-visible fields. Internal room sessions, saves, Hardcore death and item history still refer to stable `characterId` and `accountId`. Legacy provenance visibly carries the original owning Account `Name#Tag`.

**Design consequence:** retain native 1–4-player gameplay transport, add a separate central identity/committed-state service. Preserve deterministic keying across client joins, host departure and room shutdown; explicitly test whether Normal and Hardcore different game modes can be cleanly isolated at join.

## 9. Licensing and asset boundaries

- DevilutionX native source carries **Sustainable Use License 1.0**, permitting noncommercial/personal and internal use, with distribution free of charge for noncommercial purposes; review before any commercial distribution or charge. [`LICENSE.md:1-27`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/LICENSE.md#L1-L27).
- Original Diablo game assets are not automatically supplied or licensed by the source fork. Game README calls for legal original MPQ data or limited shareware `spawn.mpq` for the shareware subset. [README](https://github.com/diasurgical/DevilutionX/blob/master/README.md).
- Auxiliary [devilutionx-assets](https://github.com/diasurgical/devilutionx-assets) is an optional set of fonts, audio, localization and bundled-resource sources, not a substitute for the original Diablo game data. Keep its asset-specific licenses distinct.

## 10. Required work to finish Phase 0

1. **Pin the actual checked-out upstream base**, record local OS/compiler/CMake toolchain versions and a clean `git status`. Build **without game code modifications**. Windows: use Visual Studio C++ desktop/CMake and follow [`docs/building.md:263-312`](https://github.com/LinhaiShen/DeathPact/blob/bf574c9b74f1d718b0bcee15d648b20a8bfc927c/docs/building.md#L263-L312); alternatively use supported WSL/Linux prerequisites and build instructions from the same document.
2. Run native unit tests actually configured by CMake/CTest; capture exact commands and real results. If a test is unavailable, record a named blocker, not “pass”.
3. Perform **single-player** smoke: character create/load, kill, cast, consume Book, equipment and inventory, native save/reload, Normal monster death and, where possible, original resurrection.
4. Perform **2- and 4-player** smoke through TCP and ZeroTier separately: host create/join, kill and projectile behavior, pickup/drop, character save, host exit, reconnect and resync.
5. Populate `config/skill_action_mapping.csv` only after *per-Computer-action* evidence is collected; action IDs, `MissileID` and `SpellID` are not interchangeable. Leave ambiguous physical attacks unmapped.
6. Confirm original max-level/table configuration for selected Diablo/Hellfire ruleset, classless visual/presentation strategy, explicit stat cap policy, possible nonstandard Lua/dev hook influence, and BLOB/save migration strategy.
7. Write a short reproducible build-and-smoke appendix below **with actual logs** and only then declare **Phase 0 PASS**. The static audit alone is a **PARTIAL** result.

### Test execution log (pending)

| Check | Environment / command | Result |
| --- | --- | --- |
| Clean native configure | NOT RUN | BLOCKED: no checked-out local repo in this assistant session |
| Clean native build | NOT RUN | NOT VERIFIED |
| Native unit tests | NOT RUN | NOT VERIFIED |
| Single player launch/smoke | NOT RUN | NOT VERIFIED |
| TCP 2/4-player smoke | NOT RUN | NOT VERIFIED |
| ZeroTier smoke | NOT RUN | NOT VERIFIED |
| Native save/backwards compatibility | NOT RUN | NOT VERIFIED |

**Phase 1 recommendation:** prioritize (a) server-issued per-spawn and per-item identity boundaries and idempotency, (b) native save vs Knowledge reconciliation, (c) Normal/Hardcore death dispatch and resurrection lock, before large gameplay/UI rewrites. These are proposed implementation dependencies, not proof that the new features already exist.
