# Codex instructions — DevilutionX ARPG V1

Read `docs/README.md` first, then only the relevant spec files for the current task.

## Non-negotiable rules
- This is a **native desktop DevilutionX fork**, not a browser game. Reuse the pinned upstream engine. Do not implement Three.js, browser rendering, microservices, replay, or a server-authoritative combat simulation.
- Do not replace original combat, Mana, STR/MAG/DEX/VIT, inventory, equipment, affixes, monster decision/action mechanisms, hotkeys, or transport except where a numbered V1 requirement explicitly calls for a minimal hook.
- One character has one **Momentum** value. Base natural Mana recovery = 0; Momentum grants recovery. Battle Settlement resets Momentum. No Cooldown Group or Momentum Group.
- Maximum 30 distinct Knowledge skills. Following (even below Lv1) and Learned (including Followed Lv1+) each count **once** toward Burden. Starting Burden Cap = 10; full MAG point allocation at configured max level can reach 30. Misfire = min(0.50, max(0, Burden-Cap)/Cap), consuming normal Mana with no skill effect. **Manual skill abandonment is allowed ONLY as part of Battle Settlement**; it irreversibly removes the selected skill's true learned level, stored learning progress and Follow state, frees one Burden/skill slot, and clears associated spell hotkeys. Acquiring the skill again starts from Lv0/zero progress. Never reintroduce abandoned progress through Native Save or a queued credit.
- Computer `SkillUsedEvent` represents an actually executed vanilla attack/spell, not preassigned monster skills. Deduplicate per action/cast and credit at most once per Computer+Skill on kill. Room party members share the eligible knowledge credit.
- Spell Book grants Lv1 only if below Lv1; never increases Lv1+. Skill Points increase progress only for Followed skills.
- **Battle Settlement commits only pending Knowledge** (including user-requested skill abandonment), is irreversible once started, clears Momentum; Normal death loses uncommitted Knowledge. Do not confuse abandoning a **skill within** Settlement with cancelling Settlement itself. Native Save/critical events synchronize non-Knowledge state. Loot Load is server-ledgered on pickup.
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
