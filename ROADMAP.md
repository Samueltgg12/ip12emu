# ip12emu — Roadmap

Phase tracker for the ip12emu project. Update checkboxes as work is done.
Legend: `[x]` done · `[ ]` pending · `[ ]` (next) = currently in progress

Project: C++ emulator of the **SGI Indigo (IP12 / "Hollywood")** — MIPS R3000A @ 33 MHz, ≤96 MB RAM.
Mission: **maximum speed with every optimization + JIT; not hardware accuracy.**

---

## Phase 1 — Documentation (CURRENT)

Goal: everything an implementer needs is written down and sourced before any code.

- [x] Create `docs/` skeleton (per-chip folders, references, opcodes, firmware)
- [x] Acquire reference source trees into `samples/` (NetBSD usr/src, Linux kernel, GXemul 0.7.0)
- [x] Acquire IP12 PROM dumps (`samples/*.u56` — SGI 070-8088-001 / -002 / unknown, 256 KB each)
- [x] Identify that PROM dumps are **16-bit halfword byte-swapped** (needs de-swap before use)
- [x] Download key NetBSD sgimips files into `docs/references/netbsd-sgimips/`
- [x] Add Seeq 80C03 datasheet (`docs/references/pdf-md/80c03.pdf` + `.md`)
- [x] Add MIPS opcode tables (`docs/opcodes/MipsOpcodes.cpp` + README)
- [ ] Document **IP12 memory map** (from `sgimips/machdep.c`, `hpcreg.h`, `picreg.h`)
- [ ] Document **MIPS R3000A** CPU (ISA, caches, TLB, exceptions, boot reset) → `docs/cpu/`
- [ ] Document **HPC1/HPC1.5** chip: DMA descriptors, SCSI windows, ethernet glue, interrupts → `docs/hpc/`
- [ ] Document **PIC** chip: CPU control, big-endian bit, refresh, graphics DMA, GIO provision → `docs/pic/`
- [ ] Document **WD33C93** SCSI controller (registers, DMA handshake) → `docs/scsi/`
- [ ] Document **Seeq 80C03** Ethernet controller (DIF bus interface) → `docs/ethernet/`
- [ ] Document **Motorola DSP56001** audio subsystem (4ch 16-bit, codec, DSP program) → `docs/dsp/`, `docs/audio/`
- [ ] Document **Z8530** serial + console semantics → `docs/` (console)
- [ ] Document **Firmware/ARCS**: PROM layout after de-swap, boot sequence, ARCS calls, IRIX entry → `docs/firmware/`
- [ ] Document **GIO32 bus**: addressing, slots, graphics boards expected (LG1, XS24) → `docs/gio/`
- [ ] Appendix: how GXemul's IP12/ARCS emulation works → `docs/firmware/` or references

## Phase 2 — The Emulator

Goal: boot the real machine under emulation, fast.

### Core scaffolding
- [ ] CMake project (`ip12emu`), C++20, zero unintended dependencies
- [ ] `src/` layout mirroring `docs/` (core / devices / jit)
- [ ] Word access helpers (big-endian load/store, host-endian safe)

### CPU
- [ ] MIPS I (R3000A) interpreter — full integer + FPU (R3010) + coprocessor 0
- [ ] 64-entry TLB (or simplified mapping if proving it's enough)
- [ ] Exceptions/interrupts, hazard behaviour sufficient for guest software
- [ ] Instruction cache + data cache as timing/coherency model (as simple as possible)
- [ ] Basic **JIT**: per-block compilation, trampoline dispatch, direct block linking
- [ ] Opcode coverage test vs `docs/opcodes/MipsOpcodes.cpp`

### Memory & system
- [ ] RAM up to 96 MB, flat fastmem-style access (pointer-mapped, host-endian swapped)
- [ ] PROM mapping at `0xbfc00000` (de-swapped images from `samples/*.u56`, local only)
- [ ] Device aperture dispatch (function-pointer table; lazy device creation)

### Devices
- [ ] **PIC** (CPU control, endianness, refresh, graphics DMA hooks) — BIOS/PROM requirements first
- [ ] **HPC1 DMA engine** (descriptor chains, SCSI + ethernet channels)
- [ ] **WD33C93 SCSI** + disk/tape image backends (SCSI-1) — enough to boot from disk
- [ ] **Seeq 80C03 Ethernet** + host networking backend (pcap/TAP or null-device stubs)
- [ ] **Z8530** serial console (IRIX/NetBSD console via tty)
- [ ] RTC + NVRAM (DS1386/DS1286-class), front-panel button
- [ ] **DSP56001** audio (mix 4ch 16-bit, codec emulation) — minimal then full

### Graphics / GIO
- [ ] GIO32 bus model + slots
- [ ] Text console path via PROM + serial; framebuffer (XS24-class) as stretch milestone

### Boot milestones
- [ ] PROM image de-swapper tool/step works; PROM starts and prints
- [ ] PROM reaches ARCS prompt (interactive or automated script)
- [ ] NetBSD/sgimips `GENERIC32_IP12` boots to multi-user
- [ ] IRIX 4.0.x boots (runs `sash` → kernel → userland)
- [ ] Suspend/resume (snapshot) core state

## Phase 3 — Performance & Polish

- [ ] JIT: register allocation, larger superblocks, block chaining, code cache flushing, FPU emission
- [ ] Threaded device execution / async DMA, host SIMD for framebuffer/audio blending
- [ ] Memory: direct-mapped guest pages, page-table fast paths, hugepage-friendly allocations
- [ ] CPU frequency governor / dynamic timekeeping so guest timers track host time cheaply
- [ ] Performance harness: host clock vs guest boot time, per-device traffic counters
- [ ] Networking: full pcap/TAP modes, built-in virtual hub
- [ ] Save states, CLI/config, friendly debugger (disasm + watchpoints)
- [ ] Packaging (static binary), docs polish, GitHub release
- [ ] Extra board support as stretch: Hollywood Light (HPLC), 4D/30 EISA/VME variants