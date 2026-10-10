# DeathPact — DevilutionX ARPG V1 specification

This is the **native desktop DevilutionX-based DeathPact** product and Codex implementation specification. The repository is a fork of [diasurgical/DevilutionX](https://github.com/diasurgical/DevilutionX).

The specification is deliberately kept under `docs/deathpact/`; all existing upstream engine files and documentation remain unchanged. It describes **planned DeathPact extensions**, not features already implemented in the fork.

Start with [AGENTS.md](AGENTS.md), then [docs/README.md](docs/README.md), [docs/V1_SPEC.md](docs/V1_SPEC.md), [docs/ENGINE_INTEGRATION.md](docs/ENGINE_INTEGRATION.md), [docs/SERVER_CONTRACT.md](docs/SERVER_CONTRACT.md), and [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md). See [docs/OPEN_DECISIONS.md](docs/OPEN_DECISIONS.md) for unsettled parameters.

Confirmed rule: abandoning a skill **during Battle Settlement** permanently discards its level, progress and Following state, without refunding invested Skill Points. Relearning begins at Lv0.

Codex must begin with Phase 0: audit the actual checked-out DevilutionX revision, then implement milestones incrementally. Read the package-local `AGENTS.md` explicitly when working on gameplay code outside this directory.
