# Lecture 5: Memory-Centric Computing II — Processing Near Memory


> This lecture shifts from the bottom-up **processing using memory (PUM)** paradigm to the top-down **processing near memory (PNM)** paradigm, where functional units are placed physically close to DRAM to reduce data movement. It surveys PNM accelerators across a wide range of application domains — graph analytics, ML inference, LLM inference, genome sequencing, and consumer workloads — and shows consistent gains of 10–30× in performance and energy. The lecture concludes with an honest assessment of PIM adoption barriers: programmability, cache coherence, virtual memory, synchronization, and security all require fundamental new solutions before PIM can be deployed at scale.


---
### 1. 🔄 Recap: PUM and Additional In-DRAM Operations

> Before moving to PNM, this section consolidates the PUM landscape with two more capabilities that commodity DRAM chips can perform by violating timing parameters: true random number generation and physical unclonable functions. It also introduces DRAM-as-lookup-table as a bridge toward more complex in-memory computation.

* **What PUM Showed Us (Recap)**
  * By violating DRAM timing parameters on unmodified commodity chips, experiments confirmed:
    * **Multi-row copy**: one source row copied to up to **31 destination rows** with ~99.98% reliability.
    * **NOT**: complement of a full row computed using the sense amplifier's inherent cross-coupled structure.
    * **AND, OR, NAND, NOR**: computed by concurrently activating rows across neighboring subarrays — up to 16 inputs.
  * These capabilities were not designed into the chips — they emerge from the underlying physics. Intentionally designed PUM chips would be far more reliable and powerful.

* **Random Number Generation (RNG) in DRAM**
  * Core observation: violating the **TRCD** timing parameter (time from row activation to read command) causes some cells to fail — and some cells fail **randomly**.
    * Cells with enough stored charge (strong cells) produce a correct read even with reduced TRCD.
    * Cells very close to the sensing threshold **Vmin** fail randomly due to thermal noise and process variation → these are **RNG cells**.
    * Other cells fail **deterministically** every time — these become Physical Unclonable Functions (see below).
  * **RNG Procedure**:
    * Identify RNG cells by profiling the DRAM chip.
    * Repeatedly read RNG cells with reduced TRCD → aggregate the random bit pattern → post-process for uniformity.
    * Output passes all 15 **NIST randomness tests**.
  * **QUAC-RNG** — a higher-throughput variant:
    * Initializes four rows with conflicting data (alternating all-ones and all-zeros).
    * Performs rapid activate–precharge–activate sequences violating multiple timing parameters simultaneously.
    * Sense amplifiers resolve to random states, collected as random bits.
    * Performance:

      | Metric | QUAC-RNG |
      |---|---|
      | Peak throughput | 54 Gbit/s |
      | Average per-channel throughput | 344 Gbit/s |
      | vs. state-of-the-art RNG | **15× better** |
      | NIST tests passed | All 15 |
      | Area overhead | Negligible |

    * Entropy quality depends on temperature and drifts slowly over time — periodic recalibration required (stable for up to ~1 month).
  * **SMRA (Simultaneous Multi-Row Activation)** variant:
    * Activating more rows increases entropy but also increases latency, reducing throughput.
    * Optimal sweet spot found at **8 simultaneously activated rows**.
  * **Why this matters**: Most embedded systems need true hardware RNG for security (e.g., key generation, nonces). Dedicated RNG hardware is costly. Since embedded systems already contain DRAM, DRAM-based RNG provides a free, high-quality entropy source.

* **Physical Unclonable Functions (PUFs) in DRAM**
  * Cells that fail **deterministically** when TRCD is violated — unlike RNG cells that fail randomly — produce the same output pattern every time for a given chip.
  * This consistent pattern is unique per chip due to manufacturing variation → a **hardware fingerprint** or PUF.
  * Use cases:
    * **Device authentication**: a cloud provider or user can challenge a device by requesting its DRAM PUF and verifying it matches the enrolled signature.
    * **Anti-counterfeiting**: verify that a device contains the genuine DRAM chip it claims to have.
  * No additional hardware required — the PUF is an intrinsic property of the existing DRAM.

* **DRAM as Lookup Tables (LUTs)**
  * PUM's native operations (AND, OR, NOT) are limited to bulk bitwise logic — complex arithmetic is expensive in bit-serial form.
  * Bridge approach: use DRAM cells as **lookup tables**:
    * Store precomputed function outputs in DRAM, indexed by input values.
    * A read to address `f(A, B)` returns the result of the function — computation becomes a memory access.
    * This is exactly how **FPGAs implement logic** using their SRAM-based LUTs.
  * Opens the door to arbitrary complex operations in DRAM without needing custom circuits, at the cost of memory space for the table.


---
### 2. ⚡ Processing Using Other Memory Technologies

> PUM principles extend beyond DRAM. Flash memory can perform AND/OR operations through its NAND string structure. Emerging non-volatile memories (PCM, ReRAM, memristors) can perform analog multiply-accumulate operations via Kirchhoff's current law — a natural fit for neural network inference.

* **FlashCosmos: Computation Inside NAND Flash**
  * **Core physical insight**: A NAND flash string is a series connection of cell transistors — structurally equivalent to a CMOS NAND gate.
    * Sensing all cells in a NAND string simultaneously implements an **AND** function.
    * Different NAND strings are connected in parallel to the bit line → parallel strings implement **OR**.
    * Combinations enable **NAND**, **NOR**, and compound **AND-OR** expressions.
  * **Reliability challenge**: Standard flash sensing techniques ensure reliable reads for storage but are insufficient for computation, where cell states are intentionally ambiguous.
  * **Enhanced SSD Mode Programming (ESP)**:
    * Maximizes the voltage margin between programmed states, increasing the reliability of in-flash computation.
    * Result: zero observed bit errors during in-flash compute operations in tested devices.
  * **Practical results on real chips** (no cell array modifications required):
    * AND operations with up to **48 operands**.
    * OR operations with up to **4 operands**.
    * Small increase in sensing latency — acceptable for the gained compute capability.
  * **Cipher-Match**: extends FlashCosmos primitives to accelerate **homomorphic encryption**:
    * Homomorphic encryption (HE) allows computation on encrypted data without decryption — preserving privacy on untrusted servers.
    * HE is notoriously slow (often 1,000–1,000,000× slower than plaintext computation).
    * Cipher-Match uses in-flash serial addition (built from AND/OR primitives) plus algorithmic optimizations to dramatically accelerate HE-based exact string matching inside the SSD.

* **Emerging Non-Volatile Memories: Analog MAC via Kirchhoff's Law**
  * **Crossbar array structure**: non-volatile memory cells (memristors, ReRAM, PCM) arranged in a grid with word lines (rows) and bit lines (columns).
  * **Physical computation**:
    * Apply voltage V to a word line → current I = V × G flows through a cell with conductance G.
    * Currents from multiple cells sum on the bit line: I_total = V₁·G₁ + V₂·G₂ + ... → this is a **dot product (MAC operation)**.
  * **Neural network inference mapping**:
    * **Input activations** → encoded as voltages on word lines.
    * **Weight matrix** → encoded as cell conductance values (programmed into the crossbar).
    * **Output** → read as current on bit lines → one full matrix-vector multiplication **per sense cycle**.
  * Required peripherals: DAC (digital input → analog voltage), ADC (analog current → digital output), sample-and-hold circuits.
  * Advantage: a single crossbar operation replaces what would require thousands of multiply-accumulate instructions on a digital processor.
  * Key challenge: analog noise, device variability, and limited conductance precision introduce errors — reliability and accuracy management are active research areas.


---
### 3. 🖥️ Processing Near Memory (PNM): The Top-Down Paradigm

> PNM places conventional logic (ALUs, cores, accelerators) physically close to DRAM banks — typically in a 3D-stacked logic layer — rather than exploiting DRAM's own physics. It is a top-down approach: identify the most data-movement-bound applications, then design accelerators that eliminate that movement. PNM is easier to adopt than PUM and dominates current industry activity.

* **PNM vs. PUM: Fundamental Difference**
  * **PUM** (bottom-up): start from what DRAM physics can naturally do → build up to applications.
  * **PNM** (top-down): start from application data movement bottlenecks → design logic to sit near memory and eliminate those bottlenecks.

  | Dimension | PUM | PNM |
  |---|---|---|
  | Logic source | DRAM device itself | Separate logic layer or chip |
  | Operations supported | Bulk bitwise, copy, RNG | Arbitrary (full ALUs, cores) |
  | Hardware change | Timing violations or DRAM redesign | 3D stacking or side-car logic |
  | Adoption difficulty | Harder | Easier — industry focus |
  | Flexibility | Limited by DRAM physics | High |

* **Key Technology: 3D-Stacked Memory**
  * Multiple DRAM dies stacked vertically, connected by **Through-Silicon Vias (TSVs)**.
  * A **logic layer** at the bottom sits millimeters from all DRAM banks above.
  * Benefits:
    * **Internal bandwidth**: 10–100× higher than external DRAM bus.
    * **Latency**: drastically reduced vs. off-chip DRAM access.
    * **Energy**: data moves micrometers rather than centimeters.
  * Example products: **HBM (High Bandwidth Memory)**, formerly **HMC (Hybrid Memory Cube)**.


---
### 4. 📊 PNM Applications: Graph Analytics

> Graph processing is one of the most memory-bound workloads in existence. Its irregular access patterns defeat caching, and its low arithmetic intensity means data movement cost can never be amortized by computation. PNM with 3D-stacked memory and redesigned programming models delivers over 30× speedup and major energy reductions.

* **Why Graph Analytics Is Memory-Bound**
  * **Irregular (random) memory accesses**: graph traversal follows pointer chains — next node address is only known after reading the current node. Prefetchers and caches cannot predict these patterns.
  * **Low arithmetic intensity**: for each cache line fetched (64 bytes), only a few arithmetic operations are performed. The data movement cost cannot be amortized.
  * **Sparse data structures**: large graphs are stored in compressed sparse formats (CSR, CSC), which produce even more non-sequential memory access patterns.
  * Even high-end **GPUs** are severely underutilized running graph workloads — their cores sit idle waiting for memory.

* **Tesseract: Graph Analytics on 3D-Stacked Memory**
  * **Architecture**: multiple HMC (or HBM) cubes connected together.
    * Each cube contains multiple **vaults** — independent memory partitions, each with its own DRAM controller and a simple compute core.
    * Adding more cubes scales both **memory capacity** and **aggregate compute power**.
  * **Key innovation — Remote Function Calls (RFC)**:
    * Graph traversal frequently needs data in a different vault or cube from where the current computation is running.
    * Instead of moving the data to the compute vault (expensive), **send the computation to the vault that owns the data**.
    * The vault executes the function locally on its data and returns only the result.
    * This is highly effective for random-access workloads where data cannot be pre-mapped to a single vault.
  * **Prefetching**: a dedicated prefetching unit inside each vault exploits what little locality exists to further utilize internal memory bandwidth.
  * **Results**:
    * Over **30× performance improvement** vs. CPU with DDR3 memory.
    * Internal memory bandwidth utilized: **2.9 TB/s** of a theoretical **8 TB/s** — indicating room for further optimization.
    * Significant **energy reduction**.

* **Algorithm-Architecture Co-Design for Graph PIM**
  * Algorithmic representation dramatically affects how well a workload maps to PIM:
    * **Linear algebra** (adjacency matrices, semiring operations): suits workloads like triangle counting and single-source shortest path.
      * Example: in shortest-path, replace × with + and + with min in the matrix multiplication — the hardware sees the same operation.
    * **Set algebra** (union, intersection): suits graph pattern mining and subgraph matching.
      * **Set union** = bulk OR on bit vectors (maps directly to Ambit).
      * **Set intersection** = bulk AND on bit vectors (maps directly to Ambit).
  * Important insight: algorithms designed for processor-centric systems (maximizing ILP) may perform worse on memory-centric systems, and vice versa. **Algorithms must be redesigned** with PIM in mind — not just the hardware.


---
### 5. 🤖 PNM for Machine Learning

> ML workloads exhibit extreme heterogeneity: different models and different layers within the same model span a 200× range in arithmetic intensity. A monolithic accelerator (like a TPU) inevitably over-provisions for some layers and under-provisions for others. PNM-aware heterogeneous accelerators that dynamically match compute resources to layer characteristics achieve major gains over monolithic designs.

* **The Problem with Monolithic ML Accelerators (Edge TPU)**
  * A study of 24 Google Edge ML models (CNNs, LSTMs, transducers, RCNNs) found:
    * TPUs operate **far below peak throughput** when running memory-bound layers.
    * TPUs exhibit **far below theoretical energy efficiency** for the same reason.
    * Root cause: a **monolithic "one-size-fits-all" design** cannot handle the heterogeneity of modern ML workloads.
  * Heterogeneity observed:
    * **Across models**: CNN layers have high flops/byte (compute-bound); LSTM and transducer layers have very low flops/byte (memory-bound).
    * **Within a single model**: adjacent layers in one CNN differ by up to **244× in flops/byte** and **200× in MAC intensity**.

* **Mensa: Heterogeneous PNM for Edge ML**
  * **Design principle**: dynamically categorize each layer at runtime, then route it to the most suitable accelerator.
  * Layer families:
    * **Family 1 & 2**: low parameter footprint, high data reuse, high MAC intensity → **compute-centric** (suitable for TPU-like units).
    * **Family 3, 4 & 5**: high parameter footprint, low data reuse, low MAC intensity → **data-centric** (suitable for PIM accelerators).
  * Mensa places **PIM engines** with high memory bandwidth near DRAM for data-centric families.
  * Result: significant **energy reduction** and **throughput improvement** vs. monolithic TPU design.

* **Papy: PNM for LLM Inference**
  * LLM inference parallelism has two dimensions:
    * **Token-level parallelism (TLP)**: how many tokens are generated in parallel.
    * **Request-level parallelism (RP)**: how many concurrent requests are served.
  * Depending on TLP and RP values, the same LLM kernel can be either compute-bound or memory-bound — and these values **change at runtime**.
  * **Papy** addresses this with a **heterogeneous architecture**:
    * Compute-centric accelerator (GPU-like) for compute-bound kernels.
    * PIM engine with high memory capacity for memory-bound kernels.
    * **Runtime classification** of each kernel's TLP/RP → dynamic routing to the right accelerator.
  * Related work **Send** (ASPLOS): demonstrated LLM inference that **outperforms GPUs without using any GPU** — entirely through PIM-based execution.

* **PNM for Google Consumer Workloads**
  * Study of popular consumer applications (Chrome, TensorFlow Mobile, video playback, video capture):
    * **62.7% of total system energy** is spent on data movement between DRAM and the SoC.
  * Proposed solution: embed lightweight PIM cores or fixed-function accelerators **inside or near DRAM**.
    * Key finding: a **significant fraction of data movement** comes from **simple, repetitive functions** — perfect targets for lightweight in-DRAM logic.
    * Trade-off: fixed-function accelerators offer better efficiency for specific kernels; PIM cores offer flexibility for diverse kernels.
  * Result: significant performance improvement and energy reduction for all tested consumer workloads.


---
### 6. 🔬 PNM for Bioinformatics: Genome Sequence Analysis

> Genome analysis workloads are simultaneously compute-intensive and data-movement-intensive — prior accelerators addressed only the compute side, often making data movement worse. Moving the compute to where data resides (inside SSDs) eliminates both bottlenecks simultaneously.

* **Read Mapping: The GenStore Approach**
  * **Problem**: read mapping aligns short DNA fragments ("reads") to a reference genome by computing **edit distance** — how many insertions, deletions, or substitutions separate two strings.
    * Computation cost: alignment algorithms are quadratic in read length.
    * Data movement cost: reads and reference segments must be fetched from storage for every alignment.
  * **Prior approaches**: heuristic algorithms and compute accelerators reduced computation overhead but **increased data movement** (more data had to be examined to compensate for lower per-comparison accuracy).
  * **GenStore key idea**: place a **filter inside the SSD** that eliminates the vast majority of reads before they ever leave storage.
    * **Filter 1 — Exact matches**: reads that perfectly match the reference (detected by XOR = 0) need no alignment → discarded immediately.
    * **Filter 2 — No-match reads**: reads with no plausible alignment location (e.g., from contaminating species) → discarded with high confidence using lightweight in-storage hashing.
  * Only the small fraction of reads surviving both filters are transferred to the host for full edit-distance computation.
  * Result: **both** computation and data movement overhead reduced simultaneously — unlike prior work that traded one for the other.

* **Metagenomics at Scale: MEGAS**
  * **Metagenomics**: sequence all DNA in a mixed sample (e.g., ocean water, human gut) and identify which species are present and in what proportions.
    * Requires comparing millions of reads against a database containing thousands of reference genomes.
    * Data volume: tens to hundreds of GB per experiment.
  * **MEGAS** (Metagenomics in-Storage System): first end-to-end in-storage metagenomics pipeline.
  * **Computation partitioning**:
    * **Step 1 — Input preparation**: simple pre-processing → executed on the host (low data movement cost).
    * **Step 2 — Presence/absence identification**: database lookups for each read → executed **inside the SSD** (massive data movement avoided).
    * **Step 3 — Abundance estimation**: frequency counting across identified species → executed **inside the SSD**.
  * Design principles:
    * **Storage-aware algorithms**: redesigned to minimize writes to flash (flash has limited write endurance — excess writes rapidly wear out the SSD).
    * **Hardware-software co-design**: carefully coordinated pipeline between host and PIM-enabled SSD minimizes inter-component communication overhead.
  * **Results** vs. prior metagenomics tools:
    * Prior tools face a hard accuracy–performance trade-off: fast tools (e.g., Kraken2) sacrifice accuracy; accurate tools (e.g., MetaPhlAn) sacrifice performance.
    * MEGAS breaks this trade-off: **higher accuracy AND higher performance** than both baselines.
    * A low-cost MEGAS SSD **outperforms** accuracy-optimized and performance-optimized baselines running on high-cost systems with 16× more DRAM and much higher memory bandwidth.


---
### 7. 🛠️ Tools and Infrastructure for PIM Research

> PIM research requires specialized tools that do not exist in mainstream toolchains. Three open-source tools — D-Move (workload characterization), Ramulator 2.0 (DRAM simulation), and MQC (SSD simulation) — form the foundation for evaluating whether PIM is beneficial and how to design it.

* **D-Move: Characterizing Data Movement Bottlenecks**
  * **Problem**: before building a PIM accelerator, you must know whether a workload is actually bottlenecked by data movement — and if so, which kind.
  * **D-Move** profiles applications and classifies memory bottlenecks using:
    * **Temporal locality (LFMR)**: how often the same data is reused.
    * **Misses per kilo instruction (MPKI)**: how frequently cache misses occur.
    * **Arithmetic intensity**: ratio of computation to data moved.
  * Output: classification of each workload kernel into categories — some benefit from PIM, others from increasing cache size, others from prefetching.
  * Available open-source on GitHub with a complete toolchain.

* **Ramulator 2.0: Modular DRAM and PIM Simulation**
  * **Ramulator** is a long-established cycle-accurate DRAM simulator used in hundreds of research papers.
  * **Ramulator 2.0** adds:
    * **Modular architecture**: easier to extend with new DRAM standards, timing models, and scheduling policies.
    * **PIM-specific extensions**: models for in-DRAM compute operations, near-memory logic layers, and PIM command dispatch.
  * Enables fair, controlled comparison of PIM designs against CPU/GPU baselines under identical workloads.

* **MQC: Modern SSD Simulator**
  * No existing simulator could accurately model the internal architecture of modern high-end SSDs (complex flash controllers, multi-level caching, parallel flash channels).
  * **MQC** was developed specifically to enable in-storage PIM research (e.g., MEGAS evaluation).
  * Models realistic SSD behavior: flash channel parallelism, internal DRAM buffer, garbage collection, wear leveling.


---
### 8. ⚠️ PIM Adoption Challenges

> PIM has compelling benefits but faces significant adoption barriers spanning the entire computing stack. Programmability is the most immediate bottleneck; coherence, virtual memory, and security are deeper architectural challenges that require new fundamental solutions.

* **Programmability**
  * The most immediate adoption barrier — without good programming tools, even great hardware goes unused.
  * Lesson from GPU history:
    * Early GPGPU programming required manually mapping matrix operations to graphics shader programs — extremely tedious.
    * **NVIDIA** recognized this usage pattern and built **CUDA**, making GPU programming accessible to non-graphics programmers.
    * GPU adoption exploded after CUDA was released.
  * PIM needs an equivalent: high-level programming models, compiler support, and runtime systems that hide PIM complexity.
  * **Simple PIM framework** (for UPMEM hardware):
    * Provides iterators (map, reduce, zip) and collective primitives (broadcast, scatter, gather).
    * OpenMP applications can be reimplemented for PIM with **up to 6× fewer lines of code**.
    * Performance matches or exceeds hand-optimized implementations for histogram, reduction, and linear regression.
  * **GPU-PIM hybrid systems**:
    * Compute-bound layers → offloaded to main high-performance GPU.
    * Memory-bound layers → offloaded to simpler GPU embedded in 3D-stacked memory (lower power, constrained by thermal limits).
    * Thermal and power constraints inside the memory stack are key design limitations.

* **PIM-Enabled Instructions (PI): A Simple Adoption Path**
  * A pragmatic approach to PIM adoption that requires **no changes to the programming model or virtual memory**.
  * **Core ideas**:
    * Extend the host ISA with new **PIM instructions (PIs)** that operate on a **single cache block**.
    * Each PI is cache-coherent and virtually addressed — it looks like a regular instruction to the programmer.
    * At runtime, a hardware predictor **dynamically decides** whether to execute the PI on the host CPU or offload it to a near-memory compute unit (PCU).
  * Why single cache block?
    * A cache block is always in one memory module → no cross-module coherence complexity.
    * Analogous to atomic instructions, which are also restricted to single cache blocks.
  * **Execution comparison for a graph update operation**:

    | Approach | Data transferred | Where computed |
    |---|---|---|
    | CPU (cache miss) | 64 bytes in + 64 bytes out | Host processor |
    | PI offloaded to PIM | ~8 bytes (instruction only) | Near-memory PCU |

  * **Results across 10 data-intensive workloads**:
    * Large input sets: **47% average speedup**, **25% energy reduction**.
    * Small input sets: **32% average speedup** (smaller benefit — data fits in cache, PIM less useful).
  * **Limitation**: single cache block restriction prevents exploiting bulk PIM operations — leaves performance on the table.

* **Cache Coherence**
  * In a PIM system, both the host processor and the PIM engine operate on the **same physical memory**.
  * Changes made by one must be visible to the other — standard cache coherence requirement, now extended to PIM engines.
  * Naive approaches (page-level non-cacheability, page-level locks) lose most of the performance benefit.
  * **Lazy PIM** and **Condu**: research proposals that provide fine-grained coherence with near-ideal PIM performance — covered in later lectures.

* **Virtual Memory**
  * One of the hardest unsolved challenges in PIM.
  * Current commercial PIM (e.g., UPMEM 2019): **no virtual memory support**.
    * Data must be manually copied between host address space and PIM address space.
    * This manual data movement largely negates the benefit of PIM.
  * Full virtual memory support in PIM requires resolving address translation, TLB management, and page fault handling — all in a near-memory context.

* **Synchronization**
  * Multiple PIM engines operating concurrently require synchronization primitives (locks, barriers) — analogous to parallel programming on multi-core CPUs.
  * **Synchron**: a research proposal for PIM-aware lock design — covered in later lectures.

* **Security**
  * PIM raises both positive and negative security implications:
    * **Positive**: PIM can accelerate **homomorphic encryption** (Cipher-Match), enabling private computation on untrusted servers.
    * **Negative**: PIM introduces new **side-channel and covert-channel attack surfaces**.
      * In traditional systems, memory is physically distant from the processor and protected by cache hierarchies.
      * With PIM, computation offloaded to memory can leak information through timing, power, or electromagnetic side channels.
      * The **Impact** project (published at DSN) explored this threat in detail.

* **Summary of Adoption Barriers and Solutions**

  | Challenge | Status | Key Research |
  |---|---|---|
  | Programmability | Partial solutions exist | Simple PIM, CUDA-like frameworks needed |
  | Cache coherence | Research proposals available | Lazy PIM, Condu |
  | Virtual memory | Largely unsolved | Active research area |
  | Synchronization | Addressed in research | Synchron |
  | Security | Both opportunities and new threats | Impact, Cipher-Match |
  | Infrastructure/tools | Open-source tools available | D-Move, Ramulator 2.0, MQC, DRAM Bender |
