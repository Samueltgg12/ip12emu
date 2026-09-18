# AGENTS.md — Project Guidelines for AI Agents

This file is the canonical guide for any AI agent (Claude, Gemini, opencode, Cursor, etc.)
working in this repository. **Read this file first, before any other work.**

---

## 1. What this project is

**ip12emu** is a C++ emulator of the **original SGI Indigo (IP12, board codenames "Hollywood" / "Hollywood Light")**,
the low-end MIPS workstation SGI released in 1991.

The emulated machine:

| Component | Hardware |
|---|---|
| CPU | MIPS R3000A @ 33 MHz (MIPS I, 32-bit, big-endian) |
| RAM | up to 96 MB |
| System / DMA chip | **HPC** (HPC1 / HPC1.5) — DMA engine, hosts the WD33C93 SCSI and the Seeq 80C03 Ethernet, panel button, keyboard controller |
| SCSI | **WD33C93** (SCSI-1, fast/wide-wire SCSI host adapter) |
| Ethernet | **Seeq 80C03** (10 Mbit Ethernet controller), DMA via HPC |
| Audio | **Motorola DSP56001** DSP, 4-channel 16-bit audio |
| GIO bus | **PIC** chip (also CPU/peripheral control: big-endian mode, DRAM refresh, cache refill, graphics DMA). 32-bit GIO slots for graphics boards (LG1, XS24, etc.) |
| Serial / console | Z8530 (2 ports), keyboard/mouse via zs or pckbc |
| Clock / NVRAM | DS1386 / DS1286-class RTC+NVRAM (see NetBSD `sgimips/clock.c`) |
| Firmware | SGI ARCS-style PROM (IP12 PROM, SGI part 070-8088-001 / 070-8088-002) |
| OS targets | IRIX 4.0.x, NetBSD/sgimips `GENERIC32_IP12`, Linux (debug) |

## 2. Non-negotiable design philosophy

> **This emulator is NOT about hardware accuracy. It is about SPEED.**
> Fast as possible, using all kinds of optimizations and JIT.

Acceptable trade-offs (encouraged, not optional):
- No cycle-accurate timing. Instruction budgeting / sampling is fine.
- No wait states, no bus-timing modelling, no DRAM/parity simulation unless something needs it.
- Caches and TLB (R3000 has a 64-entry TLB) may be simplifed or skipped if a workload doesn't need them.
- Devices can be lazily evaluated, state-snapshot trampolined, or half-baked in behavior as long as
  target software (PROM → ARCS → IRIX/NetBSD) runs.
- The R3000A core is small and simple: prefer a **JIT** (block compilers, template-dispatched interpreter
  fallback) over a "textbook" interpreter that is slower.
- Prefer `memcpy`/vector loads for DMA, not word-by-word loops.
- Everything performance-sensitive goes through `restrict`, local hot variables, branch-predictable code.

If a change makes the guest boot that much faster but is "not historically accurate", it is welcome.
If a change is historically accurate but measurably slower, it is usually rejected.

## 3. Phases and where we are

Project is divided into phases, tracked in **`ROADMAP.md`** (authoritative progress tracker —
update it whenever you complete/abandon work).

- **Phase 1 — Documentation** (CURRENT). Build docs in `docs/`, gather datasheets, reverse-engineer notes.
- **Phase 2 — The Emulator.** Real code: R3000A core, memory, devices, JIT, boot firmware, OS boot milestones.
- **Phase 3 — Performance & polish.** JIT optimization, multithreading, save states, UI, debugger, distribution.

**Do not start Phase 2 implementation without checking Phase 1 status and the user.** When in doubt, ask.

## 4. Repository layout

```
ip12emu/
├── AGENTS.md                  <- this file
├── CLAUDE.md / GEMINI.md      <- agent-specific thin wrappers around AGENTS.md
├── README.md                  <- GitHub-facing description
├── ROADMAP.md                 <- phase progress tracker (update it!)
├── .gitignore                 <- excludes samples/ and build artifacts
├── docs/                      <- all documentation lives here
│   ├── audio/  cpu/  dsp/  ethernet/  firmware/  gio/
│   ├── hpc/  memory/  pic/  scsi/
│   ├── opcodes/               <- MIPS opcode tables (MipsOpcodes.cpp + README) — Phase 2 goldmine
│   └── references/
│       ├── pdf-md/            <- datasheets as .pdf + .md (e.g. seeq 80C03)
│       └── netbsd-sgimips/    <- key NetBSD sgimips sources (HP/C/PIC/clock/machdep...)
├── samples/                   <- LOCAL REFERENCE MATERIAL. Read-only, never commit.
│   ├── netbsd/                <- full NetBSD source tree (usr/src) — sgimips arch is the #1 reference
│   ├── linux/                 <- Linux kernel tree — sgi-ip22 drivers (sgiseeq, wd33c93, zs...)
│   ├── gxemul-0.7.0/          <- GXemul source — has an experimental "Iris Indigo IP12" machine + arcbios.c (ARCS PROM emulation)
│   └── *.u56                  <- IP12 PROM dumps (see §6)
```

## 5. Agent working rules

1. **Read AGENTS.md, then ROADMAP.md, then the relevant `docs/` files before any change.**
2. `samples/` is **read-only reference material — never modify files under it.**
3. When you produce a hardware doc, put it in the matching `docs/<chip>/` folder.
4. Update `ROADMAP.md` checkboxes after finishing/abandoning a task.
5. Do not commit anything unless explicitly asked. If asked to commit, keep PROM dumps and samples out of Git.
6. Follow the C++ conventions below for any code you write.
7. Never add comments to code unless asked (keep code self-documenting); docs live in `.md` files.
8. Verify code: prefer `cmake --build` + the project's tests. There is no lint/CI yet — Phase 2 may add it.
9. If a Phase 1 doc is missing for a chip you must implement, write the doc FIRST (even terse), then implement.

## 6. Critical hardware gotchas

- **Endianness**: MIPS R3000A is big-endian. The PIC must be initialised with `PIC_CPUCTRL_BIGENDIAN`.
- **PROM images in `samples/*.u56` are 16-bit halfword byte-swapped.** (Verified: strings read
  `eMomyrd aingsoit...`, `1032547698BADCFE`; first word `f00b 8000` de-swaps to `0x0bf00080` = `j 0xbfc00200`,
  a sane reset entry.) De-swap each 16-bit word to get the true big-endian image.
- Three distinct images exist: `070-8088-001__a.u56`, `ip12prom.070-8088-002.u56`, `ip12prom.070-8088-xxx.u56`
  (all 256 KB, all pairwise different — treat as different PROM revisions/board variants).
- Boot: MIPS reset vector is `0xbfc00000`; IP12 maps the PROM there. NetBSD IP12 bootstrap load
  address is `0x80368000`, kernel load `0x80002000`.
- IP12 subvariants per NetBSD `machtype.h`: `MACH_SGI_IP12_4D_3X` (120, 4D/3x), `MACH_SGI_IP12_VIP12` (121, 6U VME card),
  `MACH_SGI_IP12_HP1` (122, Hollywood = Indigo R3K), `MACH_SGI_IP12_HPLC` (123, Hollywood Light).
- HPC1 vs HPC3 differ (HPC1 is older, byte-weighted DMA descriptors; HPC3 is 128-bit). IP12 uses **HPC1 family**.
- SCSI DMA regs are windowed through HPC1 (e.g. `HPC1_SCSI0_REGS` at offset `0x88`); the WD33C93 itself is
  programmed via the HPC's DMA/shared buffers.
- The Seeq 80C03 is a DIF/memory-mapped Ethernet controller; HPC provides its DMA ring/descriptor glue.

## 7. Reference sources (in priority order)

1. `samples/netbsd/usr/src/sys/arch/sgimips/` — **netbsd/sgimips is the definitive IP12 reference.**
   Key files: `hpc/hpcreg.h`, `hpc/hpcdma.{c,h}`, `hpc/hpc.c`, `hpc/wdsc.c` (WD33C93 + HPC),
   `hpc/if_sq.c`, `hpc/sqvar.h` (Seeq), `hpc/pckbc_hpc.c`, `hpc/pi1ppc.{c,var}` (parallel port),
   `dev/pic.c`, `dev/picreg.h` (PIC!), `dev/zs*.c`, `dev/dsclock.c`, `dev/dpclock.c`,
   `sgimips/machdep.c` (memory map / boot), `sgimips/clock.c`, `sgimips/autoconf.c`,
   `include/machtype.h`, `include/bootinfo.h`, `include/intr.h`, `conf/GENERIC32_IP12`.
2. `samples/linux/` — IP22 cousins: `arch/mips/sgi-ip22/*` (HPC/GIO/setup/irq), `drivers/net/ethernet/seeq/sgiseeq.c`
   (Seeq 80C03!), `drivers/scsi/wd33c93.{c,h}`, `drivers/tty/serial/zs.*`, `arch/mips/sgi-ip22/ip22-gio.c`.
3. `samples/gxemul-0.7.0/` — experimental **IP12** machine (`src/machines/machine_sgi.c`, subtype "IP12"),
   ARCS PROM emulation (`src/promemul/arcbios.c`), guest-side boot strapping ideas.
4. `docs/references/pdf-md/` — real datasheets (e.g. `80c03.pdf` + `80c03.md`).
5. `docs/references/netbsd-sgimips/` — quick-reference copies of the most important NetBSD files.
6. `docs/opcodes/` — MIPS I/III/IV opcode tables (from a PS1/PS2 assembler) for the core/JIT.

## 8. C++ conventions (Phase 2 onwards)

- C++20, single self-contained project. **No external dependencies** unless clearly justified (a JIT backend/DSP
  are candidates; justify first).
- Directory-per-concern under `src/` (e.g. `src/core/`, `src/devices/`, `src/jit/`), mirror `docs/` naming.
- CMake build (project name `ip12emu`).
- Big-endian-aware helpers for loading/storing words (MIPS is BE; host may be LE).
- Performance first: hot loops compiled to tight straight-line code; device access via fast function-pointer
  dispatch tables; avoid virtual dispatch on the hot path; keep the memory model flat (e.g. pointer-mapped RAM,
  fastmem-style page tables).
- Use the terminos: **core/interpreter**, **JIT**, **device**, **memory map** — consistent naming across src and docs.
- No comments in code unless asked; write into `.md` instead.

## 9. Legal note

The IP12 PROM dumps (`samples/*.u56`) are **copyrighted SGI/IRIX firmware**. They must stay out of any public
repository, distribution, or documentation dump. `samples/` is gitignored. Never copy PROM content into docs
verbatim beyond short disassembly snippets used for documentation purposes.