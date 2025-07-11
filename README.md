# Towards Full-System Heterogeneous Simulation in gem5

As SoC designs evolve, they are rapidly embracing heterogeneity—not just CPUs and GPUs, but tightly integrated programmable accelerators optimized for specific workloads. These accelerators are now central to domains like mobile inference, AR/VR, real-time computer vision, and edge analytics. Unlike traditional CPU-GPU systems, these emerging platforms require precise coordination among diverse compute engines, shared memory systems, and software-managed control flows. Accurately modeling such interactions demands a simulator that is both cycle-accurate and full-system capable.

gem5 has long supported detailed CPU simulation and, more recently, added GPU support. But for programmable accelerators, researchers relied on external tools like gem5-SALAM, built on gem5 v21.1. While SALAM offers key accelerator modeling features, it remains isolated from mainline gem5 -- missing support for new ISAs, memory models, configuration infrastructure, and continuous validation.

To address this gap, we merged gem5-SALAM into gem5-develop v25. This integration makes accelerators first-class components in gem5, allowing users to simulate CPUs, GPUs, and custom accelerators under one unified software stack. The result is a full-system simulation framework capable of evaluating heterogeneous SoCs with realistic OS support, shared resources, and software-controlled coordination.

## Integration at a Glance

We integrated SALAM’s accelerator modeling infrastructure into gem5-develop through a series of architectural, interface, and validation updates.

We began by the key accelerator modeling components from SALAM into gem5. These include the `LLVMInterface`, which executes LLVM IR kernels using a cycle-accurate datapath; the `CommInterface`, which provides software-visible control and interrupt signaling; and a suite of configurable memory components such as scratchpads, DMA engines, and stream buffers. Together, these elements enable detailed and flexible modeling of a wide range of accelerator microarchitectures and memory hierarchies. 

To support realistic SoC integration, accelerators and local memories can be grouped into an `AccCluster`, reflecting the modular structure of accelerator subsystems commonly found in commercial SoCs. This abstraction simplifies system-level integration by treating compute and memory components as a unified module exposed through a single port. For rapid prototyping, we also integrated and automated SALAM’s hardware profile generator, which converts user-defined timing specifications into executable datapath models -- eliminating the need for manual microarchitectural implementation. Finally, we refactored CACTI-SALAM for compatibility with gem5’s infrastructure, enabling timing and energy estimation for scratchpad memories using CACTI’s file-based configuration methodology. These changes bring cycle-level accelerator modeling, full-system memory interaction, and scalable design space exploration into gem5 mainline.

Bringing SALAM into gem5-develop also meant modernizing its interface. We updated all components to use gem5’s new SimObject constructors, port APIs, and memory address semantics. Legacy C-style pointer handling in the LLVM runtime was replaced with safe, architecture-compliant constructs. We also added support for `llvm-config` to automatically detect system LLVM installations, simplifying the build process.

All imported components pass gem5’s standard pre-commit and regression tests. We also ported SALAM’s original system-validation suite and verified correctness by matching outputs against the baseline. 

## What This Enables

### Broader Heterogeneity Studies

With accelerators now fully integrated into gem5 mainline, researchers can simulate complete heterogeneous systems comprising CPUs, GPUs, and custom accelerators—co-existing under a single OS kernel and sharing interconnects and memory. This allows detailed studies of performance interference, resource arbitration, and synchronization mechanisms across diverse compute engines, grounded in full-system behavior rather than simplified models.

### System-Level Exploration

The framework supports rich exploration of architectural tradeoffs at the system level. Users can evaluate different memory organizations—such as private scratchpads, shared LLCs, or DMA-managed SPMs—and compare strategies for offloading, synchronization, and kernel placement. Static vs. dynamic scheduling, locality-aware memory partitioning, and software-managed DMA schemes can all be studied in realistic OS-driven settings.

### Domain-Specific Workload Support

This infrastructure also enables architectural research targeting emerging domains like real-time vision, mobile inference, AR/VR, and edge computing. These applications demand predictable latency, software-accelerator coordination, and careful memory management. The integrated framework allows researchers to study these workloads using real software stacks and bootable Linux images, with accelerator behavior modeled at cycle-level fidelity.

### Evaluating Accelerator Behavior Under Emerging Operating Regimes

Finally, the toolchain enables system-level studies of high-frequency accelerator operation under new regimes such as transient overclocking and advanced cooling. In our workshop paper, we use this framework to scale accelerator frequency up to multi-GHz speeds and present a preliminary study of how performance and power behavior evolve across this range. The results illustrate how system bottlenecks shift as frequency increases, underscoring the need to analyze accelerator behavior in context with host latency and memory interactions. Further details on the experimental setup and results are included in our ISCA ’25 workshop paper.

## Modeling Your Own Accelerator

Creating a new accelerator model in the integrated gem5 framework is simple. You begin by writing the desired accelerator algorithm in C or C++. This is then compiled to LLVM IR using `clang`. A YAML profile describes the pipeline structure, functional unit delays, and memory interfaces. The hardware-profile generator converts this into a timing model that integrates directly with gem5.

The user then places the accelerator inside an `AccCluster`, attaches scratchpads or DMAs as needed, and configures the system topology using gem5’s Python interface. The accelerator is exposed to software via memory-mapped control registers and an interrupt line, allowing realistic coordination from user-space drivers running inside the simulated OS.

This modeling flow enables fast iteration on architectural ideas, from exploring low-level datapath variations to studying how different memory interface strategies affect system performance.

## Getting Started

To begin using the integrated accelerator support:

```bash
git clone https://github.com/akanksha-sc/gem5
cd gem5
scons build/ARM/gem5.opt -j$(nproc) SALAM_ACCELS=1
```

Then run a validation benchmark, such as BFS from the system-validation suite:

```bash
$M5_PATH/tools/run_system.sh --bench bfs --bench-path benchmarks/sys_validation/bfs
```

This boots Linux, launches a user-space driver, and executes the accelerator. Simulation outputs include `stats.txt` (performance counters), `system.terminal` (console output), and `SALAM_power.csv` (energy/area estimates if CACTI is enabled). Additional examples and documentation are available under `src/hwacc/docs`.

## Conclusion

This integration brings gem5 one step closer to serving as a comprehensive platform for heterogeneous system simulation. By unifying CPUs, GPUs, and programmable accelerators within a single, full-system simulation environment, we enable researchers to study modern SoC behavior with realistic timing, software, and architectural detail.

From co-scheduling policies to memory-system tuning and sprint-aware accelerator design, this infrastructure supports a broad range of experimental directions. We encourage the community to build on this work—whether by modeling new accelerators, porting benchmarks, or exploring advanced cooling regimes—and contribute back through the standard gem5 development process.

## References
