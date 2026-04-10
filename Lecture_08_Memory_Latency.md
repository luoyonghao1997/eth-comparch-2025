# Lecture 8: Memory Latency — Sources, Inefficiencies, and Architectural Solutions


> Memory latency has barely improved over two decades despite massive capacity and bandwidth gains, because manufacturers optimize for cost-per-bit, not speed. This lecture identifies two root causes of this stagnation: DRAM is not architecturally designed for low latency, and its timing parameters are set conservatively for worst-case conditions that rarely occur. It then presents four architectural solutions — **TLDRAM**, **SALP**, **CROW**, and **ALDM** — that each exploit a different form of heterogeneity within or across DRAM chips to reduce latency substantially, without requiring a full redesign of the memory system.


---
### 1. 📊 Why Memory Latency Is a Persistent Problem

> Despite decades of caching, prefetching, and out-of-order execution, memory latency has not fundamentally improved. These techniques hide latency at great hardware cost and complexity — they do not reduce it. Latency is now the primary bottleneck for an increasingly important class of workloads with irregular, low-reuse memory access patterns.

* **The Fundamental Distinction: Hiding vs. Reducing Latency**
  * All mainstream techniques — caches, prefetchers, out-of-order execution, multi-threading — attempt to **hide** latency by overlapping it with useful work.
  * None of them **reduce** the underlying memory access time.
  * Hiding latency requires significant hardware investment:
    * Large, multi-level cache hierarchies (area, power, complexity).
    * Out-of-order execution engines (reorder buffers, reservation stations).
    * Thread-level parallelism (more cores, more context switching).
  * The hidden latency still consumes time and energy — it is simply overlapped with other work when possible.
  * For workloads with **irregular access patterns and low arithmetic intensity** (e.g., graph analytics, pointer chasing, genome analysis), hiding fails because there is insufficient parallelism to overlap.

* **The Stagnation of DRAM Latency**
  * DRAM capacity and bandwidth have improved dramatically; latency has not:

    | Metric | Improvement (1999–2017) |
    |---|---|
    | Capacity | ~128× |
    | Bandwidth | ~20× |
    | Latency (tRCD, tRAS) | ~1.3× (nearly flat; some parameters worsened) |

  * Specific parameter trends:
    * **Restoration latency (tRAS)**: reduced by ~27% over 16 years — marginal.
    * **Activation latency (tRCD)** and **pre-charge latency (tRP)**: actually increased at some generations.
  * Root cause: manufacturers optimize for **cost-per-bit** (capacity), not latency. Reducing latency does not directly improve yield or reduce fabrication cost.

* **Workloads That Cannot Hide Latency**
  * **Graph analytics**: traversal requires reading a node to discover the address of the next node (**pointer chasing**). Each memory access is dependent on the previous result → cannot be parallelized or prefetched.
    * GPU cores are often **<5% utilized** running graph workloads — not because of compute shortage, but because all cores are stalling on memory.
  * **Database queries with random access patterns**: irregular access defeats cache spatial and temporal locality.
  * **Genome analysis**: pointer-based data structures with unpredictable access sequences.
  * For all of these, reducing DRAM latency directly translates to application speedup — no architectural trick can substitute.

* **New DRAM Types Often Increase Latency**
  * A study of 9 modern DRAM types found:
    * **DDR4** has higher latency than DDR3 (though higher bandwidth).
    * **GDDR5/6** and **HBM** prioritize bandwidth for GPU SIMD pipelines — latency is a secondary concern.
    * These designs assume that latency can be tolerated via massive thread-level parallelism in GPUs.
  * For non-GPU, non-parallelizable workloads, these new DRAM types are worse, not better.

* **Systemic Effects of High Memory Latency**
  * **Multi-core contention**: with many cores issuing concurrent requests, the memory request queue fills up. Higher per-request latency → longer queue depth → more blocking → cascade of performance degradation.
  * **QoS and predictability**: latency-sensitive CPU tasks suffer when mixed with bandwidth-intensive GPU tasks on a shared memory bus.
  * **Processor design complexity**: out-of-order cores, large reorder buffers, and speculative execution engines exist primarily to tolerate memory latency — these would be unnecessary if latency were low.

* **Run-Ahead Execution: The State-of-the-Art Latency Tolerance Technique**
  * Developed by Onur Mutlu — one of the most effective known latency tolerance techniques:
    * When the oldest instruction in the pipeline is a long-latency cache miss, instead of stalling, the processor enters **run-ahead mode**.
    * In run-ahead mode, the processor continues executing younger instructions **speculatively** (results not committed).
    * These speculative executions generate future memory addresses → prefetch requests are issued for Load 2, Load 3, etc.
    * When Load 1's data finally arrives, the pipeline is flushed and re-executed from a checkpoint — but Load 2's data is likely already in cache.
    * Net effect: latency of Load 1 is used productively to prefetch future data.
  * Real-world impact: implemented in **Nvidia Denver** processors; related technique **"Scout"** at Sun Microsystems achieved equivalent performance with 7 MB less L2 cache.
  * Won the **ISCA Test of Time Award**.
  * **Limitation**: only effective when future access addresses can be computed from speculatively executed instructions — fails for purely data-dependent pointer chasing.


---
### 2. 🔬 Two Root Causes of DRAM Latency Inefficiency

> DRAM latency is high for two independent reasons that compound each other: the memory architecture itself was not designed with latency as a goal, and the timing parameters that govern every DRAM access are set conservatively for worst-case conditions that rarely occur in practice.

* **Root Cause 1: DRAM Architecture Is Not Optimized for Low Latency**
  * DRAM design priorities, in order: **cost per bit** (capacity) → reliability → bandwidth → latency.
  * Architectural decisions that maximize density inherently increase latency:
    * **Long bit lines**: amortize the cost of large sense amplifiers across many cells. Longer bit lines → higher bit-line capacitance → slower charge sharing → higher activation latency.
    * **Shared row decoders across subarrays**: reduces area but serializes access to different subarrays within the same bank.
    * **Limited subarray-level parallelism**: multiple subarrays share global structures, preventing concurrent accesses.

* **Root Cause 2: Timing Parameters Are Set for Worst-Case Conditions**
  * Every DRAM timing parameter (tRCD, tRAS, tRP, tRC, ...) is set to guarantee correct operation under:
    * **Worst-case temperature**: 85°C (JEDEC standard).
    * **Worst-case chip**: the weakest cell across all chips in a production batch.
    * **Worst-case voltage**: minimum supply voltage.
    * **Worst-case data pattern**: most challenging bit pattern for charge sharing and sensing.
  * Real operating conditions are far from worst-case:
    * Server DRAM typically operates at **27–34°C** (active cooling).
    * Desktop DRAM operates at **~50°C**.
    * Standard is written for **85°C** → massive timing slack exists in typical operation.
  * This conservative one-size-fits-all timing is identical to the retention problem: refresh rates are set for the weakest cell, timing parameters are set for the worst-case chip.
  * **Analogy**: a highway speed limit set for the worst-case car in the worst-case weather, applied uniformly on a sunny day with a modern vehicle.


---
### 3. 🏗️ DRAM Microarchitecture: Key Concepts

> Understanding DRAM latency solutions requires a clear model of how a DRAM bank is structured and how each access command maps to physical operations on cells, bit lines, and sense amplifiers.

* **Physical Structure of a DRAM Chip**
  * A DRAM module contains multiple **DRAM chips** operating in parallel (typically 8 chips per module for 64-bit data width).
  * Each chip contains multiple **banks** (typically 8–16).
  * Each bank contains multiple **subarrays** — the fundamental access unit.
  * Each subarray contains:
    * A 2D array of **DRAM cells** (capacitor + access transistor).
    * A **bit line** per column, connecting all cells in a column to one sense amplifier.
    * A **sense amplifier (row buffer)** per bit line — detects and amplifies the tiny voltage change caused by cell charge sharing.
    * A **word line** per row — activates all access transistors in a row simultaneously.
    * A **row decoder** — selects which word line to assert.

* **The Access Sequence: Activate → Read/Write → Pre-charge**
  * **Pre-charge (tRP)**: bit lines are equilibrated to VDD/2 (half supply voltage). This is the idle/ready state. Any previously open row must be closed first.
  * **Activate (tRCD)**: the selected word line is asserted → access transistors connect all cells in the row to their bit lines → charge sharing occurs → bit lines deviate slightly from VDD/2 → sense amplifiers are enabled and amplify the deviation to full VDD or 0.
    * Sensing is **destructive**: the cell loses charge during sharing → must be restored.
  * **Read/Write (CL)**: after activation, specific columns are selected via column address → data is transferred to/from the row buffer.
  * **Restoration (tRAS - tRCD)**: sense amplifiers drive the full VDD or 0 back through the access transistors → cell capacitors are recharged to their correct values.
  * **Pre-charge again (tRP)**: deassert word line → equilibrate bit lines for the next access.

* **Why Reading Is Destructive (and Why This Creates Latency)**
  * The bit line capacitor is much larger than the cell capacitor (necessary to wire up many cells).
  * When the access transistor opens, charge sharing causes the cell to lose most of its charge to the bit line.
  * After sensing, the sense amplifier must restore the cell — this restoration time (tRAS) is a major component of DRAM latency.
  * Faster cells (larger capacitance, more stored charge) create a larger initial voltage perturbation → sense amplifier converges faster → lower latency. This is the physical basis for all latency heterogeneity exploited by the techniques below.


---
### 4. ⚡ Technique 1: Tier-Latency DRAM (TLDRAM)

> TLDRAM exploits the trade-off between bit line length and access latency by creating two distinct speed tiers within the same physical bank. A small, fast "near segment" (short bit lines) serves as a hardware cache for the larger, slower "far segment" (long bit lines), achieving average-case latency significantly below the commodity baseline.

* **The Bit Line Length vs. Latency Trade-off**
  * In DRAM, sense amplifiers are expensive (large area) — they are amortized by connecting many cells to one sense amplifier via a long bit line.
  * **Longer bit lines**: lower area per bit, lower cost — but higher bit-line capacitance → slower charge sharing → higher latency.
  * **Shorter bit lines**: lower latency — but higher area per bit (more sense amplifiers needed) → higher cost.
  * Quantified trade-off:
    * 512-row subarray (long bit lines): latency ~50 ns.
    * Chopped bit line for low latency: latency ~20 ns — but area increases ~4×.

* **TLDRAM Solution: Heterogeneous Segments via Isolation Transistors**
  * Add **isolation transistors** at fixed points along each bit line, dividing it into a **near segment** (few rows, short bit line, fast) and a **far segment** (many rows, long bit line, slower).
  * When accessing the near segment: close the isolation transistor → only the short portion of the bit line is active → low capacitance → fast access.
  * When accessing the far segment: open the isolation transistor → full bit line length engaged → slower access, plus slight extra latency from isolation transistor capacitance.

  | Segment | vs. Commodity DRAM |
  |---|---|
  | **Near segment latency** | **−56%** (from ~52 ns to ~23 ns) |
  | Far segment latency | +23% (isolation transistor adds capacitance) |
  | Near segment power | Lower |
  | Far segment power | Higher |
  | Area overhead | ~3% |

* **Using TLDRAM: Near Segment as a Cache**
  * The near segment is small → it must be managed as a cache for the far segment.
  * Management options:
    * **Hardware-managed inclusive cache**: near segment caches hot rows from the far segment transparently.
    * **Hardware-managed exclusive cache**: no duplication — a row is in either near or far, never both.
    * **OS-managed page mapping**: OS profiles access patterns and places hot pages in near-segment-mapped physical addresses.
  * Cache block size: 8 Kbit (one DRAM row) — much larger than the 64-byte CPU cache blocks. Replacement policies must be adapted (standard LRU works but must operate at row granularity).
  * Row migration between segments uses isolation transistors + row buffer (similar to RowClone intra-subarray copy).

* **Results**
  * Optimal near segment size: ~32 rows (balances hit rate vs. near segment's own latency).
  * Average performance improvement: **11–12%**; some workloads see **20–40%** improvement.
  * Power consumption reduced due to more accesses hitting the energy-efficient near segment.


---
### 5. 🔀 Technique 2: Subarray-Level Parallelism (SALP)

> In conventional DRAM, all subarrays within a bank share a single row decoder and a single global IO path — so only one subarray can be active at a time, even though each has its own local row buffer. SALP minimally modifies these shared structures to allow multiple subarrays to activate and serve data concurrently, nearly eliminating intra-bank conflict latency at negligible hardware cost.

* **The Bank Conflict Problem**
  * A DRAM bank contains many subarrays, each with its own local sense amplifiers (row buffer).
  * However, all subarrays share:
    * A **single global row decoder** (one address latch for the entire bank).
    * A **single global IO path** (only one local row buffer can connect to the external data pins at a time).
  * Consequence: even if requests target different subarrays, they cannot execute in parallel — one must wait for the other to complete its full activate–read–pre-charge cycle.
  * This is a **bank conflict** — one of the most common causes of DRAM latency inflation in real workloads.

* **SALP's Minimal Modifications**
  * **Modification 1 — Per-subarray address latch**:
    * Replace the single global address latch with a **separate latch per subarray**.
    * Allows the addresses of multiple incoming requests to be latched simultaneously.
    * Multiple subarrays can begin their activation sequences in parallel.
  * **Modification 2 — Per-subarray global IO switch**:
    * Add a **small single-bit latch per subarray** that controls a switch connecting the subarray's local row buffer to the global IO path.
    * Only one switch is closed at a time for actual data transfer — but all subarrays can have data ready in their local row buffers simultaneously.
    * Switching between subarrays for data transfer is fast (nanoseconds) vs. a full pre-charge + activate cycle (tens of nanoseconds).

* **How SALP Reduces Latency**
  * Without SALP — two requests to different subarrays (rows A and B):
    ```
    Activate A → Read A → Pre-charge A → Activate B → Read B → Pre-charge B
    ```
  * With SALP — both subarrays activate simultaneously:
    ```
    Activate A + Activate B (parallel)
    → Read A (switch A's local buffer to global IO)
    → Read B (switch B's local buffer to global IO — microseconds later)
    → Pre-charge A + Pre-charge B (parallel)
    ```
  * The long pre-charge and activation latencies are **overlapped** → effective latency for the second access is reduced to just the switching time.

* **Results and Industry Context**
  * Hardware overhead: **0.15–2%** additional chip area.
  * Performance improvement: **~17%** — approaching the theoretical ideal (20%) of making every subarray a separate physical bank.
  * Energy reduction: significant, from fewer wasted activate/pre-charge cycles.
  * Samsung and Intel explored SALP to compensate for performance overhead introduced by increased write restoration latency (needed for reliability improvements in advanced nodes) — and found SALP could fully offset the overhead.
  * Despite manufacturer interest and negligible cost, SALP was not standardized due to JEDEC committee dynamics.


---
### 6. 🐦 Technique 3: CROW (Copy-Row DRAM)

> CROW adds a small set of dedicated "copy rows" with a separate fast-access decoder to each DRAM subarray. These copy rows serve triple duty: as a low-latency cache for hot data, as a remapping target for retention-weak rows (reducing refresh frequency), and as a defense mechanism against RowHammer. All three benefits come from one low-overhead architectural addition.

* **The CROW Architecture**
  * Conventional DRAM: all rows share a single, complex row decoder → uniform access latency.
  * CROW adds a **separate, simpler decoder** dedicated to a small set of designated **"copy rows"** within each subarray.
  * The simpler decoder bypasses the complexity of the main decoder → copy rows can be accessed faster.

* **Use Case 1: Low-Latency Cache**
  * Copy a frequently accessed "hot" row from regular DRAM storage into the CROW copy region.
  * When that row is accessed, activate **both** the copy row and the original row simultaneously via their separate decoders.
  * Both cells contribute charge to the bit line → **doubled current** → sense amplifier settles faster → **lower activation latency**.
  * The speed benefit comes from the constructive combination of two charge sources — no new technology required.

* **Use Case 2: Reduced Refresh Overhead**
  * Retention-weak rows (identified via profiling) are **remapped** into the CROW copy region.
  * CROW copy rows are assumed to have stronger, more uniform cells (higher charge retention due to simpler structure and less density pressure).
  * The weak original row's data is maintained in the copy row → original row refreshed at a longer interval → fewer total refreshes.

* **Use Case 3: RowHammer Defense**
  * (Details deferred to the memory robustness lecture.)
  * The ability to quickly activate copy rows provides a mechanism to protect victim rows adjacent to hammered rows.

* **Combined Results**
  * CROW cache + CROW refresh together: **~20% performance speedup**, **~22% DRAM energy reduction**.
  * Hardware overhead: modest — only a few extra rows per subarray plus the simplified decoder.


---
### 7. 🔧 Technique 4: CLR DRAM (Capacity-Latency Reconfigurable DRAM)

> CLR DRAM makes the fundamental capacity-latency trade-off in DRAM dynamically configurable at runtime — per row, not per chip. In high-performance mode, it stores both a value and its complement, enabling two cells to drive the bit line together and cutting activation latency by 60%. This trades 50% capacity for dramatically lower latency, on-demand.

* **The Capacity-Latency Trade-off in DRAM**
  * Low-latency DRAM requires shorter bit lines or more charge per cell — both reduce density.
  * A single DRAM chip cannot be both maximum capacity and minimum latency simultaneously.
  * **CLR DRAM** makes this trade-off **dynamically configurable at the row level**.

* **How CLR DRAM Works**
  * Based on an **open bit line architecture** (standard in modern high-density DRAM):
    * Bit lines connect to sense amplifiers from one side only.
    * The true (O) and complementary (Ō) bit lines of the sense amplifier face opposite sides of the cell array.
  * CLR adds two types of isolation transistors (Type 1 and Type 2):
    * **Max Capacity Mode** (Type 1 only active): identical to standard open bit line — full density, standard latency.
    * **High Performance Mode** (both Type 1 and Type 2 active):
      * The row stores both O (true value) and Ō (complement) in adjacent physical rows.
      * During activation, both O and Ō are connected to their respective sides of the sense amplifier.
      * The sense amplifier receives **twice the current** (from both O and Ō simultaneously) → converges to the correct value much faster.
      * This is analogous to how CROW uses two copies to amplify — but here the complement is used rather than a duplicate.

* **Results**

  | Timing Parameter | Reduction in High-Performance Mode |
  |---|---|
  | Activation latency (tRCD) | **−60%** |
  | Restoration latency (tRAS) | Significant reduction |
  | Pre-charge latency (tRP) | Significant reduction |
  | Write recovery latency | Improved |

  * System-level results: **~18–19% performance improvement**, **~30% energy reduction**, **66% DRAM refresh energy reduction** (faster cells retain charge better).
  * Cost: **50% capacity reduction** in high-performance mode (each logical row occupies two physical rows).
  * Runtime reconfigurability: the OS or memory controller can switch rows between modes based on access frequency — hot rows in high-performance mode, cold rows in max capacity mode.
  * Analogous to **SLC vs. MLC** reconfiguration in Flash memory — a concept well-understood in storage but newly applied to DRAM.


---
### 8. 🌡️ Technique 5: Adaptive-Latency DRAM (ALDM)

> ALDM exploits the fact that DRAM timing parameters are set for 85°C worst-case operation, but real systems routinely operate at 27–55°C. By profiling actual DRAM modules at different temperatures and applying correspondingly lower timing parameters at runtime, ALDM reduces latency by 17–55% with no hardware modifications — only changes to the memory controller and DRAM profiles.

* **The Temperature Slack Problem**
  * JEDEC DRAM timing standards assume operation at up to **85°C**.
  * Real operating temperatures:
    * **Servers with active cooling**: ~27–34°C.
    * **Desktop PCs**: ~50°C.
    * **Laptops**: ~55–65°C under load.
  * At lower temperatures, DRAM cells leak less charge → more residual charge during sensing → sense amplifier converges faster → lower latency is achievable.
  * The timing slack between actual operating temperature and worst-case standard is enormous — and entirely wasted under fixed parameters.

* **Additional Sources of Latency Heterogeneity**
  * **Process variation**: cells vary in capacitor size, transistor characteristics, and contact resistance across a chip and across chips.
    * Larger cells: more stored charge → faster charge sharing → lower latency.
    * Smaller cells: less charge → slower sensing → higher latency.
  * **Voltage variation**: higher supply voltage → sense amplifiers converge faster.
  * **Manufacturer conservatism**: a single standard latency is chosen to guarantee all chips work, including the slowest chip at worst-case temperature and voltage — artificially slowing all faster chips.

* **Experimental Evidence (115 Real DRAM Modules)**
  * Using FPGA-based infrastructure (SoftMC / DRAM Bender), 115 DRAM modules were tested at reduced timing parameters:

    | Parameter | Average Reduction Without Errors |
    |---|---|
    | **tRCD** (sensing latency) | **−17%** (some chips: −40–50%) |
    | **tRAS** (read restoration) | **−37%** |
    | **tWR** (write restoration) | **−54%** |

* **ALDM Mechanism**
  * **Offline profiling** (by manufacturer or at deployment time):
    * Test each DRAM module at multiple temperatures to find the minimum reliable timing parameters.
    * Store per-module, per-temperature timing profiles (e.g., timings for 20–40°C, 40–60°C, 60–85°C).
  * **Online adaptation** (at runtime):
    * Monitor DRAM module temperature (via on-DIMM thermal sensors, already present in standard DIMMs).
    * Select the appropriate timing profile for the current temperature.
    * Apply reduced timing parameters to the memory controller (BIOS/firmware setting).
  * No DRAM hardware modifications required — only memory controller parameter changes.

* **Results**

  | Configuration | Latency Reduction at 55°C |
  |---|---|
  | Read latency | **−33%** |
  | Write latency | **−55%** |
  | Sensing (tRCD) | −17% |
  | Read restoration (tRAS) | −37% |
  | Write restoration (tWR) | −55% |
  | Pre-charge (tRP) | −35% |

  * **System performance improvement**:
    * Single-core, memory-non-intensive: ~1.4% average.
    * Single-core, memory-intensive: ~6.7% average.
    * **Multi-core systems**: significantly higher — reduced latency shortens queue depths and reduces blocking time, amplifying per-request latency improvements into large system throughput gains.

* **Why Multi-Core Benefits More**
  * In a multi-core system, many cores issue concurrent memory requests → request queues form.
  * Lower per-request latency → requests complete faster → shorter queues → less blocking of subsequent requests → multiplicative system-level benefit.
  * ALDM's advantage grows with the number of competing cores — making it increasingly valuable as systems scale.

* **Comparison of All Five Latency Reduction Techniques**

  | Technique | Key Mechanism | Primary Benefit | Hardware Cost |
  |---|---|---|---|
  | **TLDRAM** | Two speed tiers via isolation transistors | Near segment: −56% latency | ~3% area overhead |
  | **SALP** | Per-subarray latches + IO switches | Parallel subarray activation | ~0.15–2% area overhead |
  | **CROW** | Copy rows with fast dedicated decoder | −latency via dual-drive amplification | Few extra rows per subarray |
  | **CLR DRAM** | Store value + complement for dual-drive | −60% activation latency | 50% capacity in fast mode |
  | **ALDM** | Temperature-adaptive timing parameters | −17% to −55% various latencies | None (firmware only) |
