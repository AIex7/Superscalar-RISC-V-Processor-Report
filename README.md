# Apex-RV: Out-of-Order Superscalar RISC-V Processor

[![Language](https://img.shields.io/badge/Language-SystemVerilog-blue.svg)](https://en.wikipedia.org/wiki/SystemVerilog)
[![ISA](https://img.shields.io/badge/ISA-RISC--V%20(RV32IMC)-orange.svg)](https://riscv.org/)
[![Target](https://img.shields.io/badge/Target-FPGA%20Synthesis-green.svg)](#fpga-synthesis--physical-design)
[![Timing](https://img.shields.io/badge/Clock-100%20MHz%20(Slack%20%2B1.238ns)-brightgreen.svg)](#timing--slack-closure)
[![Architecture](https://img.shields.io/badge/Microarch-Tomasulo%20%2B%20ROB%20%2B%20RAT-purple.svg)](#microarchitecture-overview)

Apex-RV is a synthesizable, high-performance, 32-bit out-of-order (OoO) speculative RISC-V processor core designed and implemented from scratch in SystemVerilog for FPGA deployment. 

Developed as a collaborative team engineering project, the processor implements an out-of-order execution engine based on **Tomasulo's Algorithm** with dynamic register renaming via a **Register Alias Table (RAT)**, in-order retirement via a **Reorder Buffer (ROB)**, a multi-entry **Common Data Bus (CDB)**, a hybrid **Tournament Branch Predictor**, an **L1 Next-Line Instruction Cache Prefetcher**, a **Split Load-Store Queue (Split LSQ)**, and a **Post-Commit Store Buffer** with store-to-load forwarding.

Across standard industry and systems benchmarks (`coremark`, `aes_sha`, `compression`, `fft`, `mergesort`), our microarchitectural optimizations achieved a **>2x IPC uplift** (up to **+162.5%** on memory/branch-heavy workloads) while meeting all timing constraints at **100 MHz** with **+1.238 ns** positive slack on the FPGA target.

---

## Key Architectural Highlights

- **Dynamic Speculative Out-of-Order Execution**:
  - Reorder Buffer (ROB) supporting precise exceptions and in-order retirement.
  - Dual Register File scheme: Architectural Register File (ARF) and speculative Register Alias Table (RAT) for zero-stall register renaming (eliminating WAR and WAW hazards).
  - Multi-station Common Data Bus (CDB) broadcasting tag/data pairs directly to waiting reservation stations and the ROB.
- **Advanced Branch Handling**:
  - Early branch decode stage operating directly off the instruction line buffer.
  - Hybrid Tournament Predictor coupling a global-history **GSelect** predictor and a local **Two-Level Adaptive** predictor managed by a 4-bit saturating counter meta-predictor.
  - Slashes branch misprediction flushes by **>51.4%**.
- **High-Throughput Memory Subsystem**:
  - Split L1 Caches: Non-blocking L1 Instruction Cache with hardware next-line prefetcher and L1 Data Cache.
  - Burst Memory Interface adapter (`bmem_*`) bridging multi-word DRAM bursts to single-cycle cache lines.
  - Dedicated circular **Post-Commit Store Buffer** with address merging and instantaneous store-to-load forwarding.
  - **Split LSQ** with separate Load and Store queues for age-prioritized memory disambiguation.
- **FPGA Timing Optimized**:
  - 100 MHz target operating frequency cleanly closed with **1.24 ns** slack.
  - Optimized age-priority queue issue logic to truncate critical combinational paths.
  - Decoupled reservation station dispatch from functional unit execution to maximize IPC.

---

## Microarchitecture Overview

### Top-Level Pipeline Architecture

The complete processor datapath and control structure is organized into decoupled pipeline stages connected via latency-insensitive handshakes and FIFO queues.

![Pipeline Architecture](docs/images/pipeline_architecture.png)

1. **Memory Interface**:
   - Connects to the system memory via a 64-bit/burst DRAM interface (`bmem_*`).
   - A hardware `cache_line_adapter` serializes/deserializes burst data packets to serve 256-bit cache lines to the L1 caches.
2. **Caches**:
   - **Instruction Cache**: Directly interfaces with the speculative next-line prefetcher to keep the pipeline fed.
   - **Data Cache**: Interfaces with the memory execution unit and the post-commit store buffer.
3. **Fetch & Buffer**:
   - `inst_line_buffer` caches incoming cache lines.
   - Pushes 32-bit instructions into a parameterized `instruction_queue` FIFO to decouple front-end fetch latency from decode.
4. **Dual-Stage Decode**:
   - **Early Decode Stage (`decode_stage_early`)**: Sits immediately behind the line buffer to extract branch target offsets and opcode hints early, driving the `tournament_predictor` without stalling.
   - **Standard Decode Stage (`decode_stage`)**: Pulls instructions from the queue, decodes registers, control words, and immediate fields, and feeds the dispatch logic.
5. **Predict & Dispatch (`dispatch_issue`)**:
   - Queries the RAT to obtain physical operand tags or current ARF values.
   - Allocates the next sequential entry in the ROB.
   - Dispatches instructions and operands/tags to the appropriate reservation station.
6. **Out-of-Order Execution**:
   - Multiple specialized functional units operating concurrently:
     - `alu_unit`: Single-cycle integer arithmetic, logic, and shifts.
     - `mul_unit`: Hardware multiplier pipeline.
     - `br_unit`: Branch condition verification and target address validation.
     - `mem_unit`: Effective address calculation and load/store queue interface.
7. **Common Data Bus & In-Order Commit**:
   - Completed functional units arbitrate for the `cdb` (Common Data Bus).
   - Results broadcast tag/value pairs to all listening reservation stations and the `rob_inst` (Reorder Buffer).
   - The ROB retires instructions strictly in program order, committing values to the `rat_arf` (ARF/RAT) and releasing store buffer entries to memory.

---

### Tomasulo Execution Engine

The core execution mechanics were originally modeled and iteratively refined from the classic Tomasulo algorithm with speculative Reorder Buffer extensions:

![Tomasulo Microarchitecture Plan](docs/images/tomasulo_diagram.png)

- **Register Renaming**: Read operands check the RAT. If an in-flight instruction produces the register, the RAT returns the corresponding ROB tag; otherwise, it returns the committed ARF value.
- **Snooping & Forwarding**: When a functional unit outputs a tag match on the CDB, reservation stations latch the value into their pending operand slots in the same cycle.
- **Decoupled Dispatch**: Instructions dispatch into available reservation station entries even if functional units are currently busy executing previous instructions, preventing front-end stalls.

---

## Advanced Features & Optimizations

### 1. Tournament Branch Predictor (GSelect + Two-Level)

Branch mispredictions are extraordinarily expensive in an out-of-order processor due to the penalty of flushing the ROB, RAT, reservation stations, and fetch buffers. To maximize prediction accuracy, we built a hybrid **Tournament Predictor**:

- **GSelect Predictor**: Correlates global branch history (PHT index derived from hashing branch PC bits with the global branch history register). Ideal for patterned sequences and nested control flows.
- **Two-Level Adaptive Predictor**: Utilizes local branch history tables (BHT) indexed by PC, referencing per-branch pattern history tables. Ideal for independent, loop-bound branches.
- **Chooser Meta-Predictor**: A 4-bit saturating counter array indexed by the branch address decides whether to use GSelect or Two-Level. When predictions disagree, the counter increments toward the winning predictor or decrements toward the other.

![Branch Predictor Performance](docs/images/branch_predictor_stats.png)

#### Real Hardware Impact (Measured on `coremark_im`):
| Branch Predictor Scheme | Total Branches Encountered | Pipeline Misprediction Flushes | Core IPC |
| :--- | :---: | :---: | :---: |
| **Static Not-Taken** | 111,945 | 34,821 | 0.335 |
| **Tournament Predictor** | 75,303 | **16,899** | **0.413** |
| **Net Improvement** | — | **-51.4% Flushes** | **+23.3% IPC** |

---

### 2. Next-Line Instruction Cache Prefetcher

Front-end starvation was one of our primary initial bottlenecks—small reservation station depths meant instructions had to arrive rapidly to keep execution units saturated.

We designed a dedicated **Next-Line Prefetcher** module interfacing directly between the L1 instruction cache and the memory line adapter. Whenever cache line $N$ is requested, the prefetcher speculatively requests line $N+1$ into the line buffer during available bus cycles.

![Prefetcher Performance](docs/images/prefetcher_stats.png)

#### Prefetcher Verification on `coremark_im`:
| Metric | Without Prefetcher | With Prefetcher | Impact |
| :--- | :---: | :---: | :---: |
| **Cycles in Compare State** | 102,586 | 203,157 | +98.0% |
| **Hits in Compare State** | 99,493 | 199,869 | +100.9% |
| **L1 Instruction Cache Hit Rate** | **96.9%** | **98.3%** | **+1.4% Hit Rate (Near-Zero Stall)** |

---

### 3. Post-Commit Store Buffer & Store-to-Load Forwarding

In an out-of-order processor, memory stores cannot write to the cache or DRAM speculatively—they must wait until commit to preserve precise architectural state. However, stalling the ROB until a store finishes a multi-cycle DRAM write destroys performance.

We implemented a **Post-Commit Store Buffer**:
- Formed as an asynchronous circular ring buffer.
- Stores retire from the ROB into the Store Buffer immediately, freeing the ROB entry for younger instructions.
- **Store-to-Load Forwarding**: When an in-flight load executes, it snoops the Store Buffer in parallel with the L1 cache. If an unwritten store matches the load address, data is forwarded directly with zero memory stall cycles.
- **Address Merging**: Consecutive stores writing to the same cache block merge into a single transaction, minimizing memory bus contention.

---

### 4. Split Load-Store Queue (Split LSQ)

Rather than maintaining a monolithic, complex associative LSQ, we split memory operations into separate **Load Reservation Stations** and **Store Reservation Stations**:
- Eliminates head-of-line blocking where pending stores would unnecessarily stall independent loads.
- Enforces memory disambiguation by tracking program order: loads only execute when all older stores with matching or unknown addresses have resolved their effective addresses.
- Significantly simplifies issue logic and cuts arbitration delay by half.

---

### 5. Age-Ordered Issue Scheduling

In our baseline design, issuing the oldest ready instruction required an $O(N^2)$ all-to-all age matrix comparison across every active reservation station. This dominated our critical path and capped our clock speed.

To close timing:
- We overhauled the scheduling logic into an **Age-Ordered Queue Structure**.
- The head of each reservation station group tracks the most likely oldest ready station.
- If the oldest station is waiting on an operand, the arbiter ripples down to the next ready station in the queue.
- This eliminated dozens of levels of combinational logic from the dispatch clock path while maintaining high IPC.

---

### 6. RISC-V Compressed Instruction Extension (RV32C)

To enhance code density and maximize instruction fetch bandwidth through our cache lines, we developed support for the RISC-V 'C' extension:
- Implemented a **secondary parallel decode module** operating side-by-side with the standard 32-bit decoder.
- Expands 16-bit compressed instructions into internal 32-bit canonical RV32I micro-ops in single-cycle combinational logic.
- Maintained on a dedicated synthesis branch to evaluate area vs. density trade-offs on FPGA slices.

---

## Benchmark Results & Performance Evaluation

We verified and profiled the processor across five standard benchmarks: `coremark`, `aes_sha`, `compression`, `fft`, and `mergesort`. Performance counters were integrated directly into the datapath to collect exact cycle counts, instruction retirements, cache hits, and pipeline flushes.

### Full Benchmark Suite Comparison

Comparing our initial baseline out-of-order processor against our final optimized architecture:

| Benchmark | Baseline IPC | Final Optimized IPC | IPC Uplift (%) | Baseline Delay (ps) | Final $DP^2$ |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **`coremark`** | 0.2102 | **0.4131** | <span style="color:green">**+96.5%**</span> | 13,870,820,000 | 37,283.25 |
| **`aes_sha`** | 0.1923 | **0.4077** | <span style="color:green">**+112.0%**</span> | 40,345,330,000 | 1,953,559.83 |
| **`compression`** | 0.2163 | **0.5678** | <span style="color:green">**+162.5%**</span> | 22,273,410,000 | 49,858.68 |
| **`fft`** | 0.1922 | **0.4251** | <span style="color:green">**+121.2%**</span> | 60,419,880,000 | 8,243,623.53 |
| **`mergesort`** | 0.2284 | **0.4221** | <span style="color:green">**+84.8%**</span> | 18,388,980,000 | 256,849.65 |

```
Benchmark IPC Comparison (Baseline vs Final)
========================================================================
coremark     [██████████ 0.210]  --->  [████████████████████ 0.413]   (+96.5%)
aes_sha      [█████████ 0.192]   --->  [████████████████████ 0.408]   (+112.0%)
compression  [██████████ 0.216]  --->  [████████████████████████████ 0.568] (+162.5%)
fft          [█████████ 0.192]   --->  [████████████████████ 0.425]   (+121.2%)
mergesort    [███████████ 0.228] --->  [████████████████████ 0.422]   (+84.8%)
========================================================================
```

### Key Takeaways:
1. **Dramatic IPC Doubling**: Across all workloads, IPC increased by **+84.8% to +162.5%**.
2. **Compression Benchmark Spike (0.5678 IPC)**: The compression workload benefited enormously from the combination of the Next-Line Prefetcher and the Post-Commit Store Buffer due to its high frequency of sequential memory accesses and pointer chasing.
3. **Low Delay-Product-Squared ($DP^2$)**: Demonstrates that high instruction throughput was achieved without ballooning energy-delay metrics.

---

## FPGA Synthesis & Physical Design

### Timing & Slack Closure

Synthesis and timing closure were conducted with tight design constraints at a target frequency of **100 MHz** (10.0 ns cycle period).

```text
================================================================================
Design Timing Report (Target: 100 MHz)
================================================================================
Clock period:              10.000000 ns
Data required time:         9.958930 ns
Data arrival time:         -8.720723 ns
--------------------------------------------------------------------------------
Slack (MET):               +1.238207 ns
Status:                    TIMING MET (Clean Positive Slack)
================================================================================
```

### Cell Area Breakdown

Our goal throughout development was to maximize microarchitectural performance while strictly policing hardware resource utilization and FPGA gate counts.

```text
================================================================================
Area Synthesis Report
================================================================================
Combinational Area:                 71,657.47  units
Non-combinational Area:             60,164.14  units
Macro / Black Box Area:             79,531.08  units
Buffer / Inverter Area:              5,322.66  units
--------------------------------------------------------------------------------
Total Cell Area:                   211,352.70  units
================================================================================
```

---

## Engineering Challenges & Design Iterations

Building an out-of-order core involves intricate race conditions and corner cases. Below are some of the critical engineering hurdles we encountered and resolved during development:

### 1. The "Reservation Station Depth" Paradox
- **Issue**: Due to FPGA slice area limitations, we had to reduce non-ALU reservation station depths from **8 entries down to 4 entries** (leaving the ALU station depth larger).
- **Surprising Outcome**: Counterintuitively, reducing depth **increased overall IPC**! 
- **Root Cause**: In our priority issue scheme, smaller queues reduced arbitration tree latency and increased the probability of dispatching the actual oldest ready instructions sooner, avoiding queue stall bubbles.

### 2. Eliminating Artificial Pipeline Stalls
- **Issue**: In our early prototypes, we introduced a conservative stall every other cycle to prevent multi-cycle functional units (like multipliers and memory loads) from being preempted.
- **Fix**: We rebuilt the functional unit issue logic with explicit valid/ready latency-insensitive handshakes, completely removing the artificial stall cycles and doubling instruction throughput.

### 3. Pipelining Combinational Logic in Clocked Blocks
- **Issue**: Early timing analysis revealed severe negative slack caused by massive combinational decoding trees nested directly within `always_ff` blocks.
- **Fix**: We refactored the front-end into distinct pipeline registers, isolating immediate decoding, control signal generation, and branch offset calculation into dedicated early decode stages.

### 4. Memory Bus Handshake Deadlocks
- **Issue**: An early bug caused the burst memory read request signal (`bmem_read`) to remain asserted across multiple consecutive cycles after DRAM grant, causing spurious reads and memory corruptions.
- **Fix**: Added a stateful edge-triggered handshake flag in the cache line adapter to enforce single-pulse assertion per cache miss.

---

## Repository Structure

```text
.
├── docs/
│   └── images/
│       ├── pipeline_architecture.png     # Full datapath & pipeline block diagram
│       ├── tomasulo_diagram.png          # Tomasulo execution engine microarchitecture
│       ├── branch_predictor_stats.png    # Branch predictor hardware comparison
│       └── prefetcher_stats.png          # L1 cache prefetcher hit rate statistics
├── hdl/
│   ├── cpu.sv                            # Top-level CPU core integration
│   ├── datapath/
│   │   ├── fetch/                        # Instruction fetch, line buffer, queue
│   │   ├── decode/                       # Dual decode stages (standard + early)
│   │   ├── dispatch/                     # RAT, reservation station allocation
│   │   ├── execute/                      # ALU, multiplier, branch, memory units
│   │   └── commit/                       # Reorder Buffer (ROB), ARF, CDB logic
│   ├── branch_prediction/
│   │   ├── tournament_predictor.sv       # Hybrid chooser meta-predictor
│   │   ├── gselect.sv                    # Global history predictor
│   │   └── two_level.sv                  # Local two-level adaptive predictor
│   └── memory/
│       ├── cache_line_adapter.sv         # DRAM burst deserializer
│       ├── prefetcher.sv                 # Next-line L1 instruction prefetcher
│       ├── l1_icache.sv                  # L1 instruction cache
│       ├── l1_dcache.sv                  # L1 data cache
│       └── store_buffer.sv               # Post-commit ring buffer with forwarding
├── sim/                                  # Testbenches and verification harness
└── synth/                                # Synthesis scripts, constraints & reports
```

---

## Team & Workflow

This processor was collaboratively designed, implemented, and verified by our engineering team:
- **Alex**: Parameterizable FIFO queues, DRAM cache line adapter, memory subsystem, and system verification.
- **Erin**: Pipeline datapath microarchitecture, instruction fetch pipeline, and architectural diagrams.
- **Sujin**: Datapath integration, early decode logic, instruction fetch buffers, and hardware verification.

### Collaborative Engineering Practices:
- **Pair Programming via VSCode Live Share**: Used live collaborative sessions to trace complex CDB arbitration bugs, register renaming edge cases, and resolve synthesis/linting warnings.
- **Modular Feature Branching**: Isolated high-risk microarchitectural features (such as RV32C compression and split LSQ) onto dedicated Git branches for independent simulation and FPGA resource profiling before merging into the main branch.
