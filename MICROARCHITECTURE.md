# POWER10 and POWER11 microarchitecture notes

Status: microarchitecture research. This file complements, but does not replace, the Power ISA 3.1C inventory in INSTRUCTIONS.md.

The organizing question here is not "how does BLAS implement QR?" It is "what machinery is physically available, how is it fed, what state can remain close to execution, and where does data movement become expensive?"

## Evidence ladder

Use these levels in order:

1. IBM processor presentations, Redbooks, and IBM Research papers.
2. Processor-specific measurements on an actual Power Virtual Server or physical machine.
3. Library/compiler behavior only as evidence of what software currently chooses to do.

Do not infer a POWER11 pipeline detail merely because POWER10 used it. Do not infer a circuit from an ISA instruction name.

## Rentable targets

IBM Power Virtual Server is the practical first execution target for this branch.

IBM Cloud currently documents PowerVS capacity on:

- POWER10 S1022, E1080, and E1050 systems;
- POWER11 S1122 and E1150 systems, depending on data center availability.

PowerVS billing includes shared capped, shared uncapped, and dedicated core-hour models. This makes POWER10 and POWER11 more useful for this project than a processor family that is documented but cannot be obtained for a short experiment.

When a rented VSI is used, record the exact machine type, SMT mode, entitlement/core mode, OS, visible NUMA/cache topology, and processor generation. A virtual processor does not by itself identify every physical contention boundary.

## POWER10 core: the useful machine picture

IBM's POWER10 Hot Chips material exposes a much more useful optimization model than the ISA alone.

### Front end and in-flight work

The published POWER10 core diagram shows:

- fetch plus branch prediction;
- predecode with fusion and prefixed-instruction handling;
- a 48 KiB, 6-way L1 instruction cache in the half-core view;
- fetch delivery of 8 instructions;
- a 128-entry instruction buffer;
- decode/fuse capacity of 8 internal operations;
- a 512-entry instruction table.

IBM describes POWER10 as an 8-way superscalar SMT8 core. The Hot Chips diagram explicitly shows one half of an SMT8 core, equivalent to an SMT4 resource domain, so repeated execution resources in that figure must not be mistaken for the full-core count.

The important consequence is a deep window with substantial opportunity to overlap independent work. A transform represented as a long serial dependency chain can leave much of this machinery idle even when its instruction count is small.

### Execution resources

The full SMT8 POWER10 core has two execution-resource domains. IBM Redbooks describe:

- eight 128-bit Vector Scalar Unit execution slices per core;
- four Matrix Math Accelerator units per core;
- two quad-precision / decimal floating-point units.

The half-core Hot Chips diagram shows four 128-bit execution slices and two 512-bit MMA blocks, matching that two-domain organization.

The eight VSU slices handle scalar and 128-bit SIMD work. MMA is not merely "wider VSX": it adds accumulator state and outer-product-style operations intended to keep partial matrix results local rather than repeatedly round-tripping them through ordinary vector registers and memory.

### MMA

IBM's POWER10 MMA guidance gives the practical headline:

- 8 independent fixed/floating SIMD engines per core;
- 4 MMA engines per core;
- each MMA engine produces a 512-bit result per cycle;
- supported arithmetic includes single and double precision plus BF16, FP16, int16, int8, and int4 matrix operations.

For our purposes, the most interesting property is **state locality**. MMA accumulator state exists so repeated multiply/accumulate work can stay near the matrix engine. That means a future "many small rotations" lowering should ask whether the transformation can be organized around resident accumulated state, not whether it can be made to resemble a particular BLAS entry point.

The current ISA does not supply our E3M2/E4M3/E5M2/E5M3 formats directly. Storage conversion and accumulation policy therefore remain our problem.

### Load/store and translation machinery

The published POWER10 core diagram includes:

- two load effective-address paths;
- two store effective-address paths;
- two 32-byte load paths in the shown domain;
- a 32-byte store path, with gathered stores;
- 128-entry load queue in SMT mode / 64 entries in single-thread mode;
- 80-entry store queue in SMT mode / 40 entries in single-thread mode;
- 12-entry load-miss queue;
- 16 prefetch streams;
- 64-entry ERAT;
- 4K-entry TLB in the shown organization;
- 48 L3-prefetch entries.

IBM's system documentation reports a 2 MiB 8-way L2 per core and substantially enlarged translation resources relative to POWER9.

For a high-dimensional transform, these are not secondary details. A representation that saves arithmetic while creating irregular gathers, spills accumulator state, or defeats prefetch may lose.

### Cache latency and bandwidth landmarks

IBM reports nominal POWER10 data-access latencies of approximately:

- L1 data: 4 cycles, with no additional store-forwarding penalty;
- L2: 13.5 cycles;
- L3: 27.5 cycles;
- effective-to-real translation after an ERAT miss: about 8.5 cycles.

The Hot Chips material also advertises doubled load/store bandwidth relative to POWER9.

These are processor-level landmarks, not promised timings for an oversubscribed PowerVS guest. Measure the rented system again before relying on them in scheduling.

### SMT matters

POWER10 supports SMT1/2/4/8 modes. Resource sharing changes with SMT mode. A result obtained in SMT8 is not automatically a single-thread latency/throughput result, and a single-thread kernel should not be designed from aggregate throughput numbers without recording the thread mode.

## POWER11: documented continuation, not a guessed POWER10

POWER11 remains closely related to POWER10 at the ISA and matrix-acceleration level, but keep its implementation facts separate.

Current IBM Redbooks document:

- up to 16 SMT8 cores on a processor chip;
- 2 MiB L2 per core;
- up to 128 MiB on-chip L3;
- four MMA units per core;
- two times general SIMD and four times matrix SIMD per core relative to POWER9;
- two times OMI memory bandwidth relative to POWER10 on the documented systems;
- 2 TB/s raw PowerAXON + OMI signaling in the high-end description;
- 96 KiB aggregate L1 instruction cache and 64 KiB aggregate L1 data cache per core in the current Power11 processor description.

IBM also says POWER11 improves effective MMA throughput through improved feeding and memory bandwidth.

What the public material does **not** justify is copying every POWER10 queue size, issue rule, or instruction latency into a POWER11 scheduler. Until an IBM source or measurement establishes that detail, mark it unknown.

## What to measure on the first rented POWER machine

Before adding a POWER-specific high-dimensional transform lowering, collect:

- exact processor generation and system model;
- SMT mode and virtual processor entitlement;
- cache sizes/topology visible to the partition;
- VSX scalar/vector latency and throughput for the operations we actually use;
- MMA setup, accumulator transfer, outer-product, and accumulator-discharge costs;
- sustained load/store rates for resident L1/L2/L3 and memory-sized working sets;
- stride and gather sensitivity;
- conversion cost for candidate compact formats;
- cost of keeping transform state in VSRs versus MMA accumulators;
- branch/fusion behavior for the proposed control structure;
- scaling from one hardware thread through the SMT modes available to the partition.

Benchmark intent-level kernels such as "apply N independent small rotations" and "compose many nearby rotations" in addition to conventional GEMM. The conventional library kernels are useful controls, not our API.

## Sources

Primary IBM sources:

- IBM POWER10 Hot Chips 32 processor presentation:
  https://hc32.hotchips.org/assets/program/conference/day1/HotChips2020_Server_Processors_IBM_Starke_POWER10_v33.pdf
- IBM, "Introduction to the MMA (Matrix Math Accelerator) component of Power10 systems":
  https://www.ibm.com/support/pages/introduction-mma-matrix-math-accelerator-component-power10-systems
- IBM Redbooks, Power E1050 technical overview, including POWER10 execution resources and cache/TLB latency:
  https://www.redbooks.ibm.com/redpapers/pdfs/redp5684.pdf
- IBM Redbooks, Power11 E1150 Introduction:
  https://www.redbooks.ibm.com/redbooks/pdfs/sg248589.pdf
- IBM Redbooks, Power11 Scale-Out Servers:
  https://www.redbooks.ibm.com/redbooks/pdfs/sg248590.pdf
- IBM Cloud, Power Virtual Server architecture:
  https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-on-cloud-architecture
- IBM Cloud, Power Virtual Server pricing:
  https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-pricing-ibm-data-center

## Boundary

This file records machine facts and questions. It does not claim that Idriç emits VSX or MMA, and it deliberately does not organize the machine around QR, Householder, BLAS, or another inherited software API.
