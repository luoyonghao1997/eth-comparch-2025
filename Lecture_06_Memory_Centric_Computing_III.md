# Lecture 6: Memory-Centric Computing III — PIM Adoption Challenges


> This lecture takes a practical, engineering-focused view of PIM adoption. It begins by establishing rigorous methodologies for identifying which applications actually benefit from PIM — showing that simple metrics like arithmetic intensity or LLC miss rate are insufficient alone. It then addresses three layers of the adoption stack: **programmability** (from raw SDK code to high-level compiler frameworks), **system integration** (virtual memory and cache coherence), and **runtime optimization** (exploiting narrow values, inter-bit parallelism, and redundant binary representation to accelerate processing-using-DRAM). Throughout, the lecture emphasizes that PIM adoption requires a holistic, full-stack approach — no single layer can be fixed in isolation.


---
### 1. 🗂️ Identifying PIM-Suitable Applications

> Before building a PIM accelerator, you must know whether a workload is actually bottlenecked by data movement — and what kind. Simple metrics like arithmetic intensity or LLC miss rate are necessary but not sufficient. A comprehensive multi-metric methodology (DAM-Move) is needed to reliably classify applications and predict PIM benefit.

* **The Core Question: Is This Application Data-Movement-Bound?**
  * Intuition says: look for applications with high data movement.
  * Problem: data movement is abstract — slow performance or high memory traffic are symptoms, not quantifiable metrics.
  * Goal: tools and metrics that produce a number to classify applications as memory-intensive or compute-intensive with confidence.

* **The Roofline Model — and Its Limits**
  * The **roofline model** classifies applications using two axes:
    * **X-axis — Arithmetic Intensity**: number of arithmetic operations per byte of data accessed.
      * Example: C = A + B — if A and B are 4 bytes each and C is 4 bytes (12 bytes total), with 1 addition, arithmetic intensity = 1/12.
    * **Y-axis — Performance**: giga-operations per second achieved by the kernel.
  * Two "roofs" define the system limits:
    * **Compute roof** (horizontal line): peak arithmetic throughput (e.g., 1 GHz × 1 ALU = 1 GOPS).
    * **Memory roof** (sloped line): peak memory bandwidth × arithmetic intensity (e.g., DDR at 19.2 GB/s).
  * Classification:
    * **Below compute roof** → compute-bound (not suited for PIM).
    * **Below memory roof** → memory-bound (candidate for PIM).
  * **Critical limitation** — experiments across 100+ applications showed:
    * Many **memory-bound applications did not benefit from PIM** — the model gives false positives.
    * Some **compute-bound applications did benefit from PIM** — the model gives false negatives.
  * Root cause: arithmetic intensity captures only one dimension of data movement. It ignores how frequently data is reused, how far apart accesses are, and whether the cache can capture locality.

* **LLC Miss Rate — Another Incomplete Metric**
  * **Last-Level Cache (LLC) misses per kilo-instruction**: counts how often the L3 cache fails per 1,000 instructions.
  * High LLC misses/kilo-instruction → many instructions servicing memory → memory-bound → PIM candidate.
  * **Same limitation**: experiments show many applications with **low LLC miss rates still benefited from PIM**, and some with high miss rates did not.
  * Conclusion: no single metric reliably predicts PIM suitability.

* **DAM-Move: A Comprehensive Multi-Metric Methodology**
  * DAM-Move combines application-level and system-level metrics into a structured classification pipeline:
    * **Step 1 — Application profiling**: identify time-consuming, potentially memory-bound code regions to focus analysis.
    * **Step 2 — Locality classification**: collect memory traces via simulation; compute spatial and temporal locality scores.
    * **Step 3 — Scalability analysis**: run simulations under varying system configurations to compute system-dependent metrics.
    * **Step 4 — Classification**: combine all metrics into one of **six data movement bottleneck classes**, each correlated with a specific mitigation strategy.

* **Locality Metrics (Application-Dependent)**
  * **Spatial locality**: how often consecutive memory accesses fall within the same cache line (64 bytes).
    * Measured via **stride profiling histograms**: for each pair of consecutive accesses, record the stride (distance between addresses).
    * Stride = 1 → high spatial locality (accessing adjacent elements). Large strides → low spatial locality.
    * Normalized to [0, 1]: 1 = perfectly sequential, 0 = fully random.
  * **Temporal locality**: how often the same address is accessed multiple times.
    * Measured via **reuse histograms**: count how many times each address is revisited.
    * Normalized to [0, 1]: 1 = high reuse, 0 = each address accessed exactly once.
  * **Key example — deep learning convolutions**:
    * High spatial locality (accessing a spatially clustered sliding window of elements).
    * Low temporal locality (once the window moves, those elements are not reused).
    * → Memory-intensive: caches are largely ineffective, making PIM highly suitable.
  * These locality metrics are **application properties** — independent of hardware. The same application has the same locality on CPU, GPU, or PIM.

* **System-Dependent Metrics**
  * **Arithmetic intensity**: same as roofline model — compute vs. data moved.
  * **LLC misses per kilo-instruction**: same as before.
  * **Last-to-First Miss Ratio (LFMR)**: LLC misses / L1 misses.
    * High LFMR → a miss at L1 also missed at L2 and L3 → caches provide almost no benefit → strong PIM candidate.
    * Low LFMR → misses are absorbed by intermediate cache levels → increasing cache size may suffice (PIM not needed).

* **Six Data Movement Bottleneck Classes**
  * The combined metrics classify applications into six classes, each pointing to a different solution:

    | Class | Characteristics | Best Mitigation |
    |---|---|---|
    | L1 cache capacity-bound | Low temporal locality, misses absorbed by L2/L3 | Increase L1 cache size |
    | L2/L3 cache capacity-bound | High LFMR, some temporal locality | Increase LLC size |
    | DRAM bandwidth-bound | Low temporal locality, high LFMR, high LLC misses/kI, low AI | **PIM (high bandwidth)** |
    | DRAM latency-bound | Same as above but low LLC misses/kI | **PIM (low latency)** |
    | Compute-bound | High arithmetic intensity | CPU/GPU compute accelerator |
    | Mixed | Varies by kernel | Split: PIM for memory-bound kernels, CPU for compute-bound |

  * DAM-Move is open-source and available on GitHub with a complete analysis toolchain.

* **Direct Deployment: Running on Real PIM Hardware**
  * The most direct suitability assessment — run the application on actual PIM hardware and measure.
  * **UPMEM system**: commercial PIM architecture with RISC-like DPU cores embedded next to DRAM banks.
  * The **Prim benchmark suite**: 16 applications from diverse domains (ML, sparse matrix, genome alignment, homomorphic encryption, reinforcement learning), all implemented and profiled on UPMEM.
  * Trade-off: requires significant engineering effort and access to PIM hardware, but produces the most concrete and realistic results.

* **Locality Requirements for PUM Specifically**
  * PUM (processing using DRAM) operates on data **at its storage location** — computation occurs in the same subarray as the data.
  * Locality requirements for PUM differ from PNM:
    * **Spatial locality is required**: PUM operates on vectors the width of a DRAM row (or a DRAMat fraction) — data must be organized consecutively in memory.
    * **Temporal locality is NOT required**: once data is in the DRAM subarray for computation, it does not need to be reused — in fact, PUM excels at workloads with **high spatial but low temporal locality** (e.g., convolutions, bulk transforms).
  * This is a key differentiator: GPUs also use SIMD (wide parallelism) but still suffer from data movement when temporal locality is low. PUM eliminates that movement entirely.


---
### 2. 💻 Programmability: From Raw SDK to Transparent Compilers

> Programming PIM systems today requires detailed knowledge of hardware internals — alignment constraints, scratchpad management, explicit data movement between host and PIM memory. Three progressive approaches address this: SimplePIM (high-level APIs), DAPA (data-flow programming), and MimRAM (LLVM-based automatic code generation). Each trades off abstraction level against flexibility and performance.

* **The Programmability Barrier: UPMEM as a Case Study**
  * **UPMEM architecture**:
    * Host CPU + standard DRAM + PIM-enabled DIMMs containing thousands of **DPUs (DRAM Processing Units)**.
    * Each DPU: 64 MB DRAM bank + DMA engine + **64 KB WRAM scratchpad (SRAM)**.
    * The DPU pipeline accesses **only WRAM** — all data must be manually DMA'd from the DRAM bank to WRAM before any arithmetic.
  * **Programming model: loosely coupled accelerator** (like GPU):
    * PIM memory is a **disjoint address space** from host memory — data cannot be directly shared.
    * Data must be explicitly copied from host DRAM → PIM DRAM before computation, and results copied back afterward.
    * This seems to contradict PIM's goal of reducing data movement — but the idea is to **amortize** the initial transfer:
      * Copy input data once → PIM produces large amounts of intermediate results that stay local → copy only the small final output back.
  * **What a programmer must do for a simple vector addition**:
    * Align input arrays to 8-byte boundaries.
    * Explicitly distribute data to DPUs using SDK scatter APIs.
    * Launch DPU execution (synchronous or asynchronous) and manage synchronization.
    * Explicitly copy results back with gather APIs (8-byte aligned).
    * Inside the DPU: manually DMA data from DRAM bank → WRAM scratchpad → perform arithmetic → DMA results back.
  * The actual addition logic may be 3–5 lines; surrounding management code can be 50–100 lines.

* **SimplePIM: High-Level C++ APIs**
  * SimplePIM wraps the UPMEM SDK with three families of C++ classes:
    * **Management interface**: `register_array` (mark data as PIM-resident), `lookup`, `free`.
    * **Communication interface**:
      * `broadcast`: copy one array to all DPUs.
      * `scatter`: distribute an array across DPUs in interleaved fashion.
      * `gather`: collect results from all DPUs into a single host array.
      * PIM-to-PIM primitives: `reduce` (aggregate across DPUs), `all_gather`.
    * **Processing interface** (data-parallel programming, MapReduce-style):
      * `map`: apply the same function to every element → same-size output array.
      * `reduce`: aggregate elements → smaller output.
      * `zip`: combine two same-size arrays element-wise.
  * **Backend optimizations** automatically applied: strength reduction, loop unrolling, bound checking elimination, function call inlining, transfer size alignment.
  * **Results on real UPMEM (2,400+ DPU cores)**:
    * Histogram computation: SDK code → dozens of lines; SimplePIM → a few API calls.
    * Performance matches or exceeds hand-optimized SDK code for strong and weak scaling.
  * **Remaining limitation**: programmer must still understand host–PIM memory separation and explicitly specify scatter/gather patterns — not fully transparent.

* **DAPA: Data-Flow Programming for PIM**
  * DAPA removes the need for programmers to specify communication patterns by shifting to a **data-flow** model:
    * Programmer specifies computation as a **pipeline of stages**, each stage being a parallel pattern.
    * DAPA handles all data distribution, DPU assignment, scratchpad management, and inter-stage communication automatically.
  * **Extended primitive set** (beyond SimplePIM):
    * `filter`: apply a predicate; copy elements where predicate is true, discard others.
    * `window`: apply a rolling window function across the array.
    * `group`: non-interleaving rolling window (for grouped aggregations).
    * Primitives are composable — complex pipelines built from primitive chains.
  * **Example — dot product**:
    * Stage 1: `map` (element-wise multiply).
    * Stage 2: `reduce` (sum all products).
    * No PIM-specific code — this is standard parallel programming notation.
  * **Dynamic template-based compilation**: DAPA's backend analyzes the data-flow graph, identifies input arrays for PIM, distributes them, manages scratchpad, and triggers stage-by-stage computation.
  * **Portability**: the same front-end DAPA code targets any PIM architecture — backend generates hardware-specific code.
  * **Results**: better performance than hand-optimized implementations (backend optimizations), significantly fewer lines of code, and higher abstraction than SimplePIM (no communication patterns needed).

* **MimRAM: LLVM-Based Compiler for Processing-Using-DRAM**
  * SimplePIM and DAPA require programmers to rewrite applications using their APIs. MimRAM goes further: **automatically generate PUM code from standard C programs** via LLVM compiler modifications.
  * **Approach — leverage existing auto-vectorization**:
    * Modern compilers (GCC, Clang/LLVM with `-O3`) automatically identify regular loops and vectorize them into SIMD instructions (Intel AVX-512, ARM Neon).
    * MimRAM hijacks this auto-vectorization pipeline — instead of generating AVX code, it generates PUM instructions for the DRAM substrate.
  * **Key differences between CPU SIMD and PUM SIMD**:

    | Property | CPU SIMD (AVX-512) | PUM (MimRAM/MimDRAM) |
    |---|---|---|
    | Lane count | Fixed (e.g., 512 bits) | Flexible, up to ~65,000 lanes |
    | Registers | Explicit vector registers | No registers — DRAM cells are the "register" |
    | Register spilling | Common at high parallelism | Does not exist |
    | Load/store needed | Yes — data fetched from memory | No — data already in DRAM |
    | Granularity | Fixed to vector width | Configurable per DRAMat |

  * **Compiler modifications**:
    * Remove load/store instructions (data already in DRAM).
    * Adjust cost models to reflect no register spilling and wider SIMD.
    * Generate PUM-specific instruction sequences instead of AVX intrinsics.
  * **Orchestrating computation — data dependence graph**:
    * The compiler builds a dependence graph of operations in each loop.
    * Independent operations → assigned to different DRAMats for concurrent execution (MIMD model).
    * Dependent operations (e.g., B depends on A) → scheduled sequentially; if data is in different DRAMats, inter-DRAMat movement is inserted.
    * Instructions annotated with metadata labels: "independent — any DRAMat" or "dependent — requires data from DRAMat X".
  * Result: a standard C program with vectorizable loops is automatically compiled to efficient PUM code — no API rewriting required.


---
### 3. 🔧 System Integration: Virtual Memory and Cache Coherence

> Two fundamental operating system mechanisms — virtual memory and cache coherence — both conflict with PIM's goals in non-trivial ways. Virtual memory hides physical data location, which is critical for PUM where computation location equals data location. Cache coherence generates traffic between host and PIM that can negate PIM's data movement reduction.

* **Virtual Memory and PIM: The Address Translation Problem**
  * **Background — how memory allocation works**:
    * An application calls `malloc` → OS returns **virtual addresses** (no physical location assigned yet).
    * On first access, the OS maps virtual pages → physical **frames** (4 KB pages of physical RAM).
    * The memory controller applies a **proprietary hash function** to map physical frame numbers → DRAM rows, banks, ranks, and subarrays.
    * This hash function is **not exposed** to the OS, compiler, or programmer — it is internal to the memory controller chip.
  * **The resulting translation chain**:
    ```
    Application virtual address
           ↓ (OS page table + MMU)
    Physical frame number
           ↓ (Memory controller hash function — proprietary)
    DRAM bank / subarray / row / column
    ```
  * **Guarantees provided by standard allocation**:
    * `malloc`: no alignment guarantee at any level.
    * `posix_memalign` / aligned `new`: virtual address alignment only — no guarantee on physical frame placement.
    * **Huge pages** (2 MB): guarantee that physical frames within a huge page are contiguous and aligned — but still no guarantee on which DRAM subarray those frames land in.
  * **Why this matters for PUM**:
    * RowClone requires source and destination rows to be in the **same DRAM subarray**.
    * Triple row activation (Ambit AND/OR) requires operand rows to be in the **same or adjacent subarray**.
    * If the allocator cannot control which subarray data lands in, PUM operations either fail silently or produce wrong results.
  * **Why this matters for PNM**:
    * Graph applications use pointer chasing — each node read reveals the address of the next node.
    * If virtual addresses need translation before each pointer dereference, the translation itself generates data movement (page table lookups), negating PIM's benefit.
    * Distributing page tables across thousands of PIM cores adds memory overhead and latency.

* **Solution: PIM-Aware Memory Allocator (MimDRAM)**
  * **Key insight**: combine **huge pages** (which guarantee physical frame contiguity) with **reverse-engineered DRAM hash functions** (which reveal the frame-to-subarray mapping).
  * **Reverse engineering the hash function**:
    * Brute-force approach: attempt RowClone between pairs of rows on an FPGA platform.
    * Successful RowClone → both rows are in the same subarray → the hash function maps their frame numbers to the same subarray.
    * Collect enough data points to reconstruct the hash function.
    * Alternative: measure access latency — rows in the same subarray have lower bank conflict latency.
    * Memory manufacturers know these functions internally but do not publish them.
  * **Allocator behavior**:
    * Allocate data using huge pages → guarantees physical frame contiguity.
    * Apply the reverse-engineered hash function → predict which DRAM subarray each frame lands in.
    * For dependent PUM operations (e.g., RowClone from array A to array B), accept the source subarray as a parameter and return an address guaranteed to map to the **same subarray**.
  * **Result**: PUM operations can be reliably issued because the allocator ensures the required physical co-location of operands.

* **Cache Coherence in PIM: The Traffic Problem**
  * **Standard coherence requirement**: if the host CPU and a PIM engine both access the same memory region, they must see consistent data — writes by one must be visible to the other.
  * **The coherence overhead chain**:
    * PIM engine writes to address X → must send **invalidation message** to host CPU's L1/L2/L3 caches holding a stale copy of X.
    * Host CPU updates address Y → must send invalidation to PIM engine if PIM holds a copy.
    * Each coherence message is a bus transaction → **more data movement** → contradicts PIM's core goal.
  * **Evaluation**: applying standard fine-grained coherence protocols to PIM systems often results in performance **worse than CPU-only execution** — the coherence overhead erases all PIM benefit.

* **Solution: Optimistic Execution with Signatures (Lazy PIM)**
  * **Core observation**: in well-designed PIM applications, the host CPU and PIM engine typically operate on **disjoint memory regions** — conflicts are rare.
  * **Optimistic approach**:
    * CPU offloads computation to PIM → both continue executing without sending coherence messages.
    * PIM engine records every address it **writes** during an execution epoch in a **checkpoint area** (a compact bitmap of modified addresses).
    * At the end of the epoch, PIM sends a **signature** (the write bitmap) to the host CPU.
    * Host CPU checks its own write set for that epoch against the PIM signature.
    * If **no overlap** → no conflict → computation is valid → continue.
    * If **overlap** → conflict detected → PIM must **re-execute** the epoch with correct data.
  * **Why this works**:
    * Only writes are tracked (reads do not create coherence conflicts).
    * Signatures are sent only at epoch boundaries — not per memory access.
    * Re-execution is rare if the application is designed for PIM (disjoint access sets).
  * **Results**: significantly less coherence traffic and much better performance than fine-grained coherence, approaching the ideal PIM performance (no coherence overhead).


---
### 4. ⚡ Runtime Optimizations for Processing-Using-DRAM

> Bit-serial SIMD in DRAM (the PUM execution model) has a subtle but critical inefficiency: operations scale linearly — or even quadratically — with bit precision. Most real programs use far fewer significant bits than their declared data type suggests. Three runtime optimizations — narrow value detection, inter-bit parallelism, and redundant binary representation — exploit this to dramatically reduce PUM latency and energy.

* **Background: How Arithmetic Works in PUM (Bit-Serial SIMD)**
  * PUM (via SIM/Ambit) performs arithmetic bit by bit across an entire DRAM row simultaneously.
  * Example — adding two 8-bit vectors A and B stored column-wise in a DRAM subarray:
    * Use a ripple-carry adder circuit, translated into majority-gate operations (triple activations).
    * Compute carry-out from bits 0 → carry-in for bit 1 → carry-out from bit 1 → ... → repeat for all 8 bits.
    * Each bit position requires multiple triple activations — and carry must propagate serially from LSB to MSB.
  * For 32-bit or 64-bit data types, this process repeats 32 or 64 times → latency and energy scale **linearly with bit precision**.

* **Optimization 1: Narrow Value Detection**
  * **The problem**:
    * Programmers declare variables as `int` (32-bit) or `long` (64-bit) even when actual values are small.
    * Example: if array A contains values 0–4 (range fits in 3 bits) stored as 32-bit integers, the top 29 bits are always zero.
    * PUM's bit-serial engine still executes all 32 activation rounds — 29 rounds on inconsequential zero bits.
    * Wasted latency and energy scale with the unused bit width.
  * **Real-world validation**: analysis of real applications shows that required bit precision varies widely — many applications effectively need only 4–12 bits even when using 32-bit types.
  * **Solution: dynamic precision tracking**:
    * Track the **maximum value** of each array as data is written to DRAM (probed at cache eviction time).
    * Maximum value → required bit precision = ⌈log₂(max\_value)⌉.
    * For operations: max\_output = max\_A + max\_B (for addition) → required output precision = ⌈log₂(max\_A + max\_B)⌉.
    * Issue triple activations only for the necessary bit positions → skip all higher zero bits.
    * Result: latency and energy reduced proportionally to the ratio of effective precision to declared precision.

* **Optimization 2: Inter-Bit Parallelism**
  * **The problem**: even with narrow value detection, carry propagation in adders creates **inter-bit serialization** — bit N cannot complete until bit N-1's carry is resolved.
  * **Observation**: within each bit position's computation, many majority gates are **independent of the carry** and could execute concurrently — but cannot if all bits are in the same subarray sharing one sense amplifier.
  * **Solution: distribute bits across subarrays**:
    * Bit 0 of all elements of A and B → subarray 0.
    * Bit 1 of all elements → subarray 1.
    * ... and so on.
    * Each subarray has its own sense amplifiers → can process its bit position independently.
    * Add a **fast inter-subarray interconnect** (e.g., via LISA isolation transistors) to propagate carry from subarray N to subarray N+1.
    * Once carry arrives, subarray N+1 can complete its independent gates concurrently with subarray N finishing its remaining operations.
    * Result: pipeline carry propagation across subarrays → significant reduction in total DRAM cycles.

* **Optimization 3: Redundant Binary Representation (RBR)**
  * **The problem**: multiplication in bit-serial PUM requires shift-and-add operations → latency scales **quadratically** with bit precision. For high-precision multiplications, neither narrow value detection nor inter-bit parallelism suffices.
  * **Root cause**: two's complement binary representation inherently requires carry propagation chains that grow with bit width.
  * **Solution — Redundant Binary Representation (RBR)**:
    * Instead of one bit per digit (values 0 or 1), use two bits per digit (values −1, 0, or +1).
    * Redundancy (multiple ways to represent the same number) enables **carry-free addition**:
      * In RBR, partial products in multiplication can be added without carry propagation beyond 2 bit positions.
      * This makes the latency of high-precision multiplication **constant** with respect to bit width (for the main computation phase).
    * Trade-off: encoding (standard binary → RBR) and decoding (RBR → standard binary) add overhead.
    * Benefit: for high-precision, quadratically-scaling operations, the conversion overhead is small relative to the computation savings.

* **ProTails: Runtime Framework Combining All Three Optimizations**
  * **Architecture** — three hardware components between the LLC and memory controller:
    * **Offline microprogram library**: multiple implementations of each arithmetic operation (ripple-carry adder, carry-lookahead adder, RBR-based multiplier, etc.), each with a different latency-precision trade-off profile.
    * **Dynamic precision engine**: hardware that probes maximum values of arrays being evicted to DRAM → stores per-array maximum values in a small SRAM table.
    * **Microprogram selection unit**: at PUM execution time, queries the precision engine → computes required output precision → selects the optimal microprogram for that precision.
  * **Runtime operation**:
    * A PIM operation is triggered (e.g., add arrays A and B).
    * Precision engine reports max(A) and max(B) → required precision = ⌈log₂(max\_A + max\_B)⌉.
    * Selection unit picks the best-performing adder microprogram for that precision.
    * Microprogram is dispatched to the DRAM controller → triple activations issued only for necessary bits.
  * **Results across 12 applications**:
    * **SIM (static 32-bit precision)**: often performs worse than CPU — wasted activations on zero bits.
    * **ProTails**: significant speedup over SIM by skipping inconsequential zero-bit activations.
    * Performance gains translate directly to proportional **energy savings**.
  * **General principle**: understanding the data running on a system — not just the computation — enables optimizations unavailable to generic, data-agnostic hardware.


---
### 5. 🛠️ Testing Infrastructure for PIM Research

> PIM research requires specialized infrastructure to test timing parameter violations, measure reliability, and simulate new DRAM behaviors. FPGA-based platforms provide the fine-grained control that commercial memory controllers deliberately withhold.

* **FPGA-Based Memory Testing Platforms**
  * Standard memory controllers enforce JEDEC timing parameters and do not allow violations — by design, to protect reliability margins.
  * FPGA-based platforms replace the memory controller with programmable logic, allowing:
    * **Arbitrary timing parameter values**: reduce tRCD, tRRD, tRP, etc. to any value.
    * **Custom command sequences**: issue activate–activate, activate–triple, or any non-standard sequence.
    * **Per-row, per-column measurements**: observe which cells fail, when, and under what conditions.
  * **SoftMC**: first open-source FPGA-based DRAM testing platform — demonstrated RowClone feasibility on commodity chips.
  * **DRAM Bender**: successor to SoftMC with enhanced capabilities — used for PUM operation validation (AND, OR, NOT, multi-row copy, RNG) on real chips from Samsung, SK Hynix, and Micron.
  * Both tools are open-source on GitHub and form the foundation for most experimental PUM research.

* **Simulators**
  * **Ramulator / Ramulator 2.0**: cycle-accurate DRAM simulator supporting many DRAM standards; extended in v2.0 with PIM-specific modules.
  * **Remulator**: a simulator specifically designed for PIM research — used in course projects.
  * Role of simulators: evaluate PIM designs at future technology nodes or with hardware modifications not yet physically built — essential for design-space exploration before committing to fabrication.
