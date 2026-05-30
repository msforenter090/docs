# CPU Frontend Block Diagram

A common mental model for the frontend of a modern out-of-order CPU. The
blocks and their order are broadly the same across Intel, AMD, and ARM —
the names and sizes differ, and some blocks are specific to variable-length
ISAs (see notes below the diagram).

```
                        CPU FRONTEND
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              Branch Prediction Unit (BPU)           │  │
│   │   ┌───────────┐   ┌───────────┐   ┌─────────────┐  │  │
│   │   │    BTB    │   │    BHB    │   │     RSB     │  │  │
│   │   │  (Branch  │   │  (Branch  │   │   (Return   │  │  │
│   │   │  Target   │   │  History  │   │    Stack    │  │  │
│   │   │  Buffer)  │   │  Buffer)  │   │   Buffer)   │  │  │
│   │   └───────────┘   └───────────┘   └─────────────┘  │  │
│   └──────────────────────────┬──────────────────────────┘  │
│                              │ predicted PC                 │
│   ┌──────────────────────────▼──────────────────────────┐  │
│   │                 Instruction Fetch Unit               │  │
│   │        ┌─────────────────┐   ┌──────────────┐       │  │
│   │        │    L1 ICache    │   │     ITLB     │       │  │
│   │        └────────┬────────┘   └──────────────┘       │  │
│   └─────────────────┼───────────────────────────────────┘  │
│                     │ raw instruction bytes                 │
│   ┌─────────────────▼───────────────────────────────────┐  │
│   │                  Pre-decode / ILD                   │  │
│   │            (finds instruction boundaries)           │  │
│   └─────────────────┬───────────────────────────────────┘  │
│                     │                                       │
│   ┌─────────────────▼───────────────────────────────────┐  │
│   │               Instruction Queue (IQ)               │  │
│   └──────────────┬──────────────────────────────────────┘  │
│                  │                                          │
│      ┌───────────▼───────────┐   ┌───────────────────────┐ │
│      │        Decode         │   │      uop Cache        │ │
│      │   (legacy path)       │   │   (decoded-uop        │ │
│      │   x86 → uops          │   │    fast path)         │ │
│      └───────────┬───────────┘   └───────────┬───────────┘ │
│                  │                            │             │
│                  └──────────────┬─────────────┘             │
│                                 │                           │
│                  ┌──────────────▼──────────────┐            │
│                  │             LSD             │            │
│                  │   (Loop Stream Detector)    │            │
│                  └──────────────┬──────────────┘            │
│                                 │                           │
│                  ┌──────────────▼──────────────┐            │
│                  │             IDQ             │            │
│                  │  (Instruction Decode Queue) │            │
│                  └──────────────┬──────────────┘            │
│                                 │                           │
└─────────────────────────────────┼───────────────────────────┘
                                  │ uops
                    ┌─────────────▼──────────────┐
                    │           BACKEND           │
                    │     (Allocator / Renamer)   │
                    └─────────────────────────────┘
```

**ISA notes:**
- **Variable-length ISAs (x86):** need a heavy **Pre-decode / ILD** stage to
  find instruction boundaries before decoding, which is why the **uop cache**
  exists — to skip that expensive work on repeat execution. The **LSD** is a
  further optimization for tiny loops.
- **Fixed-width ISAs (ARM/AArch64, RISC-V):** instruction boundaries are
  trivial (step by a fixed width), so pre-decode is minimal and a uop cache is
  often unnecessary. These designs tend to drop the uop cache/LSD and instead
  decode very wide directly.

---

## Typical Values Across Architectures

Rough figures to build intuition — exact numbers vary by specific core revision.
"—" means the block is absent or not applicable on that design.

| Block / metric        | Intel Skylake   | AMD Zen 3        | Apple M1 (Firestorm) |
|-----------------------|-----------------|------------------|----------------------|
| ISA                   | x86-64 (var len)| x86-64 (var len) | AArch64 (fixed 4B)   |
| L1 ICache             | 32 KB, 8-way    | 32 KB, 8-way     | 192 KB               |
| ITLB entries (4K)     | 128             | 64               | ~256                 |
| Fetch width           | 16 B/cycle      | 32 B/cycle       | 8 instr/cycle        |
| Legacy decode width   | 4 instr/cycle   | 4 instr/cycle    | 8 instr/cycle        |
| uop cache             | DSB, 1536 uops  | Op cache, ~4096  | — (not needed)       |
| uop cache deliver     | 6 uops/cycle    | 8 ops/cycle      | —                    |
| LSD                   | yes (~25 uops)  | — (op cache)     | —                    |
| IDQ depth             | 64 uops         | ~? (op queue)    | very large           |
| Rename/issue width    | 4 uops/cycle    | 6 uops/cycle     | 8 uops/cycle         |
| Branch mispred penalty| ~16 cycles      | ~17 cycles       | ~13 cycles           |

Takeaways:
- **x86 cores** (Intel/AMD) lean on a uop cache to dodge expensive variable-length
  decode; AMD's op cache is larger and wider than Intel's DSB.
- **Apple M1** has no uop cache at all — fixed-width AArch64 makes decode cheap,
  so it just decodes 8-wide directly and spends the transistor budget on a huge
  L1 ICache and reorder window instead.
- Decode/rename **width grows** as you move toward wider designs (4 → 6 → 8),
  which is the headline frontend throughput number.

---

## Block Descriptions

**Branch Prediction Unit (BPU)**
Predicts the next fetch address before the current instruction finishes executing.
Contains three sub-structures: BTB for branch targets, BHB for branch history
patterns, and RSB for return addresses. A misprediction flushes the pipeline
and costs on the order of 15–20 cycles.

**L1 ICache**
First-level instruction cache. Delivers raw instruction bytes to the
pre-decoder each cycle. On a miss, fetches from L2, then L3/DRAM.

**ITLB**
Translates virtual instruction addresses to physical. A miss triggers a page
table walk costing hundreds of cycles.

**Pre-decode / ILD (Instruction Length Decoder)**
Scans the raw byte stream to find instruction boundaries. Needed for
variable-length ISAs where instructions span a range of byte widths, so this
step must happen before decoding. Minimal on fixed-width ISAs.

**Instruction Queue (IQ)**
Small buffer between pre-decode and the decoders. Absorbs bubbles caused by
pre-decode throughput variations.

**Decode (legacy path)**
Converts architectural instructions into uops. Typically several decoders in
parallel (one complex + several simple). Active when the uop cache misses.

**uop Cache**
Caches already-decoded uops indexed by instruction address. On a hit, uops
flow directly to the IDQ, bypassing pre-decode and decode entirely — the main
frontend fast path on variable-length ISAs.

**LSD (Loop Stream Detector)**
Detects small loops that fit entirely within the IDQ. Locks them in the IDQ
and replays them without going back to the uop cache or decode. Zero frontend
overhead — the best possible case for tight loops.

**IDQ (Instruction Decode Queue)**
Buffer between the frontend and backend. Acts as the handoff point. The
backend pulls uops from the IDQ. Frontend stalls show up as IDQ empty cycles.

---

## Parameters & Perf Events

Perf event names below are Intel-specific (use `perf list` for your CPU; AMD
and ARM expose analogous counters under different names).

| Block       | Parameter              | Description                                      | Perf Event                              |
|-------------|------------------------|--------------------------------------------------|-----------------------------------------|
| BPU         | Branch prediction rate | % of branches correctly predicted               | `branch-misses` / `branches`            |
| BPU / BTB   | BTB misses             | Branch target not found in BTB                   | `br_inst_retired.indirect`              |
| BPU         | Misprediction penalty  | Pipeline flush cycles on wrong prediction        | `int_misc.recovery_cycles`              |
| BPU / RSB   | RSB underflow          | Return address not in RSB, falls back to BTB     | `br_misp_retired.ret`                   |
| L1 ICache   | Miss rate              | Instruction fetches that miss L1                 | `L1-icache-load-misses`                 |
| L1 ICache   | Miss penalty           | Cycles stalled on L1 icache miss                 | `icache_64b.iftag_stall`                |
| ITLB        | Miss rate              | Instruction address translations that miss ITLB  | `iTLB-load-misses`                      |
| ITLB        | Miss penalty           | Cycles stalled on ITLB miss / page walk          | `itlb_misses.walk_duration`             |
| Decode      | Legacy decode uops     | uops coming from slow decode path (not uop cache)| `idq.mite_uops`                         |
| Decode      | Legacy decode cycles   | Cycles the legacy decoder was active             | `idq.mite_cycles`                       |
| uop Cache   | uop-cache uops         | uops delivered from the uop cache (fast path)    | `idq.dsb_uops`                          |
| uop Cache   | uop-cache active cycles| Cycles the uop cache was delivering uops         | `idq.dsb_cycles`                        |
| uop Cache   | Fallback switches      | Penalty cycles when falling back to legacy decode| `dsb2mite_switches.penalty_cycles`      |
| LSD         | LSD uops               | uops delivered by the loop stream detector       | `lsd.uops`                              |
| LSD         | LSD active cycles      | Cycles the LSD was active                        | `lsd.cycles_active`                     |
| LSD         | LSD not active         | Cycles LSD could have been used but wasn't       | `lsd.not_active`                        |
| IDQ         | IDQ empty cycles       | Frontend stall — IDQ ran dry                     | `idq.empty`                             |
| IDQ         | Uops not delivered     | uops the frontend failed to deliver to backend   | `idq_uops_not_delivered.core`           |
| IDQ         | IDQ full cycles        | Backend stall — IDQ was full                     | `idq.full`                              |
| Frontend    | Frontend bound         | % of cycles frontend was the bottleneck          | `topdown-fetch-bubbles` / TMA level 1   |
