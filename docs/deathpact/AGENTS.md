# Codex instructions — DevilutionX ARPG V1

Read `docs/README.md` first, then only the relevant spec files for the current task.

## Non-negotiable rules
- This is a **native desktop DevilutionX fork**, not a browser game. Reuse the pinned upstream engine. Do not implement Three.js, browser rendering, microservices, replay, or a server-authoritative combat simulation.
- Do not replace original combat, Mana, STR/MAG/DEX/VIT, inventory, equipment, affixes, monster decision/action mechanisms, hotkeys, or transport except where a numbered V1 requirement explicitly calls for a minimal hook.
- Retain valid native Player::_pClass/HeroClass strictly for engine, serialization, sprites and animation. **Eliminate ALL gameplay profession differences** (starting skills/stats, growth and caps, damage/defense, weapon handling, innate abilities, equipment/Skill eligibility). Shared neutral numeric values remain OPEN-10.
- One character has one **Momentum** value. Base natural Mana recovery = 0; Momentum grants recovery. Battle Settlement resets Momentum. No Cooldown Group or Momentum Group.
- No 30-distinct-Skill hard cap: CurrentBurden=|Following union Learned| may exceed 30; only BurdenCap tops out at 30 and excess invokes Misfire rather than preventing learning. Voluntary abandonment only in Settlement resets Knowledge and reduces Burden, never refunds spent Skill Points, and removes only learned-Spell hotkeys (not independent Scroll/Staff shortcuts).
- For each Computer spawn, keep a set of canonical mapped Skill IDs actually executed at least once. At death, each set member yields exactly Pending Learning +1 to each eligible Character already Following; others see a candidate. Repeated casts/projectiles/monster level do not affect credit. Idempotent by roomSessionId+computerSpawnId+skillId.
- Spell Book preserves original one valid Book consumed -> +1 TRUE Spell Level up to native MaxSpellLevel, via an immediate atomic item+Knowledge server mutation. Do not waste an already-maxed Spell's Book; Books for OTHER/new Skills remain legal at Burden>=30 and raise Over-Burden. Do not auto-Follow.
- **Battle Settlement commits only Pending Knowledge** including Settlement-only voluntary abandonment. Account is the **ONLY** owner of unique public `Name#Tag`; Account-friend/room/public chat identity always displays the Account `Name#Tag`, never a Character's private name. Character names are shown **only to the owning Account** in character selection/management; use opaque `characterId` for game/session/ledger data. Native peer name/UI fields must use the public Account label, not the Character name. Settlement is irrevocable after commit; Normal death loses uncommitted Knowledge, separate native save persists non-Knowledge state.
- Hardcore death is permanent. Server chooses randomly ~65% of *equipped* items to become Legacy drops after death report; only other Hardcore players present at death may pick up. Solo/unclaimed items on room/host termination vanish. Death Pact requires server registration BEFORE applying effects, and must inevitably cause death.
- V1 anti-cheat is deliberately limited to checkpointed file integrity plus server-side validation/registries. Never claim a client hash proves an honest simulation.

## Coding discipline
1. **Phase 0 first**: pin the actual upstream commit, build/test unmodified game, audit source entry points; record discoveries in `docs/UPSTREAM_AUDIT.md`. Paths in the spec are candidate locations, not guarantees about function names/signatures.
2. Implement one bounded milestone from `docs/IMPLEMENTATION_PLAN.md` per task. Begin with a brief file-level plan; do not silently enlarge scope.
3. Every persistent operation needs stable operation ID, idempotency, revision check, and tests for duplication/retry.
4. Preserve original save formats unless a versioned, backwards-compatible migration is explicitly planned. Separate Knowledge pending from vanilla saved spell levels.
5. For every ambiguity tagged `OPEN`, add a configuration placeholder or an explicit pending decision. Do not silently invent gameplay rules, values, spell mappings, or entitlements.
6. Provide changed files, tests run, any remaining risks. Do not claim tests passed if not run.

## Documents
- `docs/V1_SPEC.md` — gameplay and requirements; source of truth.
- `docs/SERVER_CONTRACT.md` — persistence, schema and protocol proposal.
- `docs/ENGINE_INTEGRATION.md` — upstream touchpoints, side effects, verification.
- `docs/IMPLEMENTATION_PLAN.md` — atomic coding milestones and acceptance.
- `docs/OPEN_DECISIONS.md` — parameters not yet frozen; do not guess.
