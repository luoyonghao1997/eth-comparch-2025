# Lecture 7: Data Retention in Memory — DRAM Refresh and Flash Retention


> All charge-based memory technologies — DRAM and Flash alike — face a fundamental physical problem: stored charge leaks away over time, causing data corruption. In DRAM, this necessitates **periodic refresh** of every row, which at scale consumes up to 47% of DRAM energy and blocks nearly half of all memory accesses. This lecture examines why the standard universal refresh rate is a massive overengineered waste, explores a hierarchy of solutions from heterogeneous refresh rates (**RADAR**) to VRT-aware adaptive refresh (**Avatar**) to hidden parallel refresh (**HERO**), and extends the discussion to analogous retention challenges in Flash memory and SSD storage.


---
### 1. 🔬 The DRAM Refresh Problem

> DRAM cells store data as charge in leaky capacitors. Because charge drains away continuously, every row in the entire DRAM must be periodically refreshed or data is lost. As DRAM density scales up, the refresh problem compounds: more rows to refresh, smaller capacitors that drain faster, and higher temperatures that accelerate leakage — threatening to consume half of all DRAM energy and access time in future high-density chips.

* **How DRAM Cells Leak**
  * A DRAM cell stores one bit as charge on a **capacitor**, accessed via an **access transistor**.
  * The access transistor is not a perfect switch — it allows a small **leaky current** even when the word line is not activated.
  * This leakage continuously drains charge from the capacitor to the bit line.
  * Leakage is **worse at smaller technology nodes**:
    * Smaller transistors inherently have higher leakage current.
    * Smaller capacitors store less charge to begin with → less tolerance for leakage before a read error occurs.
  * Result: without intervention, a DRAM cell loses its data within milliseconds to seconds depending on temperature and cell characteristics.

* **The Refresh Mechanism and Its Cost**
  * The memory controller must **periodically activate every DRAM row** to restore capacitor charge before data is lost.
  * Standard refresh interval historically: **64 ms** (every row refreshed at least once every 64 ms).
  * Modern DRAM has reduced this to ~**32 ms** due to worsening leakage at smaller nodes.
  * Refresh operations incur three distinct costs:
    * **Energy**: each refresh consumes energy equivalent to a normal row activation — even on an otherwise idle chip.
    * **Performance**: the region of DRAM being refreshed is **inaccessible** for reads/writes during the refresh command.
    * **QoS and predictability**: if data needed by a running application is in a region currently being refreshed, latency spikes unpredictably — unacceptable for real-time workloads.

* **Quantifying Refresh Overhead at Scale (RADAR Analysis)**
  * The RADAR paper modeled refresh overhead as DRAM density scales from 4 GB to 64 GB:

    | DRAM Density | Refresh Time Overhead | Refresh Energy Overhead |
    |---|---|---|
    | 4 GB | ~8% | ~15% |
    | 16 GB | ~20% | ~28% |
    | 64 GB | **~46%** | **~47%** |

  * Two compounding factors drive this exponential worsening:
    * **More rows**: larger chips have proportionally more rows to refresh.
    * **Shorter intervals needed**: smaller cells at each new technology node require more frequent refresh.
  * A vicious cycle emerges: more refresh → more heat → faster leakage → even more frequent refresh needed.
  * **Conclusion**: without solving the refresh problem, DRAM scaling is fundamentally limited — a chip that spends half its time refreshing cannot serve as useful main memory.


---
### 2. 💡 Heterogeneous Refresh: The Core Insight

> The standard 64 ms universal refresh rate is a worst-case design for the absolute weakest cell in the chip. Experimental measurements show that the vast majority of DRAM cells can safely retain data for 256 ms or longer. Refreshing all cells at the worst-case rate is an enormous, unnecessary waste.

* **Manufacturing Variation Creates Heterogeneous Cell Strength**
  * No two DRAM cells are identical — **manufacturing process variation** produces a distribution of cell strengths:
    * A small fraction of cells are **"weak"**: can only retain data for ~64 ms or less.
    * The large majority are **"strong"**: can retain data safely for 256 ms, 512 ms, or even seconds.
  * Experimental measurements on real DRAM chips confirm this profile:
    * At 64 ms: **no errors observed** (all cells survive).
    * ~30 cells needed refreshing more frequently than 128 ms.
    * ~1,000 cells needed refreshing between 128–256 ms.
    * The overwhelming majority tolerated intervals far beyond 256 ms.
  * In a 32 GB system: only ~1,000 rows require 256 ms refresh — yet all billions of rows are refreshed every 64 ms.

* **The RADAR Solution: Profile, Bin, Refresh Heterogeneously**
  * **Core idea**: identify each row's actual retention time via profiling, then refresh each row only as often as it truly needs.
  * Two-step mechanism:
    * **Step 1 — Profiling**: determine the minimum safe refresh interval for every row in the chip.
    * **Step 2 — Binning**: categorize rows into bins (e.g., "needs 64 ms", "needs 128 ms", "needs 256 ms+") and store this classification efficiently.
  * The memory controller then applies **different refresh rates** to different rows based on their bin.

* **Efficient Binning with Bloom Filters**
  * Storing per-row refresh rates naively would require one entry per row — prohibitively expensive for billions of rows.
  * **Bloom filters** provide a compact probabilistic alternative:
    * A Bloom filter uses a **bit vector** and multiple **hash functions** to represent set membership.
    * To insert a row: hash its address with K hash functions → set K bits in the bit vector to 1.
    * To test if a row is in a set: hash its address → check if all K corresponding bits are 1.
    * If any bit is 0 → row is **definitely not** in the set (no false negatives).
    * If all bits are 1 → row is **probably** in the set (false positives possible due to hash collisions).
  * **Why false positives are acceptable here**:
    * A false positive means a "strong" row is incorrectly classified as "weak" → it gets refreshed more often than needed.
    * This wastes a small amount of energy but causes **no data loss** — correctness is preserved.
    * False negatives (missing a weak row) would cause data loss — and Bloom filters guarantee zero false negatives.
  * **RADAR storage overhead**: two Bloom filters for a 32 GB DRAM system → only **1.25 KB** total in the memory controller.

* **RADAR Results**
  * Refresh reduction: **~75%** fewer refresh operations.
  * Dynamic energy reduction: **~16%**.
  * Idle DRAM power reduction: **~20%**.
  * Performance improvement: **~9%**.
  * Benefits scale favorably with DRAM density — as the refresh problem worsens, RADAR's gains increase proportionally.
  * RADAR was selected for the **ISCA 25-year retrospective** as a landmark paper in memory systems research.


---
### 3. ⚠️ Profiling Challenges: DPD and VRT

> Profiling each DRAM cell's retention time sounds straightforward — write data, stop refreshing, measure when errors appear. In practice, two physical phenomena make accurate profiling extremely difficult: the retention time of a cell depends on the data pattern stored in surrounding cells (DPD), and the retention time of a single cell changes randomly over time (VRT).

* **Challenge 1: Data Pattern Dependence (DPD)**
  * A DRAM cell's retention time is not an intrinsic property — it depends on the **values stored in neighboring cells**.
  * Physical cause:
    * When a row is activated, all bit lines in the subarray are perturbed simultaneously.
    * Nearby cells couple electrically via **bit-line-to-bit-line coupling** and **bit-line-to-word-line coupling**.
    * The magnitude of this noise — which affects reliable sensing — depends on the pattern of charged/uncharged cells in the neighborhood.
  * Consequence: to find the **true worst-case retention time** of a cell, you must test it under every possible neighboring data pattern — an exponentially large search space.
  * Complications that make DPD even harder to handle:
    * The mapping from logical addresses to physical DRAM geometry (rows, columns, subarrays) is **not exposed** to the memory controller — it uses proprietary hash functions.
    * Internal **row/column remapping** (used to route around defective cells at manufacturing time) further obscures the true physical layout.
    * **Second-order coupling effects** (e.g., bit-line-to-word-line after first-order bit-line-to-bit-line coupling is reduced) are difficult to model and require circuit-level simulation.
  * Result: manufacturers themselves are conservative about worst-case refresh intervals because the true worst-case data pattern is hard to identify — justifying the universal 64 ms safety margin.

* **Challenge 2: Variable Retention Time (VRT)**
  * **VRT** is a random phenomenon where a cell's retention time **changes unpredictably over time** — sometimes by orders of magnitude.
  * A cell might safely retain data for 10 seconds today, then fail after 64 ms two days later, then recover to 5 seconds a week after that.
  * Physical cause: **charge traps** in the gate oxide of the access transistor.
    * A defect in the gate oxide can randomly capture or release a single electron.
    * When a trap is occupied → additional leakage current flows from the transistor's drain → capacitor discharges faster → shorter retention time.
    * This trapping/detrapping is a stochastic quantum-mechanical process → unpredictable timing and magnitude.
    * Known as **trap-assisted gate-induced drain leakage (GIDL)**.
  * Example: a cell from a 2 GB chip exhibited retention times ranging from ~2 seconds to ~7 seconds across repeated measurements — with no predictable pattern.
  * **Cold boot attacks** exploit VRT at the extreme:
    * Cooling DRAM modules with liquid nitrogen dramatically extends retention time from milliseconds to seconds.
    * An attacker who physically removes a powered-off DRAM module can recover its contents by reading before charge fully drains.
    * Mitigation: encrypt all data written to DRAM, or force a DRAM reset on physical removal.
  * **Key implication for profiling**: a cell that passes a profiling test today may exhibit VRT and fail next week. A one-time profile cannot guarantee future safety.

* **Implications: Profiling is Not a One-Time Operation**
  * VRT means profiling must be **repeated periodically** throughout the system's lifetime.
  * High temperatures during normal operation (and especially during chip packaging/soldering) can **induce VRT in previously normal cells**.
  * A manufacturer's factory test may no longer reflect real-world behavior by the time the chip is deployed.
  * **Combined DPD + VRT** makes finding a reliable, low-overhead profiling strategy one of the hardest open problems in DRAM design.


---
### 4. 🔧 Advanced Profiling Solutions

> Three research systems address the profiling challenge from different angles: Avatar uses ECC error feedback to adaptively raise refresh rates for cells that exhibit VRT. Reach Profiler accelerates profiling by exploiting temperature to trigger retention failures faster. HARP solves the subtle problem of profiling accuracy when on-die ECC masks individual bit errors.

* **Avatar: VRT-Aware Adaptive Refresh**
  * **Key insight**: instead of trying to predict VRT in advance (impossible), detect it reactively using ECC error feedback and respond by increasing the refresh rate for affected rows.
  * **Algorithm**:
    * Start all rows at a long refresh interval (e.g., 256 ms+) to minimize refresh overhead.
    * Periodically (e.g., every 15 minutes), read all DRAM rows and check for ECC-correctable errors.
    * If a row shows a retention error → **"upgrade" it**: move it to a shorter refresh interval bin.
    * The upgraded row is now protected from future failures.
    * Repeat periodically — cells that develop VRT over time are caught and upgraded.
  * **Results**:

    | Condition | Refresh Reduction |
    |---|---|
    | RADAR (no VRT tolerance) | ~70% |
    | Avatar after 12 months (with VRT) | ~60% |
    | Avatar with annual reset + re-profiling | ~70% (recovered) |

  * Annual reset strategy: after 12 months, reset all bins and re-profile from scratch — captures any new VRT-affected cells and re-earns the full 70% reduction.
  * Performance improvement for a 64 GB chip: Avatar achieves approximately **two-thirds** of the ideal "no refresh" performance — a dramatic improvement over standard universal refresh.
  * Avatar was awarded the **DSN 2025 Test of Time Award** and significantly influenced the adoption of **on-die ECC** as a standard feature in modern DRAM.

* **Reach Profiler: Faster Profiling via Temperature**
  * **Key insight**: higher temperatures accelerate charge leakage — a cell with a 1-second retention time at 25°C may fail in 64 ms at 85°C. By profiling at elevated temperature, you can identify weak cells much faster.
  * **Trade-off**: profiling at high temperature finds cells that fail quickly at high temp but may not fail at normal operating temperatures → **false positives** (cells profiled as weak but actually safe at normal temp).
  * **Reach Profiler** navigates the three-way trade-off between:
    * **Speed**: how quickly the profiling pass completes.
    * **Coverage**: what fraction of truly weak cells are identified (no false negatives).
    * **False positives**: what fraction of safe cells are mistakenly classified as weak.
  * **Results** (on 368 real LPDDR4 chips):
    * 2.5× faster than state-of-the-art baselines for 99% coverage at 50% false positive rate.
    * 3.5× faster with slightly higher false positive tolerance.
    * Enables operating at longer refresh intervals → 16% performance improvement, significant energy reduction.

* **HARP: Profiling Through On-Die ECC**
  * **Problem**: modern DRAM chips include **on-die ECC** (error correction within the DRAM chip itself, invisible to the memory controller). This masks individual bit errors from the controller — making retention profiling much harder.
  * Three specific challenges created by on-die ECC during profiling:
    * **Error amplification**: when too many errors exceed ECC's correction capability, the ECC decoder introduces **additional errors** (miscorrections) — profiling sees noise on top of noise.
    * **Error masking**: ECC corrects individual bit errors before the controller sees them → the controller cannot identify which specific rows/cells need a higher refresh rate.
    * **Data pattern interference**: standard worst-case data patterns used for DRAM testing interfere with ECC codeword boundaries, making DPD analysis unreliable.
  * **HARP solution**:
    * Add an **ECC bypass path** alongside the normal ECC path.
    * During profiling, read data through **both** the ECC path and the bypass path simultaneously.
    * The bypass path reveals all **direct errors** (actual cell failures) before ECC correction.
    * A **secondary ECC** (at least as strong as the on-die ECC) is applied to the bypass path to safely correct indirect errors introduced by the ECC decoder.
    * Net effect: HARP reduces the problem of "profiling with ECC" to "profiling without ECC" — eliminating all three challenges.
  * **Results**: 20–62% faster to achieve 99th-percentile coverage; 3.7× faster at a per-bit error probability of 0.75.


---
### 5. ⚡ Hiding Refresh Latency: HERO

> Reducing the number of refreshes (RADAR, Avatar) and speeding up profiling (Reach Profiler, HARP) address one dimension of the refresh problem. A complementary approach is to hide refresh latency by executing refresh operations in parallel with normal memory accesses — so the performance penalty of refresh becomes near-zero.

* **Two Types of DRAM Refresh**
  * **Periodic refresh**: restores charge in leaking cells to prevent data loss — affects all rows on a timer.
  * **Preventive refresh** (RowHammer mitigation): refreshes rows physically adjacent to a frequently hammered aggressor row — triggered by activation count, not time.
  * Both types contribute to the total refresh overhead in modern systems, and both worsen as DRAM density increases.

* **The HERO Idea: Concurrent Refresh and Access**
  * **Key insight**: a DRAM bank contains multiple **subarrays**, each with its own sense amplifiers. Activating a row in one subarray does not physically prevent activating a row in a different subarray of the same bank simultaneously.
  * **HERO** (Hidden Row Activation) exploits this by issuing a refresh activation to one subarray **concurrently** with a normal read/write activation to a different subarray in the same bank.
  * The refresh latency is effectively **hidden behind** the data access latency.
  * Standard DRAM timing parameters prevent this (requiring full activation + precharge before the next activation) — HERO requires carefully violating these parameters.

* **Experimental Validation**
  * HERO was first proposed theoretically in simulation (Kevin Chang, 2014) — showing clear performance benefits.
  * In 2022, it was demonstrated to work on **56 real, off-the-shelf SK Hynix DRAM chips** by violating timing parameters using an FPGA-based test platform.
  * Did not work on all vendors — some manufacturers implement circuitry that prevents such timing violations.

* **HERO Controller Design**
  * A HERO-aware memory controller:
    * Maintains a buffer of pending refresh requests with deadlines (how soon each row must be refreshed).
    * Continuously scans for opportunities to pair a pending refresh with an incoming data access targeting a different subarray in the same bank.
    * Issues both activations in rapid succession (e.g., 3 ns between commands instead of the standard ~10–15 ns).
  * Two operations that can be parallelized:
    * Refresh + data access (most common case).
    * Refresh + another refresh (for high-density chips with many pending refresh deadlines).

* **HERO Performance Results**

  | Refresh Type | HERO Speedup |
  |---|---|
  | RowHammer preventive refresh | **3.7×** |
  | Periodic data retention refresh | **12.6%** |

  * Speedup is larger for higher-density chips (more refresh operations to hide).
  * Hardware overhead: negligible — only control logic changes in the memory controller, no modifications to DRAM chips.

* **Combining HERO with RADAR/Avatar**
  * HERO and RADAR/Avatar are **complementary** — they address different aspects of the refresh problem:
    * RADAR/Avatar reduce the **number** of refresh operations needed.
    * HERO reduces the **performance penalty** of each refresh operation.
  * Used together, they cover both dimensions of the refresh overhead problem.


---
### 6. 🔁 Flash Memory: Analogous Retention Challenges

> Flash memory stores data as charge trapped in a floating gate — a fundamentally different mechanism from DRAM but with a strikingly similar retention problem. Charge leaks from the floating gate over time, causing threshold voltage shifts that corrupt stored data. The problem is severe enough to reduce read throughput by 16× for "old" data on real SSDs, and worsens dramatically with cell scaling and multi-level storage.

* **How Flash Memory Stores Data**
  * A Flash cell is a transistor with an additional **floating gate** layer between the control gate and the substrate.
  * **Programming**: apply high positive voltage to the control gate → electrons tunnel from the substrate to the floating gate via the Fowler-Nordheim tunneling effect → electrons become trapped in the floating gate.
  * **Effect of trapped charge**: the negative charge in the floating gate raises the transistor's **threshold voltage** (Vt) — a higher voltage is now needed to turn the transistor on.
  * **Reading**: apply a reference voltage to the control gate:
    * Transistor turns on → Vt < reference → cell stores 0.
    * Transistor stays off → Vt > reference → cell stores 1.
  * **Multi-level cell (MLC/TLC/QLC)**: by precisely controlling the amount of trapped charge, multiple Vt levels can be set → multiple bits per cell:
    * SLC: 1 bit/cell → 2 Vt levels.
    * MLC: 2 bits/cell → 4 Vt levels.
    * TLC: 3 bits/cell → 8 Vt levels.
    * QLC: 4 bits/cell → 16 Vt levels.

* **Retention Failure Mechanism in Flash**
  * Trapped electrons in the floating gate are not permanently held — they gradually tunnel back to the substrate over time.
  * This charge loss shifts the cell's Vt **downward** toward its unprogrammed state.
  * If Vt shifts enough to cross a reference boundary, the stored value is misread → **retention error**.
  * More bits per cell = smaller Vt margins between levels = more susceptible to even small charge shifts.

* **Real-World Impact: Samsung SSD 840 Case Study**
  * Users of the Samsung SSD 840 reported **severe read performance degradation for old files**:
    * Newly written files: ~500 MB/s read throughput (expected).
    * Files retained for weeks/months: as low as ~30 MB/s — a **16× throughput reduction**.
  * Root cause:
    * Retained charge leaks → Vt shifts → first read attempt produces an error.
    * The Flash controller detects the error, adjusts its reference voltage, and **retries the read**.
    * Multiple retries per page dramatically increase read latency.
  * Retention errors account for **>99%** of all observed Flash errors for one-year retention time — dominating over program, read, and erase errors.

* **Worsening with Cell Scaling**
  * 2D planar NAND: increasing capacity required shrinking cells → fewer trapped electrons → larger relative effect of leakage → worse retention.
  * 3D NAND: temporarily improved retention by stacking layers vertically (larger individual cells) instead of shrinking horizontally.
  * However, continuous capacity demand now pushes 3D NAND cells to also shrink → retention challenges are returning.

* **Flash Retention Solutions**
  * **Periodic refreshing** (analogous to DRAM refresh):
    * Modern SSD controllers periodically read stored data and rewrite it to restore charge levels.
    * Refresh frequency scales with **Program-Erase (P/E) cycle count**: an aged Flash chip (many P/E cycles) has more retention errors → needs more frequent refresh (e.g., daily vs. weekly vs. monthly for a new chip).
  * **Remapping-based refresh** (preferred for MLC/TLC):
    * Read the old page → write the refreshed data to a **new physical location** → update the Flash Translation Layer (FTL) mapping table.
    * Avoids in-place rewrite overhead and naturally integrates with the FTL that all SSDs already maintain.
    * Does not consume additional P/E cycles on the source location (which is erased and recycled).
  * **Intelligent Flash controllers**:
    * Track per-block P/E counts, data age, and error rates.
    * Dynamically adjust read reference voltages to compensate for Vt drift.
    * Use **ECC** (Reed-Solomon, LDPC) to correct remaining errors after voltage adjustment.
    * All of this complexity is handled transparently by the SSD controller — the host sees a standard block storage interface.

* **Flash vs. DRAM: Retention Problem Comparison**

  | Dimension | DRAM | Flash |
  |---|---|---|
  | Storage mechanism | Charge on capacitor | Charge on floating gate |
  | Leakage path | Via access transistor (leaky switch) | Tunnel oxide (quantum tunneling) |
  | Retention time without refresh | Milliseconds to seconds | Years (but degrades with P/E cycles) |
  | Refresh mechanism | Controller sends periodic activate commands | SSD controller reads and rewrites pages |
  | Multi-level storage | Not standard (single bit/cell) | SLC / MLC / TLC / QLC (1–4 bits/cell) |
  | Scaling impact | Smaller cap + leakier transistor | Smaller floating gate + thinner tunnel oxide |
  | Dominant mitigation | RADAR, Avatar, HERO | FTL remapping, read retry, ECC |
