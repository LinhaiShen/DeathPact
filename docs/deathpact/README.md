# DeathPact — DevilutionX ARPG V1 specification

This is the **native desktop DevilutionX-based DeathPact** product and Codex implementation specification. The repository is a fork of [diasurgical/DevilutionX](https://github.com/diasurgical/DevilutionX).

The specification is deliberately kept under `docs/deathpact/`; all existing upstream engine files and documentation remain unchanged. It describes **planned DeathPact extensions**, not features already implemented in the fork.

Start with [AGENTS.md](AGENTS.md), then [docs/README.md](docs/README.md), [docs/V1_SPEC.md](docs/V1_SPEC.md), [docs/ENGINE_INTEGRATION.md](docs/ENGINE_INTEGRATION.md), [docs/SERVER_CONTRACT.md](docs/SERVER_CONTRACT.md), and [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md). See [docs/OPEN_DECISIONS.md](docs/OPEN_DECISIONS.md) for unsettled parameters.

**Current frozen principles:** Computer killed after actually using mapped Skill yields exactly Learning +1 per Skill for eligible pre-Following Characters, regardless of cast count/monster level; Book consumes one and grants true Spell Level+1; 30 is a BurdenCap ceiling, NOT a hard skill limit; retained HeroClass is visual/engine-only, no gameplay distinctions. Skill abandonment at Battle Settlement permanently erases level/progress/Following without refunding invested Skill Points.

Codex must begin with Phase 0: audit the actual checked-out DevilutionX revision, then implement milestones incrementally. Read the package-local `AGENTS.md` explicitly when working on gameplay code outside this directory.
