# Lecture 4: Memory-Centric Computing I — Processing Using DRAM


> This lecture introduces **memory-centric computing (Processing-in-Memory, PIM)** as the architectural response to the data movement bottleneck. It distinguishes two fundamental PIM paradigms: **processing near memory** (adding logic close to DRAM) and **processing using memory** (exploiting DRAM's inherent analog properties for computation). The lecture then dives deep into two concrete techniques — **RowClone** for bulk data copy and **Ambit** for bitwise computation — showing how existing DRAM can be leveraged for dramatic performance and energy gains by carefully violating timing parameters. Experimental results on commodity DRAM chips confirm that these capabilities exist in hardware today, even without any design modifications.


---
### 1. 🧠 Why Processing-in-Memory? Motivation and History

> The data movement bottleneck has worsened dramatically over six decades due to an asymmetric scaling trajectory: logic (CPU) scaled well under Dennard scaling, but memory and interconnects did not. The result is that moving data now costs far more energy than computing on it — making a fundamental architectural rethink necessary.

* **The Core Problem PIM Solves**
  * As established in previous lectures, data movement is the dominant cost in modern computing:
    * Moving data from DRAM to the CPU costs **~800× more energy** than a 64-bit floating-point multiply.
    * Processors spend **50–70% of cycles stalled** waiting for memory.
    * Data movement energy **dominates computation energy** across virtually all major workloads.
  * PIM addresses this by moving computation **to where data already lives**, reducing or eliminating data movement entirely.
  * PIM simultaneously improves **performance**, **energy efficiency**, **sustainability**, and can reduce hardware costs.

* **Historical Context**
  * The idea of placing logic near memory is not new:
    * **1960s**: Early researchers (including K.Z. and Harold Stone at IBM) proposed integrating computation with memory for cost and efficiency reasons.
    * These ideas were largely **forgotten in the 1990s** as the CPU-centric model dominated.
    * **Post-2010**: Dramatic worsening of the data movement bottleneck — driven by data-intensive workloads like ML, graph analytics, and genomics — triggered a major revival of PIM research.
  * What changed between the 1960s and today:
    * **Logic scaling (Dennard scaling)** made transistors faster and cheaper — benefiting CPUs greatly.
    * **Memory and interconnect scaling** did not keep pace — the latency and energy gap between compute and memory grew continuously.
    * **New applications** (ML, databases, genomics) are far more data-intensive than workloads of the 1960s.
    * **Sustainability** has become a critical design constraint — energy waste from data movement is no longer acceptable.

* **Additional Drivers for PIM Today**
  * DRAM reliability problems (RowHammer, RowPress, VRT) can be **mitigated** by integrating computation into memory — in-DRAM logic can perform protective operations without involving the host CPU.
  * Emerging memory technologies (e.g., PCM, STT-MRAM) can perform **analog matrix-vector multiplication** as a natural byproduct of their electrical properties — a capability discovered only in the 2010s.
  * Industry is actively adopting PIM: Samsung, SK Hynix, Micron, Alibaba, and others are shipping or developing PIM-capable chips.


---
### 2. 🗂️ Taxonomy: Two Fundamental PIM Paradigms

> PIM can be approached in two fundamentally different ways. Processing near memory adds conventional logic physically close to DRAM. Processing using memory exploits DRAM's own analog characteristics to perform computation — no extra logic required, but operations are limited by what the memory device can naturally do.

* **Processing Near Memory**
  * Logic and memory are **designed and manufactured separately**, then integrated closely:
    * Compute units (CPUs, GPUs, NPUs) and DRAM use incompatible fabrication processes — they cannot be built on the same die with current technology.
    * "Near" means: inside the memory controller, next to DRAM banks or subarrays, or in a logic layer of a 3D-stacked memory.
  * Key enabler: **3D stacking technology**
    * Memory dies are stacked vertically on top of a logic die using through-silicon vias (TSVs).
    * The logic layer sits physically close to all memory layers, enabling **high bandwidth** and **low-latency** access.
    * Example: **High Bandwidth Memory (HBM)** stacks used in modern GPUs.
  * Advantages:
    * Logic is designed freely — full-featured processors, FPGAs, or custom accelerators can be placed near memory.
    * Easier to adopt because it does not require changing DRAM cell design.
  * Disadvantage: thermal challenges — stacked layers trap heat, limiting capacity and clock speed.

* **Processing Using Memory**
  * Computation is performed by the **memory device itself**, exploiting its inherent analog electrical properties — no extra logic is added.
  * Examples:
    * **Charge sharing** between DRAM cells via bit lines enables bulk data copy (RowClone).
    * **Concurrent activation** of multiple DRAM rows causes sense amplifiers to compute majority functions (Ambit).
    * **Resistance-based memories** (PCM, ReRAM) can naturally perform analog multiply-accumulate operations used in neural network inference.
  * Advantages:
    * No additional logic area or fabrication cost.
    * Operates at DRAM density and bandwidth — potentially processing an entire row (8 KB) in a single operation.
  * Disadvantages:
    * Limited to operations that the memory's physical structure can naturally support.
    * Typically restricted to bulk bitwise operations in DRAM; more complex operations require multiple steps.

* **Combined Approaches and Broader Taxonomy**
  * The two paradigms are not mutually exclusive — they can be combined:
    * Processing using memory provides raw bitwise compute capability; nearby logic can orchestrate, control, and extend it.
  * PIM applies across multiple memory technologies:

    | Technology | PIM Approach | Example Use |
    |---|---|---|
    | DRAM | Using memory (charge sharing, triple row activation) | Bulk copy, bitwise operations |
    | DRAM | Near memory (3D-stacked logic layer) | General-purpose compute near HBM |
    | SRAM | Near memory (logic next to cache banks) | Cache-resident workload acceleration |
    | Flash / Storage | Near memory (computational storage) | Data filtering before host transfer |
    | PCM / ReRAM | Using memory (analog resistance) | In-memory neural network inference |

  * **FPGAs** are a special case: their lookup tables (LUTs) are small memory structures that **store precomputed function outputs** indexed by input values — a direct form of processing using memory.


---
### 3. 🏭 Industry Adoption and Real-World PIM Prototypes

> PIM has moved from academic concept to real hardware. Multiple commercial and prototype PIM chips exist today, spanning general-purpose processors embedded near DRAM banks, FPGA-based memory-side accelerators, and standardized interconnects that expose PIM to the host system.

* **Commercial and Research PIM Chips**
  * **UPMEM PIM Engine** (French startup, 2019):
    * A real, available DRAM chip with a **general-purpose processor (DPU) embedded next to each DRAM bank**.
    * Supports standard C programming — programmers can offload computation to DPUs explicitly.
    * Available in research labs for real workload evaluation.
  * **Samsung, SK Hynix, Alibaba**: all have designed DRAM chips with processing logic next to each bank, with published research results.
  * **SK Hynix HBM-PIM**: places a programmable compute unit inside HBM stacks used in AI accelerators.
  * **Micron Automata Processor**: an early PIM design that performed deterministic finite automaton (DFA) computation inside DRAM — used for pattern matching workloads.
  * **FPGA-near-memory prototypes**: FPGAs placed adjacent to memory modules to filter and aggregate data before sending to the CPU/GPU, eliminating unnecessary data movement.

* **CXL: A Standardized PIM Interface**
  * **CXL (Compute Express Link)** is an emerging interconnect standard that allows the CPU to offload computation to memory-side devices.
  * Provides a standardized interface for attaching PIM-capable memory expanders to a host processor.
  * Enables PIM adoption without requiring changes to the CPU or memory chips themselves.

* **PIM as an Accelerator Model**
  * The most practical near-term PIM programming model treats **memory as an accelerator**:
    * The CPU remains the central orchestrator.
    * Specific operations are offloaded to the PIM device, which executes and returns results.
    * This aligns with existing programming models for GPUs and other accelerators.
  * A more ambitious long-term vision: a **distributed computing model** where CPU, memory, storage, and sensors all contribute computation equally — but this requires fundamentally new programming models and is much harder to realize.


---
### 4. 📋 RowClone: Bulk Data Copy Inside DRAM

> Bulk data copy is a surprisingly expensive and frequent operation in real systems. RowClone eliminates the need to move data out of DRAM entirely for within-DRAM copies by exploiting charge sharing between sense amplifiers and DRAM rows — achieving over 10× latency improvement and 70× energy reduction compared to traditional CPU-mediated copy.

* **Why Bulk Copy Is a Significant Problem**
  * Bulk data copy operations are far more common than expected:
    * **Page zeroing**: when the OS recycles a memory page, it must zero it to prevent data leakage to the new process.
    * **Fork/clone**: copying a process's memory when spawning a child thread or cloning a VM.
    * **Page migration**: moving pages between NUMA nodes or memory tiers.
  * Measured impact in production:
    * A **Google ISCA 2015** study found that `memcpy` and `memmove` alone account for **5% of all CPU execution cycles** across Google's data centers.
    * This 5% figure represents an enormous amount of wasted computation at scale.

* **Why Traditional Copy Is Inefficient**
  * To copy a 4 KB page using the CPU:
    * Source page is loaded **byte-by-byte into L1 cache and registers**.
    * Data is copied in registers.
    * Result is written **byte-by-byte back to DRAM**.
    * Total data movement: **12 KB** (4 KB read from source + 4 KB into registers + 4 KB write to destination).
  * Even with a **DMA engine** (which bypasses the CPU cache):
    * Data still travels the full DRAM → memory bus → memory controller → memory bus → DRAM path.
    * A 4 KB DMA page copy takes approximately **1,000 ns** and consumes **3.6 µJ**.
  * Problems: high latency, high memory bus bandwidth consumption, cache pollution.

* **RowClone: In-DRAM Copy via Charge Sharing**
  * Key insight: DRAM sense amplifiers (the **row buffer**) already hold a full row of data after activation — they can be used as a transfer intermediary.
  * Mechanism for **intra-subarray copy** (source and destination rows in the same subarray):
    * **Step 1**: Activate the source row → its data is sensed and amplified into the sense amplifiers (row buffer). Source row is also restored to full charge.
    * **Step 2**: Without pre-charging, immediately activate the destination row → the sense amplifiers (already holding the source data at full VDD) **drive the destination row** to the same values via charge coupling.
    * Result: the destination row now holds a copy of the source row's data — entirely within DRAM, with no data leaving the chip.
  * This technique violates the standard **activate-to-activate (tRRD)** timing parameter — consecutive activations are not normally permitted in standard DRAM operation.
  * Performance improvement over traditional DMA copy:

    | Metric | Traditional DMA Copy | RowClone (intra-subarray) |
    |---|---|---|
    | Latency | ~1,000 ns | ~90 ns (>10× faster) |
    | Energy | ~3.6 µJ | ~0.04 µJ (>70× less) |
    | Memory bus traffic | High | Zero (data never leaves DRAM) |

* **Experimental Validation on Real DRAM Chips**
  * A Princeton research group used the **SoftMC FPGA infrastructure** to demonstrate RowClone-like operations on unmodified commodity DRAM chips — calling the result **Compute DRAM**.
  * The lecturer's group built an **end-to-end PIM system** on a RISC-V + FPGA platform:
    * OS is modified to allocate destination pages **in the same subarray** as source pages when possible.
    * When a page copy is requested between same-subarray pages, the CPU offloads the operation to a modified memory controller.
    * The memory controller **violates the tRRD timing parameter** to perform the copy inside DRAM.
    * The memory controller signals the CPU when complete — no DRAM hardware modification needed.
    * The system also supports **in-DRAM random number generation** using similar timing violations.
  * Results: significant throughput improvements for initialization and copy, especially noticeable on a low-power RISC-V processor where the CPU is the bottleneck.

* **Copying Across Subarrays and Banks**
  * Intra-subarray RowClone is the simplest case; cross-subarray and cross-bank copies are harder:
    * **Cross-bank copy**: set source bank to read mode, destination bank to write mode, pipeline reads and writes via the internal data bus. Not supported in current chips but infrastructure is present.
    * **Cross-subarray copy**: the original RowClone paper used an indirect route (copy to another bank, then to target subarray) — inefficient.

* **LISA: Low-Cost Inter-Subarray Links**
  * To enable efficient cross-subarray data movement, the **LISA** proposal adds **isolation transistors** between adjacent subarray bit lines:
    * When enabled, adjacent subarrays' bit lines are connected — effectively creating a longer bit line that spans subarrays.
    * Data flows from one subarray to the next without leaving the DRAM chip.
  * Benefits of LISA:

    | Capability | Without LISA | With LISA |
    |---|---|---|
    | Inter-subarray copy latency | ~1.3 ms | ~0.15 ms |
    | Pre-charge latency | ~13 ns | ~5 ns (shared pre-charge) |
    | DRAM caching | Not possible | Fast subarray caches slow subarray |

  * Limitation: signal integrity limits connectivity to **2–3 adjacent subarrays** without significant reliability degradation.
  * Additional hardware cost is modest: only isolation transistors between subarray boundaries.


---
### 5. ⚡ Ambit: Bitwise Computation Using Triple Row Activation

> Ambit exploits a fundamental property of DRAM sense amplifiers: when multiple rows are activated simultaneously, the sense amplifier resolves to the majority value. By carefully controlling which rows are activated and what data they hold, Ambit implements bulk AND, OR, and NOT operations on entire DRAM rows — enabling workloads like database bitmap queries, DNA sequence matching, and encryption to run orders of magnitude faster with far less energy.

* **The Core Physical Phenomenon: Triple Row Activation (TRA)**
  * Standard DRAM activates **one row at a time** — this is a fundamental assumption of the DRAM standard.
  * Ambit intentionally **activates three rows simultaneously** connected to the same bit line and sense amplifier.
  * Physical result: the sense amplifier resolves to the **bitwise majority** of the three rows:
    * If **2 or more** of the 3 cells are charged (logic 1) → bit line is pulled above the midpoint → sense amp outputs **1**.
    * If **2 or more** of the 3 cells are discharged (logic 0) → bit line is pulled below the midpoint → sense amp outputs **0**.
  * This majority operation applies simultaneously to **every column in the row** → one TRA computes an 8 Kbit (1 KB) bitwise majority in a single DRAM operation.
  * Deriving AND and OR from majority:
    * **Bitwise AND(A, B)**: set the third row C = 0 → Majority(A, B, 0) = A AND B.
    * **Bitwise OR(A, B)**: set the third row C = 1 → Majority(A, B, 1) = A OR B.

* **Implementing Ambit in DRAM**
  * A small **dedicated compute area** is added within each subarray — a few reserved rows (e.g., D0, D1, D2):
    * These rows have a **simplified row decoder** that supports triple activation, instead of the complex decoders used for normal rows.
    * This keeps hardware overhead low.
  * To compute AND(A, B) → store result in row D:
    * RowClone A → D0
    * RowClone B → D1
    * RowClone (all-zeros row) → D2 (to set the control row to 0)
    * Triple Row Activate D0, D1, D2 → sense amplifiers compute Majority(A, B, 0) = AND(A, B)
    * RowClone result → D

* **The NOT Operation**
  * Sense amplifiers are built from **cross-coupled inverters** — the complement of every sensed bit is naturally available on the **opposite side** of the sense amplifier.
  * Ambit adds a **dual contact cell** and additional wiring to expose this complemented value into a reserved row within the subarray.
  * This makes NOT available without additional logic circuits.
  * With NOT, AND, and OR, Ambit is **functionally complete** — any Boolean function can be implemented.

* **Performance of Ambit**
  * NOT operation on an 8 KB row: **Ambit is ~31× faster** and uses **~31× less energy** than a CPU performing the same bitwise negation.
  * AND/OR operations show similar bulk throughput advantages.
  * All improvements scale with row width — wider rows mean more bits computed per operation.

* **Workloads Naturally Suited to Ambit**
  * PUM operations are **bulk bitwise** — they process an entire row at once. Workloads that can be expressed as bitwise operations on large arrays benefit most:
    * **Database bitmap indices**: attributes (age, location, income) are stored as bitmaps; queries are AND/OR operations across bitmaps.
      * Example query: "Find users in Zurich, aged 25–35, studying EE" → bitwise AND of three bitmaps.
      * Systems like **BitWeaving** (database) and **BitFunnel** (Microsoft web search) are designed around this principle.
    * **DNA sequence mapping**: many alignment algorithms reduce to bitwise operations on long binary strings.
    * **Encryption**: AES, SHA, and many other ciphers are dominated by XOR, AND, and shift operations.
    * **Set operations**: union and intersection of large sets represented as bit vectors.
  * Simulations of real database workloads show **10–100× speedups** when ported to the Ambit substrate.

* **Compiler Framework for Ambit**
  * To make Ambit programmable without manual assembly:
    * Users specify a desired block bitwise operation in standard hardware logic notation.
    * This is compiled into a **Majority-Inverter Graph (MIG)** — a graph representation using only majority and NOT gates.
    * EDA tools (adapted from superconducting logic design) optimize the MIG for minimum depth (lower latency).
    * The optimized MIG is translated into a **microprogram** of basic DRAM commands (RowClone, TRA, NOT, activate, precharge).
    * The microprogram is stored in the memory controller and dispatched when the corresponding instruction is issued.
  * Limitation: **bit-serial arithmetic** (e.g., addition, multiplication) has quadratic latency with bit width — best suited for **low bitwidth operations** (2–8 bits).

* **Transparent Compiler Support: Fine-Grain PUM**
  * Current Ambit is a wide **SIMD** machine — all columns of a row perform the same operation simultaneously.
  * Problem: if application parallelism is narrower than the row width (e.g., only 512 bits needed, but the row is 65,536 bits wide), most of the row is wasted.
  * Solution — **fine-grain DRAM access**:
    * Segment the global word line and add isolation transistors between **DRAM mats** (small internal arrays within a subarray).
    * Enable only the necessary mats → different mats can execute **different instructions on different data** simultaneously.
    * This transforms Ambit from a SIMD engine into a **MIMD (Multiple Instruction, Multiple Data)** model within DRAM — effectively a small multiprocessor.
    * A DRAM mat (e.g., 512 columns) aligns naturally with modern CPU vector widths (e.g., Intel AVX-512 = 512 bits), enabling seamless integration.
  * **LLVM compiler passes** (developed for this framework):
    * **Pass 1**: automatic code identification — detect which code regions are amenable to PUM execution.
    * **Pass 2**: auto-vectorization — express identified regions as vector operations matching DRAM mat width.
    * **Pass 3**: code scheduling and data mapping — assign operations to DRAM mats, handle data layout and alignment.
  * Remaining challenges: data transposition (operating on data column-wise rather than row-wise), cache coherence, address translation, and data allocation alignment.


---
### 6. 🔬 Experimental Validation: PUM on Commodity DRAM Chips

> All prior Ambit and RowClone work was based on circuit simulation. Recent experiments on unmodified commodity DRAM chips from all three major manufacturers confirm that these operations can be triggered in real hardware by carefully violating timing parameters — without any hardware modifications. This demonstrates that PIM capability already exists latently in today's DRAM.

* **Key Experimental Findings (2024)**
  * Using FPGA-based memory testing infrastructure, researchers tested PUM operations on commodity chips from **Samsung, SK Hynix, and Micron** by deliberately violating standard timing parameters.
  * Results for each operation:

    | Operation | Method | Success Rate |
    |---|---|---|
    | Multi-row copy (1 → up to 31 rows) | Consecutive activations violating tRRD | ~99.98% (0.02% bit failures) |
    | NOT | Activate source → capture complement via destination row activation | High |
    | AND, NAND, OR, NOR (2-input) | Concurrent activation across neighboring subarrays | Variable (~50–94%) |
    | 16-input AND/OR | Extended multi-row concurrent activation | Up to 94% |

* **How Multi-Row Copy Works in Real Chips**
  * Activate a source row → data is loaded into sense amplifiers.
  * Immediately activate up to 31 destination rows **without pre-charging** between activations (violating tRRD).
  * Hypothesis for why this works: the **hierarchical row decoder** circuitry does not settle fully when timing is violated, causing multiple word lines to remain asserted simultaneously → all destination rows are driven by the sense amplifiers.
  * Practical result: one-to-many copy with near-perfect reliability, dramatically accelerating initialization and cloning operations.

* **How AND/OR Operations Work Across Subarrays**
  * Activate two rows (A, B) in one subarray and two rows (X, Y) in a **neighboring subarray that shares sense amplifiers** — all four activated simultaneously by violating timing parameters.
  * The shared sense amplifier compares the **average voltage of A+B** against **average voltage of X+Y**:
    * If both X and Y are logic 1 → average of X+Y exceeds a reference threshold (3·VDD/4) → sense amp outputs 1 → **AND(X, Y) = 1**.
    * NAND is simultaneously computed on the reference subarray side.
  * OR and NOR functions are implemented with simple modifications to the reference voltage or row selection.

* **Implications of These Findings**
  * DRAM chips already contain **latent PIM capability** — it simply isn't exposed through the standard interface.
  * If chips were **intentionally designed** to support these operations (reliable timing, dedicated compute rows, explicit commands), reliability and performance would be far higher.
  * Some manufacturers add circuitry to **prevent timing parameter violations** — driven by reliability margins, competitive secrecy, and fear of exposing RowHammer-like vulnerabilities. After seeing these results, several manufacturers expressed interest in intentionally supporting these operations.
  * These experiments provide the strongest evidence yet that **processing using DRAM is physically realizable**, not merely a theoretical proposal.
