<div align="center">

```
╔═══════════════════════════════════════════════════════════════════╗
║                                                                   ║
║   ██████╗ ██████╗ ███╗   ███╗██████╗     █████╗ ██████╗  ██████╗ ║
║  ██╔════╝██╔═══██╗████╗ ████║██╔══██╗   ██╔══██╗██╔══██╗██╔════╝ ║
║  ██║     ██║   ██║██╔████╔██║██████╔╝   ███████║██████╔╝██║      ║
║  ██║     ██║   ██║██║╚██╔╝██║██╔═══╝    ██╔══██║██╔══██╗██║      ║
║  ╚██████╗╚██████╔╝██║ ╚═╝ ██║██║        ██║  ██║██║  ██║╚██████╗ ║
║   ╚═════╝ ╚═════╝ ╚═╝     ╚═╝╚═╝        ╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

# 🏛️ 计算机体系结构 · 个人学习笔记

### ETH Zürich · Fall 2025 · Prof. Onur Mutlu


[![课程主页](https://img.shields.io/badge/🌐_课程主页-ETH_Zürich-blue?style=for-the-badge)](https://safari.ethz.ch/architecture/fall2025/doku.php?id=start)
[![YouTube](https://img.shields.io/badge/▶_讲座视频-YouTube-red?style=for-the-badge)](https://www.youtube.com/c/OnurMutluLectures)
[![进度](https://img.shields.io/badge/📖_笔记进度-进行中-orange?style=for-the-badge)](#)


---

> *"The most important problems are the ones at the boundary of what's possible."*
>
> — Prof. Onur Mutlu，SAFARI Research Group，ETH Zürich

---

</div>


## 📖 关于本仓库

本仓库收录了本人在学习 **ETH Zürich 计算机体系结构（Fall 2025）** 课程过程中整理的个人笔记。这是一门世界顶级的计算机体系结构课程，由 Prof. Onur Mutlu 主讲，涵盖了从内存系统到处理器架构的前沿研究内容。

> 🔖 **课程简介**：本课程聚焦计算机体系结构中最前沿的方向，包括内存系统、安全性、可靠性、新兴存储技术、存内计算（Processing-in-Memory）、异构架构、互联网络和多处理器等核心议题。


## 🗓️ 课程章节 & 笔记索引


### 🔷 第一周 · 导论与基础

| 日期 | 讲座 |
|------|------|
| 09/25 Thu | **L1: Introduction and Basics** — 课程导论与体系结构基础 |
| 09/26 Fri | **L2: Memory Systems: Challenges and Opportunities** — 内存系统挑战与机遇 |
| 09/26 Fri | **L2a: Course Logistics** — 课程说明 |


### 🔷 第二周 · 以存储为中心的计算（一）

| 日期 | 讲座 |
|------|------|
| 10/02 Thu | **L3: Memory Systems: Challenges and Opportunities** — 内存系统挑战与机遇（续） |
| 10/03 Fri | **L4: Memory-Centric Computing** — 以存储为中心的计算 |


### 🔷 第三周 · 以存储为中心的计算（二）

| 日期 | 讲座 |
|------|------|
| 10/09 Thu | **L5: Memory-Centric Computing II** — 存内计算（二） |
| 10/10 Fri | **L6: Memory-Centric Computing III** — 存内计算（三） |


### 🔷 第四周 · 内存数据保持与延迟

| 日期 | 讲座 |
|------|------|
| 10/16 Thu | **L7: Data Retention in Memory** — 内存数据保持机制 |
| 10/17 Fri | **L8: Memory Latency** — 内存访问延迟 |


### 🔷 第五周 · 内存延迟与控制器（一）

| 日期 | 讲座 |
|------|------|
| 10/23 Thu | **L9a: Memory Latency II** — 内存延迟（续） |
| 10/23 Thu | **L9b: Memory Controllers** — 内存控制器 |
| 10/24 Fri | **L10a: Memory Controllers II** — 内存控制器（续） |
| 10/24 Fri | **L10b: Memory Controllers: Service Quality and Performance** — 服务质量与性能 |


### 🔷 第六周 · 内存控制器（二）与鲁棒性（一）

| 日期 | 讲座 |
|------|------|
| 10/30 Thu | **L11: Memory Controllers: Service Quality and Performance II** — 服务质量与性能（续） |
| 10/31 Fri | **L12: Memory Robustness I** — 内存鲁棒性（一） |


### 🔷 第七周 · 内存鲁棒性（二）—— RowHammer 深度剖析

| 日期 | 讲座 |
|------|------|
| 11/06 Thu | **L13: Memory Robustness II** — 内存鲁棒性（二） |
| 11/07 Fri | **L14a: Memory Robustness III** — 内存鲁棒性（三） |
| 11/07 Fri | **L14b: ColumnDisturb** — 列扰动攻击与防御 |
| 11/07 Fri | **L14c: PuDHammer** — 存内处理中的读取扰动实验分析 |
| 11/07 Fri | **L14d: RowHammer 引发的侧信道漏洞** — 分析与缓解 |
| 11/07 Fri | **L14e: 存内处理中的隐蔽与侧信道攻击** — 再审视 |
| 11/07 Fri | **L14f: CoMeT** — 基于 Count-Min-Sketch 的行追踪 RowHammer 防御 |
| 11/07 Fri | **L14g: ABACuS** — 全银行激活计数器的低开销 RowHammer 防御 |


### 🔷 第八周 · 新兴存储技术与闪存

| 日期 | 讲座 |
|------|------|
| 11/13 Thu | **L15: Emerging Memory Technologies** — 新兴存储技术 |
| 11/14 Fri | **L16: Flash Memory and Solid-State Drives** — 闪存与固态硬盘 |


### 🔷 第九周 · 闪存、仿真与预取

| 日期 | 讲座 |
|------|------|
| 11/20 Thu | **L17: Flash Memory and Solid-State Drives II** — 闪存与固态硬盘（续） |
| 11/21 Fri | **L18a: Simulation** — 架构仿真方法 |
| 11/21 Fri | **L18a (extended): A Brief Introduction to Ramulator** — Ramulator 仿真器介绍 |
| 11/21 Fri | **L18b: Prefetching & Prefetcher Design I** — 预取技术与预取器设计（一） |


### 🔷 第十周 · 预取器设计与多处理器

| 日期 | 讲座 |
|------|------|
| 11/27 Thu | **L19: Prefetching and Prefetcher Design II** — 预取器设计（二） |
| 11/28 Fri | **L20a: Prefetching and Prefetcher Design III** — 预取器设计（三） |
| 11/28 Fri | **L20b: Multiprocessors** — 多处理器体系结构 |


### 🔷 第十一周 · 多处理器、内存序与虚拟内存

| 日期 | 讲座 |
|------|------|
| 12/04 Thu | **L21a: Multiprocessors II** — 多处理器（续） |
| 12/04 Thu | **L21b: Memory Ordering** — 内存访问顺序模型 |
| 12/04 Thu | **L21c: Cache Coherence** — 缓存一致性协议 |
| 12/05 Fri | **L22: Virtual Memory** — 虚拟内存系统 |


### 🔷 第十二周 · 互联网络与片上网络

| 日期 | 讲座 |
|------|------|
| 12/11 Thu | **L23: Interconnects** — 互联网络 |
| 12/12 Fri | **L24: On-Chip Networks & Router Design** — 片上网络与路由器设计 |


### 🔷 第十三 & 十四周 · 专题讲座

| 日期 | 讲座 |
|------|------|
| 12/19 Fri | **L25: Accelerating Biological Sequence Analysis** — 生物序列分析加速架构 |
| 01/05 Mon | **L26a/b: Parallelism, Heterogeneity and Bottleneck Acceleration** — 并行性、异构与瓶颈加速 |
| 01/06 Tue | **L27: VLIW Architectures** — 超长指令字架构 |
| 01/08 Thu | **L28: Systolic Array Architectures** — 脉动阵列架构 |
| 01/09 Fri | **L29a: SIMD Architectures** — 单指令多数据架构 |
| 01/09 Fri | **L29b: GPU Architectures** — GPU 体系结构 |


### 🔷 第十五周 · GPU 编程与推测执行

| 日期 | 讲座 |
|------|------|
| 01/12 Mon | **L30: GPU Programming** — GPU 编程模型 |
| 01/13 Tue | **L31: Speculation** — 推测执行技术 |


---

## 🗺️ 知识图谱

```
计算机体系结构 (Fall 2025)
│
├── 🧠 存储系统 (Weeks 1–9)
│   ├── 内存挑战与机遇
│   ├── 以存储为中心的计算 (PIM/PUD)
│   ├── 数据保持 & 访问延迟
│   ├── 内存控制器 & QoS
│   ├── 内存鲁棒性 & RowHammer
│   ├── 新兴存储技术
│   └── 闪存 & SSD
│
├── ⚙️ 微架构优化 (Weeks 9–10)
│   ├── 体系结构仿真 (Ramulator)
│   └── 预取技术 & 预取器设计
│
├── 🔗 并行与多核 (Weeks 10–11)
│   ├── 多处理器体系结构
│   ├── 内存访问顺序模型
│   ├── 缓存一致性协议
│   └── 虚拟内存
│
├── 🌐 互联与网络 (Week 12)
│   ├── 片外互联网络
│   └── 片上网络 & 路由器设计
│
└── 🚀 加速架构 (Weeks 13–15)
    ├── 生物序列分析加速
    ├── 并行性 & 异构计算
    ├── VLIW / 脉动阵列 / SIMD
    ├── GPU 架构 & 编程
    └── 推测执行
```


---

## 📚 推荐阅读

本课程强调通过阅读研究论文来深入理解前沿成果。以下是部分核心参考：

- 📄 [You and Your Research — Richard Hamming](https://safari.ethz.ch/architecture/fall2025/lib/exe/fetch.php?media=youandyourresearch.pdf)
- 🎥 [Intelligent Architectures for Intelligent Systems — Prof. Mutlu](https://www.youtube.com/watch?v=WxHribseelw)
- 📺 [历年完整讲座录像 — YouTube](https://www.youtube.com/c/OnurMutluLectures)


---

## ⚠️ 免责声明

本仓库仅为**个人学习笔记**，所有课程内容、幻灯片及论文版权均归 ETH Zürich 及相关作者所有。笔记中若有错误，欢迎提 Issue 或 PR 指正。


---

<div align="center">

制作 with ❤️ · 致敬 SAFARI Research Group

*"好的体系结构研究，始于对问题本质的深刻追问。"*

</div>
