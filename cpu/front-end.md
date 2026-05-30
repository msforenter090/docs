# CPU Frontend Block Diagram (Intel Skylake)

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
│   │        │   L1 ICache     │   │     ITLB     │       │  │
│   │        │   (32KB, 8-way) │   │  (128 entry) │       │  │
│   │        └────────┬────────┘   └──────────────┘       │  │
│   └─────────────────┼───────────────────────────────────┘  │
│                     │ raw x86 bytes (up to 16B/cycle)       │
│   ┌─────────────────▼───────────────────────────────────┐  │
│   │            Pre-decode / ILD                         │  │
│   │        (finds instruction boundaries)               │  │
│   └─────────────────┬───────────────────────────────────┘  │
│                     │                                       │
│   ┌─────────────────▼───────────────────────────────────┐  │
│   │              Instruction Queue (IQ)                 │  │
│   └──────────────┬──────────────────────────────────────┘  │
│                  │                                          │
│      ┌───────────▼───────────┐   ┌───────────────────────┐ │
│      │        Decode         │   │          DSB          │ │
│      │   (Legacy / MITE)     │   │  (Decoded Stream      │ │
│      │  4 decoders           │   │   Buffer)             │ │
│      │  4 uops/cycle         │   │  1536 uops            │ │
│      │                       │   │  6 uops/cycle         │ │
│      └───────────┬───────────┘   └───────────┬───────────┘ │
│                  │                            │             │
│                  └──────────────┬─────────────┘             │
│                                 │                           │
│                  ┌──────────────▼─────────────┐             │
│                  │             LSD             │             │
│                  │   (Loop Stream Detector)    │             │
│                  │   ~25 uops, 0 fetch cost    │             │
│                  └──────────────┬──────────────┘             │
│                                 │                           │
│                  ┌──────────────▼─────────────┐             │
│                  │             IDQ             │             │
│                  │  (Instruction Decode Queue) │             │
│                  │   64 entries               │             │
│                  └──────────────┬──────────────┘             │
│                                 │                           │
└─────────────────────────────────┼───────────────────────────┘
                                  │ up to 4 uops/cycle
                    ┌─────────────▼──────────────┐
                    │           BACKEND           │
                    │     (Allocator / Renamer)   │
                    └─────────────────────────────┘
```

---

## Block Descriptions

**Branch Prediction Unit (BPU)**
Predicts the next fetch address before the current instruction finishes executing.
Contains three sub-structures: BTB for branch targets, BHB for branch history
patterns, and RSB for return addresses. A misprediction flushes the pipeline
and costs ~15-20 cycles.

**L1 ICache**
32KB 8-way set-associative instruction cache. Delivers up to 16 bytes of raw
x86 bytes per cycle to the pre-decoder. On a miss, fetches from L2 (4-cycle
penalty) or L3/DRAM.

**ITLB**
Translates virtual instruction addresses to physical. 128 entries for 4KB
pages on Skylake. A miss triggers a page table walk costing hundreds of cycles.

**Pre-decode / ILD (Instruction Length Decoder)**
Scans the raw byte stream to find instruction boundaries. x86 instructions
are variable length (1–15 bytes), so this step must happen before decoding.
Outputs up to 6 instruction boundaries per cycle.

**Instruction Queue (IQ)**
Small buffer between pre-decode and the decoders. Absorbs bubbles caused by
pre-decode throughput variations.

**Decode (MITE — Micro-instruction Translation Engine)**
The legacy decode path. Converts x86 instructions into uops. Has 4 decoders
on Skylake (1 complex decoder handles up to 4 uops per instruction, 3 simple
decoders handle 1 uop each). Active only on DSB misses.

**DSB (Decoded Stream Buffer)**
Caches already-decoded uops indexed by instruction address. On a hit, uops
flow directly to the IDQ at up to 6 uops/cycle, bypassing pre-decode and
decode entirely. Capacity: 1536 uops (32 sets × 8 ways × 6 uops per way).

**LSD (Loop Stream Detector)**
Detects small loops that fit entirely within the IDQ (~25 uops). Locks them
in the IDQ and replays them without going back to the DSB or decode. Zero
frontend overhead — the best possible case for tight loops.

**IDQ (Instruction Decode Queue)**
64-entry buffer between the frontend and backend. Acts as the handoff point.
The backend pulls up to 4 uops/cycle from the IDQ. Frontend stalls show up
as IDQ empty cycles.

---

## Parameters & Perf Events

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
| Decode      | Legacy decode uops     | uops coming from slow decode path (not DSB)      | `idq.mite_uops`                         |
| Decode      | Legacy decode cycles   | Cycles the legacy decoder was active             | `idq.mite_cycles`                       |
| DSB         | DSB uops               | uops delivered from DSB (fast path)              | `idq.dsb_uops`                          |
| DSB         | DSB active cycles      | Cycles DSB was delivering uops                   | `idq.dsb_cycles`                        |
| DSB         | DSB → MITE switches    | Penalty cycles when falling back from DSB        | `dsb2mite_switches.penalty_cycles`      |
| LSD         | LSD uops               | uops delivered by the loop stream detector       | `lsd.uops`                              |
| LSD         | LSD active cycles      | Cycles the LSD was active                        | `lsd.cycles_active`                     |
| LSD         | LSD not active         | Cycles LSD could have been used but wasn't       | `lsd.not_active`                        |
| IDQ         | IDQ empty cycles       | Frontend stall — IDQ ran dry                     | `idq.empty`                             |
| IDQ         | Uops not delivered     | uops the frontend failed to deliver to backend   | `idq_uops_not_delivered.core`           |
| IDQ         | IDQ full cycles        | Backend stall — IDQ was full                     | `idq.full`                              |
| Frontend    | Frontend bound         | % of cycles frontend was the bottleneck          | `topdown-fetch-bubbles` / TMA level 1   |
