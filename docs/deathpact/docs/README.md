# DevilutionX ARPG V1 — Codex specification pack

**Repository context:** This pack is the DeathPact V1 native-DevilutionX implementation specification inside the DevilutionX fork. The upstream engine files and existing upstream documentation are unchanged. These are proposed game extensions, not functionality already present in upstream; use this pack as the design reference for DeathPact tasks.

**Frozen:** On skill abandonment during Settlement, previously spent Skill Points are not refunded.

Version: `V1-DRAFT-2026-10-10-REV1`  · Status: implementation-ready **at the boundary/invariant level**; tunable/open items explicitly listed.

Read order: `../AGENTS.md` → `V1_SPEC.md` → relevant `ENGINE_INTEGRATION.md` / `SERVER_CONTRACT.md` → `IMPLEMENTATION_PLAN.md` → `OPEN_DECISIONS.md`.

Priority: (1) latest explicit decisions in `V1_SPEC.md` → (2) `OPEN_DECISIONS.md` for undecided behavior → (3) pinned, actually inspected DevilutionX upstream behavior. Do not treat earlier browser/Three.js gameplay concepts as requirements.

**Not a claim that code already exists.** This pack describes intended extension hooks and contracts. Codex must inspect its actual checked-out DevilutionX version, write an upstream audit, then implement.

**License gate**: Upstream DevilutionX repository is distributed under Sustainable Use License (source is non-commercial); original Diablo assets are separately protected. Confirm legally permitted distribution/use before public or paid release. Upstream references: https://github.com/diasurgical/DevilutionX ; https://github.com/diasurgical/DevilutionX/blob/master/LICENSE.md
