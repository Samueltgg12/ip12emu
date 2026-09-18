# CLAUDE.md

This file complements `AGENTS.md` for Claude Code agents.

Follow **`AGENTS.md`** as the single source of truth for project guidelines, phases, and conventions.

Quick orientation for Claude:

- Project: **ip12emu** — a fast (not accurate) C++ emulator of the SGI Indigo **IP12** (MIPS R3000A, ≤96 MB RAM).
  **Speed and JIT are the whole point.** See `AGENTS.md` §2.
- **Phase 1 (documentation) is current.** Track progress in `ROADMAP.md`. Do not start Phase 2 code without
  confirming with the user.
- Read the relevant `docs/<chip>/` file before touching anything related to that chip.
- `samples/` (NetBSD/Linux/GXemul trees, PROM dumps) is **read-only reference**. Never edit or commit it.
- PROM dumps are copyrighted — never put them in a public repo (already gitignored).
- After finishing a task, update `ROADMAP.md`.
- Code: C++20, no comments unless asked, fast-path performance first (`AGENTS.md` §8).

Start work by reading `AGENTS.md` → `ROADMAP.md`, then dive into `docs/`.