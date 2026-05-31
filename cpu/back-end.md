# CPU Backend Block Diagram

A common mental model for the backend (the "execution engine") of a modern
out-of-order CPU. The backend receives uops from the frontend's IDQ, renames
them, schedules them out-of-order onto execution ports, and retires them in
program order. Block layout is broadly the same across Intel, AMD, and ARM;
names, port counts, and buffer sizes differ.

```
                          CPU BACKEND
        (uops from frontend IDQ)
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │            Allocator / Rename (RAT)                 │  │
│   │   - maps arch regs → physical regs (PRF)            │  │
│   │   - zero idioms / move elimination resolved here    │  │
│   │   - allocates ROB + scheduler entries               │  │
│   └──────────────────────────┬──────────────────────────┘  │
│                              │                              │
│        ┌─────────────────────┼─────────────────────┐       │
│        ▼                     ▼                     ▼       │
│  ┌───────────┐        ┌─────────────┐       ┌───────────┐  │
│  │    ROB    │        │  Scheduler  │       │  PRF      │  │
│  │ (ReOrder  │        │ (Reservation│       │ (Physical │  │
│  │  Buffer)  │        │  Station /  │       │  Register │  │
│  │ in-order  │        │  issue queue)│      │  File)    │  │
│  │ retire    │        └──────┬──────┘       └───────────┘  │
│  └─────┬─────┘               │ ready uops issued           │
│        │            ┌────────┴───────────────────┐         │
│        │            ▼ (dispatch to ports)         │         │
│        │   ┌──────────────────────────────────┐   │         │
│        │   │          Execution Ports         │   │         │
│        │   │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ │   │         │
│        │   │  │ ALU │ │ ALU │ │ FP/ │ │ FP/ │ │   │         │
│        │   │  │     │ │     │ │ VEC │ │ VEC │ │   │         │
│        │   │  └─────┘ └─────┘ └─────┘ └─────┘ │   │         │
│        │   │  ┌─────┐ ┌─────┐ ┌──────────────┐│   │         │
│        │   │  │ AGU │ │ AGU │ │ Branch / etc ││   │         │
│        │   │  └──┬──┘ └──┬──┘ └──────────────┘│   │         │
│        │   └─────┼───────┼────────────────────┘   │         │
│        │         │       │  addresses             │         │
│        │   ┌─────▼───────▼──────────────────┐     │         │
│        │   │   Load / Store Unit (LSU)      │     │         │
│        │   │   ┌──────────┐  ┌───────────┐  │     │         │
│        │   │   │ Load Buf │  │ Store Buf │  │     │         │
│        │   │   └──────────┘  └───────────┘  │     │         │
│        │   │        L1 DCache  +  DTLB      │     │         │
│        │   └────────────────┬───────────────┘     │         │
│        │                    │ results (writeback)  │         │
│        │  ◄─────────────────┴──────────────────────┘         │
│        │  (completed uops marked done in ROB)                │
│        ▼                                                     │
│   in-order RETIRE  → commit to architectural state           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                  │ on miss / fault
                  ▼  (to memory hierarchy: L2 → L3 → DRAM)
```

**Out-of-order vs in-order, the two key invariants:**
- **Execution is out-of-order:** the scheduler issues any uop whose inputs are
  ready and whose port is free, regardless of program order. This is what hides
  latency — independent work runs while older work stalls.
- **Retirement is in-order:** the ROB commits uops strictly in program order.
  This preserves precise architectural state for interrupts/exceptions and is
  why a stalled instruction at the ROB head blocks everything behind it.

---

## Typical Values Across Architectures

Rough figures to build intuition — exact numbers vary by core revision.

| Block / metric          | Intel Skylake | AMD Zen 3      | Apple M1 (Firestorm) |
|-------------------------|---------------|----------------|----------------------|
| ROB size                | 224           | 256            | ~630                 |
| Scheduler entries       | 97 (unified)  | ~160 (split)   | very large           |
| Physical int registers  | 180           | 192            | ~380                 |
| Physical vec registers  | 168           | 160            | ~430                 |
| Execution ports         | 8             | (split EUs)    | ~?                   |
| Integer ALUs            | 4             | 4              | 6                    |
| FP/SIMD pipes           | 2–3 (FMA × 2) | 4              | 4                    |
| AGUs (load/store addr)  | 2 load, 1 sto | 3              | ~?                   |
| Load buffer entries     | 72            | 116            | ~130                 |
| Store buffer entries    | 56            | 64             | ~60                  |
| Retire width            | 4 uops/cycle  | 8 uops/cycle   | 8 uops/cycle         |

Takeaways:
- **ROB / window size** sets how far ahead the CPU can look for independent work
  to hide a stall. Apple M1's ~630-entry ROB is its signature — it hides memory
  latency by finding far more independent work than x86 cores.
- **Port count and EU mix** set peak compute throughput. Two FMA units (Intel)
  means 2 × 256-bit FMAs/cycle; this is the ceiling our dot-product hit.
- **Load/store buffers** bound how many memory ops can be in flight — the real
  limit for memory-level parallelism (MLP).

---

## Block Descriptions

**Allocator / Rename (RAT — Register Alias Table)**
Maps architectural registers to physical registers, breaking false (WAR/WAW)
dependencies. Zero idioms (`xor reg,reg`) and move elimination are resolved
here without using an execution port. Allocates a ROB entry and a scheduler
slot for each uop; stalls if either is full.

**ROB (ReOrder Buffer)**
Holds every in-flight uop in program order from allocation until retirement.
Tracks completion. Retires (commits) from the head, in order. A long-latency
uop at the head blocks all retirement behind it; when full, allocation stalls.

**Scheduler (Reservation Station / Issue Queue)**
Holds uops waiting for their operands. Each cycle it picks ready uops (inputs
available, port free) and issues them out-of-order to execution ports. May be
unified (Intel) or split per port group (AMD).

**PRF (Physical Register File)**
The actual storage for register values — far larger than the architectural
register count. Renaming hands out PRF entries; they're freed at retirement.

**Execution Ports / Units**
Each port hosts one or more functional units: integer ALUs, FP/SIMD (mul, add,
FMA), branch, AGUs. A uop is issued to a port that can execute it. Port
contention is a throughput limit (this is `Block RThroughput` in llvm-mca).

**AGU (Address Generation Unit)**
Computes effective addresses (`base + index*scale + disp`) for loads, stores,
and `lea`. Separate from the ALUs so address math runs in parallel with compute.

**Load/Store Unit (LSU)**
Executes memory ops. The **load buffer** and **store buffer** track in-flight
loads/stores, enforce memory ordering, and do store-to-load forwarding.
Contains the L1 DCache and DTLB. Misses go to L2 → L3 → DRAM.

**Retire**
Commits completed uops to architectural state in program order, frees their
ROB and PRF entries. Retire width caps useful IPC.

---

## Parameters & Perf Events

Perf event names below are Intel-specific (use `perf list`; AMD/ARM differ).

| Block        | Parameter                | Description                                       | Perf Event                                  |
|--------------|--------------------------|---------------------------------------------------|---------------------------------------------|
| Overall      | IPC                      | Retired instructions per cycle                    | `instructions` / `cycles`                   |
| Rename       | Rename stalls            | Cycles allocation stalled (no PRF/ROB/RS)         | `resource_stalls.any`                       |
| ROB          | ROB full stalls          | Cycles allocation stalled because ROB was full    | `resource_stalls.rob`                       |
| Scheduler    | RS full stalls           | Cycles stalled because reservation station full   | `resource_stalls.rs`                        |
| Ports        | Port utilization         | Cycles each execution port was busy               | `uops_dispatched_port.port_0` … `port_7`    |
| Exec         | Total uops executed      | uops sent to execution units                      | `uops_executed.thread`                      |
| Exec         | Cycles ≥1 uop executed   | Cycles the backend did any work                   | `uops_executed.cycles_ge_1`                 |
| Load/Store   | L1 D miss                | Loads missing L1 data cache                       | `mem_load_retired.l1_miss`                  |
| Load/Store   | L2 miss                  | Loads missing L2                                  | `mem_load_retired.l2_miss`                  |
| Load/Store   | L3 miss (→ DRAM)         | Loads missing LLC, served by DRAM                 | `mem_load_retired.l3_miss`                  |
| DTLB         | DTLB miss + walk         | Data address translation misses / page walks      | `dTLB-load-misses`                          |
| LSU          | Store buffer full        | Stalls from store buffer pressure                 | `resource_stalls.sb`                        |
| Mem stalls   | Stalled on memory        | Cycles backend stalled waiting on any memory      | `cycle_activity.stalls_mem_any`             |
| Backend      | Backend bound            | % of cycles backend was the bottleneck            | `topdown-be-bound` / TMA level 1            |
| Backend      | Memory bound (sub-split) | Backend stalls attributable to the memory system  | TMA: Backend Bound → Memory Bound           |
| Retire       | Retire slots used        | uop slots actually retired (useful work)          | `uops_retired.retire_slots`                 |
| Speculation  | Bad speculation          | % of slots wasted on mispredicted paths           | `topdown-bad-spec` / TMA level 1            |

---

## Quick Triage (Top-down / TMA)

When a workload is slow, classify each cycle into one of four buckets first,
then drill down:

```
            ┌──────────────────────────────┐
            │      Every issue slot        │
            └──────────────┬───────────────┘
        ┌──────────┬───────┴────────┬─────────────┐
        ▼          ▼                ▼             ▼
    Retiring   Frontend Bound   Bad Spec     Backend Bound
   (good work) (fetch/decode)  (mispredict)  (exec/memory)
                                                  │
                                       ┌──────────┴──────────┐
                                       ▼                     ▼
                                  Core Bound            Memory Bound
                              (port/exec limited)   (cache/DRAM stalls)
```

```bash
perf stat --topdown ./program          # level 1 buckets
perf stat -e cycles,instructions ./program   # IPC sanity check
```

- **High Retiring** → already efficient; reduce instruction count to go faster.
- **Backend → Memory Bound** → cache misses / bandwidth (our naive dot product).
- **Backend → Core Bound** → port/dependency limited (the FMA chain experiments).
- **Bad Speculation** → branch mispredicts; improve predictability.
- **Frontend Bound** → see [front-end.md](front-end.md).
