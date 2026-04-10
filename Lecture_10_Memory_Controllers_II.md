# Lecture 10: Memory Controllers II — Scheduling, Self-Optimization, and QoS


> This lecture examines the DRAM memory controller as a complex scheduling problem with over 100 timing constraints, multiple resource conflicts, and competing optimization objectives. It introduces **self-optimizing controllers** that use reinforcement learning to automatically discover scheduling policies that outperform human-designed heuristics. It then proposes **self-managing DRAM** — a disruptive architectural concept where the DRAM chip autonomously handles its own maintenance (refresh, RowHammer mitigation) using a simple Nack signal, bypassing the slow JEDEC standardization process. Finally, it addresses **memory interference** in multi-core systems, showing how uncontrolled sharing leads to severe unfairness and denial-of-service vulnerabilities, and presents two scheduling solutions — **STFM** (fairness via slowdown equalization) and **ParBS** (performance via bank parallelism preservation).


---
### 1. 🗂️ DRAM Scheduling: Policies and Challenges

> The memory controller must continuously translate incoming cache miss requests into sequences of DRAM commands (activate, read, write, pre-charge) while obeying over 100 timing constraints and managing conflicts across banks, buses, and channels. The scheduling decisions made here have large consequences for both performance and fairness.

* **Basic Scheduling Policies**
  * **FCFS (First-Come, First-Served)**:
    * Service requests strictly in arrival order.
    * Simple, fair by arrival time, but ignores DRAM structure → poor performance.
  * **FR-FCFS (First-Ready, First-Come, First-Served)**:
    * **Priority 1**: Requests that are **row hits** (target an already-open row in the row buffer) → served immediately with column access command.
    * **Priority 2**: Among non-row-hit requests, older requests served first (FCFS).
    * **Goal**: Maximize row buffer hit rate → maximize DRAM throughput and minimize energy (avoiding repeated activate/pre-charge cycles).
    * Implementation: prioritize **column commands** (read, write) over **row commands** (activate, pre-charge) at the command level.

* **What Can Be Prioritized (Scheduling Dimensions)**
  * Modern memory controllers can consider multiple attributes when prioritizing requests:

    | Dimension | Example Policy |
    |---|---|
    | **Request age** | FCFS — oldest first |
    | **Row buffer status** | FR-FCFS — row hit first |
    | **Request type** | Read > Write (load miss > store miss) |
    | **Prefetch vs. demand** | Defer prefetches when demand requests pending |
    | **Core stall proximity** | Prioritize requests from nearly-stalled cores |
    | **Bus direction** | Group reads together; group writes together (minimize bus turnaround) |

  * **Load miss vs. store miss distinction**:
    * A **load miss** blocks the processor immediately — the load result is needed before execution continues → highest priority.
    * A **store miss** only needs to fetch a cache line to enable the write; the processor can often continue → lower priority, can be deferred.
  * **Request criticality in GPUs**:
    * GPU streaming multiprocessors (SMs) with fewer remaining active threads have less parallelism to hide latency → their requests should be prioritized to prevent the SM from fully stalling.

* **Row Buffer Management Policies**
  * After serving a request, should the activated row stay open or be closed?
  * **Open-row policy**: leave row open → next access may be a row hit (good if spatial locality is high).
  * **Closed-row policy**: pre-charge immediately → avoids row conflict penalty if next access targets a different row.
  * **Adaptive policy**: predict based on access pattern whether to keep the row open or close it.
  * **Security interaction**: keeping a row open too long increases RowHammer risk — RowPress exploits prolonged row activation to reduce the hammer count needed for bit flips (as few as one access while open).

* **Why DRAM Scheduling Is Hard**
  * Over **100 timing parameters** must be respected simultaneously:
    * **tWTR**: minimum cycles between a write command and a subsequent read (bus direction change).
    * **tRC**: minimum cycles between consecutive activates to the same bank.
    * **tRRD**: minimum cycles between activates to different banks in the same rank.
    * ... and many others, all enforced simultaneously.
  * Resources to track: channels, ranks, bank groups, banks, row buffers, address bus, data bus.
  * Reordering requests to improve performance (FR-FCFS) introduces **fairness and QoS problems** (Section 4).
  * Heterogeneous request sources (CPU threads, GPU, I/O DMA) each have different latency sensitivity and bandwidth requirements.
  * DRAM datasheets list timing values without explaining the underlying physics → understanding *why* each constraint exists requires reading research papers, not just specs.


---
### 2. 🤖 Self-Optimizing Memory Controllers via Reinforcement Learning

> Human-designed scheduling heuristics are rigid — written once and deployed in hardware, unable to adapt to new workloads or changing system conditions. Reinforcement learning offers an alternative: a controller that continuously improves its own scheduling policy by observing outcomes and updating its action-value estimates.

* **The Limitation of Hand-Designed Policies**
  * Traditional DRAM controllers implement fixed heuristics (FR-FCFS, oldest-first, etc.) chosen by designers years before deployment.
  * These heuristics:
    * Cannot adapt to workloads that were not present when the policy was designed.
    * Cannot respond to changing system state (e.g., varying memory intensity, mixed CPU/GPU workloads).
    * Require manual re-engineering for each new DRAM standard or application domain.

* **Reinforcement Learning (RL) Framework for Memory Controllers**
  * Map the memory scheduling problem directly to the RL framework:
    * **Agent**: the memory controller.
    * **Environment**: the DRAM chip + processor pipeline + running applications.
    * **State**: observable system metrics (queue depths, row buffer hit rate, core stall signals, reorder buffer occupancy, etc.).
    * **Actions**: DRAM commands to issue next (activate, read, write, pre-charge, or no-op).
    * **Reward**: a metric reflecting scheduling quality — e.g., +1 for each read or write command issued (maximizing data bus utilization).
  * The agent learns a **Q-table** mapping (state, action) pairs to expected long-term reward.
  * At each decision point: select the action with highest Q-value for the current state.
  * After observing the resulting reward: update the Q-value using the **Bellman equation**.
  * The policy continuously improves as the controller accumulates experience.

* **State Attributes and Actions (ISCA 2008 Implementation)**
  * **State examples**:
    * Number of read, write, and load-miss requests in the transaction queue.
    * Number of pending write requests.
    * Number of reorder buffer (ROB) entries waiting for a specific memory reference (proxy for request criticality).
    * Relative age of the oldest request in the queue.
  * **Action examples**:
    * Activate (open a new row).
    * Read (load miss — highest criticality).
    * Read (store miss — lower criticality).
    * Write.
    * Pre-charge (close a row).
    * **No-op** — do nothing this cycle (important when no action is beneficial; prevents wasted commands).

* **Evaluation Results**
  * Self-optimizing controllers achieve **significantly better performance** than many human-designed policies tested on the same workloads.
  * The improvement is **robust across diverse workloads** — the RL policy generalizes rather than overfitting to one scenario.

* **Advantages and Limitations**

  | Aspect | Detail |
  |---|---|
  | **Advantage: Continuous learning** | Policy adapts to new workloads over time without redesign |
  | **Advantage: Reduced designer burden** | Designer specifies state variables and reward function — not the policy itself |
  | **Limitation: Reward design** | Choosing a good reward function is non-trivial; wrong reward → wrong behavior |
  | **Limitation: Q-table size** | Large state spaces require large tables → area overhead; mitigated by quantization |
  | **Limitation: Speed** | DNN-based RL too slow for per-cycle decisions; Q-learning table lookup fast enough |
  | **Limitation: Explainability** | Cannot easily understand *why* the policy works → hard to debug and verify |
  | **Limitation: Test coverage** | No clear way to enumerate all scenarios to validate ML-based policies |

* **Broader Vision: Self-Optimizing Architectures**
  * The same RL approach applies to: cache controllers, prefetchers, storage controllers, network schedulers.
  * Multi-agent RL: coordinate multiple controllers (memory + cache + prefetcher) jointly.
  * Long-term vision: **data-driven, self-optimizing computer architectures** — systems that continuously improve without manual re-engineering.


---
### 3. 🔄 Self-Managing DRAM (SMD)

> All DRAM maintenance operations (refresh, RowHammer mitigation, ECC scrubbing) are currently dictated by the memory controller — but the DRAM chip itself has all the information needed to perform these operations autonomously. SMD delegates maintenance to the DRAM chip via a simple Nack mechanism, enabling rapid innovation without waiting for JEDEC standardization.

* **The Problem: Processor-Centric DRAM Interface**
  * The current DRAM interface is a **one-way command channel**: the memory controller sends commands; the DRAM chip executes them.
  * The DRAM chip has **no voice** — it cannot initiate actions, report internal state, or refuse commands.
  * All DRAM maintenance (when to refresh, which rows to protect from RowHammer, how to perform ECC scrubbing) is orchestrated by the memory controller, which has **no visibility** into the DRAM's internal state.
  * This means the memory controller must implement and manage all maintenance logic — with parameters that must be standardized in JEDEC specs.

* **The JEDEC Standardization Bottleneck**
  * Every new maintenance mechanism requires modifying the DRAM interface standard → JEDEC consensus among 390+ member companies:

    | Transition | Time Required |
    |---|---|
    | DDR3 → DDR4 | 5 years |
    | DDR4 → DDR5 | 8 years |

  * Even maintenance techniques that are technically ready may wait years before they can be deployed.
  * Example: same-bank refresh, refresh management, and DRAM ECC scrubbing (now in DDR5) were technically viable years before they were standardized.

* **SMD Core Idea: Separate Access from Maintenance**
  * **Observation**: DRAM operations fall into two natural categories:
    * **Access operations** (activate, read, write, pre-charge): serve memory requests. Information needed (address, data) is available to the memory controller.
    * **Maintenance operations** (refresh, RowHammer protection, ECC scrubbing): maintain data integrity. Information needed (per-cell retention time, activation counts, error locations) is local to the DRAM chip.
  * Since maintenance information is **local to the DRAM chip**, the chip is better positioned to perform maintenance — not the controller.

* **The Nack Mechanism**
  * SMD adds a single simple interface change: the DRAM chip can send a **Negative Acknowledgment (Nack)** to the memory controller.
  * **Protocol**:
    * Memory controller sends a command (activate or read/write).
    * If the DRAM chip is busy with internal maintenance on that subarray, it responds with **Nack**.
    * Memory controller receives Nack → retries the command after a brief delay.
    * Once maintenance is complete, subsequent commands are accepted normally.
  * **Implementation options**:
    * Add a dedicated Nack output pin (minimal silicon cost).
    * Repurpose an existing signal line (e.g., an alert pin already present in some DRAM standards).
  * **Granularity**: Nack can be at the subarray level — only commands targeting the subarray currently under maintenance are rejected; other subarrays remain available.

* **Benefits of SMD**
  * **Innovation velocity**: DRAM manufacturers can implement new maintenance algorithms without changing the DRAM interface standard — only the internal chip logic changes.
  * **Better maintenance**: The DRAM chip can use its internal knowledge (exact per-cell retention times, precise activation counts, error maps) to perform maintenance optimally — far more accurately than a controller that cannot see inside the chip.
  * **Performance**: Maintenance overlaps with access operations targeting other subarrays → lower effective overhead.
  * **Architectural shift**: transforms the DRAM interface from a one-way command channel to a **request-reply equal-partner relationship**.

* **Comparison: Current Interface vs. SMD**

  | Aspect | Current DRAM | Self-Managing DRAM |
  |---|---|---|
  | Who decides when to refresh | Memory controller | DRAM chip |
  | Who manages RowHammer mitigation | Memory controller | DRAM chip |
  | Interface type | One-way commands | Two-way request-reply |
  | New maintenance mechanism | Requires JEDEC standard update | Implemented inside chip, no interface change |
  | Innovation cycle | 5–8 years | Immediate (per chip generation) |


---
### 4. ⚠️ Memory Interference in Multi-Core Systems

> In multi-core systems, all threads share the same DRAM channel, and standard scheduling policies like FR-FCFS are blind to thread identity. This creates severe unfairness: threads with high row buffer locality monopolize the memory bus, starving other threads and enabling denial-of-service attacks.

* **Why Uncontrolled Sharing Is Dangerous**
  * The memory controller treats all requests from all threads identically — it does not know which thread a request belongs to or how critical it is.
  * Scheduling policies optimized for throughput (FR-FCFS) inadvertently **favor specific access patterns**:
    * Threads with **streaming, sequential access patterns** → high row buffer hit rate → continuously prioritized by FR-FCFS → low latency, high throughput.
    * Threads with **random access patterns** → low row buffer hit rate → continuously de-prioritized → high latency, low throughput.

* **Quantified Example: Stream vs. Random**
  * Microbenchmark with two simultaneous threads:
    * **Stream thread**: 96% row buffer hit rate.
    * **Random thread**: 3% row buffer hit rate.
  * Under FR-FCFS: the stream thread monopolizes the row buffer. The random thread is effectively starved.
  * Real-world case — MATLAB (streaming) co-running with GCC (random):
    * MATLAB slowdown when co-running: minor.
    * GCC slowdown when co-running with MATLAB: **~3×** — a severe, unpredictable degradation.

* **Consequences of Uncontrolled Interference**
  * **Unpredictable performance**: a thread's execution time depends on what else happens to be running — invisible to the programmer.
  * **Poor user experience**: video playback stalls, interactive response degrades.
  * **SLA violations**: cloud providers cannot meet service-level agreements when tenant performance varies unpredictably.
  * **Security vulnerability**: a malicious tenant can deliberately run a streaming workload to degrade other tenants' performance → **memory-based denial-of-service (DoS) attack**.
  * **Core underutilization**: compute-bound applications generate few but critical memory requests. If those requests are queued behind streaming requests, the core sits idle — total system throughput drops.
  * **Interconnect interference**: the same phenomenon occurs in the network-on-chip (NoC) — aggressive threads can saturate interconnect links, starving other cores.


---
### 5. ⚖️ QoS-Aware Scheduling: STFM and ParBS

> Two complementary scheduling algorithms address the interference problem from different angles. STFM focuses on fairness — equalizing slowdowns across threads. ParBS focuses on performance — preserving intra-thread bank parallelism that naive interleaving destroys.

* **STFM: Stall-Time Fair Memory Scheduling**
  * **Goal**: threads running together should each experience a similar slowdown relative to running alone — equal-priority threads get proportional service.
  * **Key metrics**:
    * **Stall time alone**: how long a thread would wait for DRAM if running in isolation (not directly measurable at runtime → must be estimated).
    * **Stall time shared**: how long the thread actually waits when co-running (directly measurable).
    * **Memory slowdown** = stall time shared / stall time alone.
    * **Unfairness** = max(slowdown across threads) / min(slowdown across threads). Ideal = 1.0.
  * **Policy**:
    * If unfairness < threshold α → use a throughput-oriented policy (FR-FCFS) to maximize bandwidth.
    * If unfairness ≥ threshold α → switch to fairness mode: prioritize requests from the **most-slowed-down thread** until the maximum slowdown is reduced below α.
    * The system oscillates between throughput and fairness modes as needed.
  * **Example timeline**:
    * Initially, FR-FCFS favors Thread 0 (streaming) → Thread 1's slowdown grows.
    * Once unfairness threshold is crossed → Thread 1's requests are prioritized until its slowdown recovers.
    * System reverts to FR-FCFS → cycle repeats.
  * **Strengths**: directly targets fairness; improves system performance by preventing extreme starvation and ensuring all cores make progress.
  * **Weaknesses**: stall-time-alone estimation is approximate → policy may be imprecise; complex to implement; does not address all interference sources.

* **ParBS: Parallel-Aware Batch Scheduling**
  * **Problem**: standard schedulers destroy **intra-thread memory-level parallelism (MLP)** in multi-threaded workloads.
  * **How MLP destruction happens**:
    * A single thread may have multiple outstanding requests to **different banks** — these can be served in parallel (overlapping latencies).
    * When multiple threads run simultaneously, their requests interleave in the queue.
    * A naive scheduler services Thread A's bank-0 request, then Thread B's bank-0 request, then Thread A's bank-1 request...
    * Thread A's two requests are now serialized — even though they could have been issued back-to-back to different banks and overlapped.
  * **Two principles of ParBS**:
    * **Principle 1 — Intra-thread bank parallelism**: schedule a thread's requests to different banks consecutively → their latencies overlap.
    * **Principle 2 — Request batching**: prevent starvation caused by Principle 1.

* **ParBS: Batching Mechanism**
  * A **batch** is formed by marking the N oldest requests (up to the "marking cap") from each thread across all banks.
  * Marked requests (current batch) are always prioritized over unmarked requests (future batches).
  * When all marked requests are serviced → form a new batch from the next N oldest requests.
  * **Effect**: each thread is guaranteed to have its current batch completed before the next batch begins → no starvation, bounded waiting time.
  * **Marking cap trade-off**:
    * Too small → batches complete quickly but destroy row buffer locality (frequent policy switches).
    * Too large → memory-non-intensive threads wait too long for a batch to complete.

* **ParBS: Within-Batch Thread Ranking**
  * Within a batch, threads are **ranked** to guide which thread's requests to issue when multiple threads have requests to the same bank.
  * **Ranking goal**: minimize average stall time while maintaining fairness.
  * **Ranking algorithm** — for each thread, compute:
    * **Max bank load**: maximum number of marked requests to any single bank.
    * **Total load**: total number of marked requests across all banks.
  * **Ranking rule**:
    * Lower max bank load → ranked higher (thread benefits most from parallelism, has shortest stall time — shortest job first principle).
    * Ties broken by lower total load → ranked higher.
  * **Example** (4 threads):

    | Thread | Max Bank Load | Total Load | Rank |
    |---|---|---|---|
    | Thread 0 | 1 | 3 | **1st** (lowest max bank load) |
    | Thread 1 | 2 | 4 | **2nd** |
    | Thread 2 | 2 | 6 | **3rd** (same max bank load as T1, higher total) |
    | Thread 3 | 5 | 9 | **4th** |

  * Higher-ranked threads have their requests issued first → their requests spread across banks → latencies overlap → shorter stall time.

* **Complete ParBS Scheduling Priority Order**
  * Within the current batch:
    * **Priority 1**: Row hits (for throughput).
    * **Priority 2**: Higher-ranked thread (for parallelism and fairness).
    * **Priority 3**: Older requests (for age-based fairness within a thread).
  * Outside the current batch: only serve unmarked requests to banks that have no marked requests pending.

* **Comparison: STFM vs. ParBS**

  | Property | STFM | ParBS |
  |---|---|---|
  | Primary goal | Fairness (equalize slowdowns) | Performance (preserve bank parallelism) |
  | Mechanism | Slowdown estimation + priority switching | Batching + thread ranking |
  | Starvation prevention | Via throughput/fairness mode switching | Via bounded batch completion |
  | Handles bank parallelism | No | Yes — first scheduler to address this |
  | Implementation complexity | High (requires stall-time estimation) | Moderate (no complex arithmetic) |
  | Critical path overhead | High | Low (not on critical path) |
  | Main limitation | Inaccurate stall estimation; complex | May delay latency-sensitive threads within a batch |

* **Broader Interference Mitigation Approaches**
  * Beyond scheduling, interference can be reduced through:
    * **Source throttling / injection rate control**: limit the rate at which a core or GPU injects requests into the memory system → reduces pressure without changing the scheduler.
    * **Channel / bank partitioning**: assign different applications to different memory banks or channels → eliminates structural interference at the cost of reduced flexibility.
    * **OS-level thread co-scheduling**: avoid scheduling high-interference thread pairs on the same machine simultaneously.
    * **Decoupled DMA**: isolate CPU and I/O memory traffic to prevent I/O from starving CPU requests.
  * These approaches can be combined — making the memory controller "smart" (scheduling) while also using OS-level policies (thread co-scheduling) simultaneously.
