# IBM z17 / Telum II microarchitecture notes

Status: processor-level research for the z/Architecture branch. This file is intentionally separate from the complete ISA inventory in INSTRUCTIONS.md.

This target is lower priority for immediate experimentation than commodity EC2 AArch64/x86 or IBM Power Virtual Server, because this pass did not establish an equally simple short-term rental path. It remains valuable as a big-iron reference because IBM publishes substantial processor and cache-topology detail.

## Evidence boundary

Prefer:

1. IBM Research processor papers;
2. current IBM Redbooks for the z17 system;
3. measurements from an actual Linux on Z / LinuxONE partition if access becomes available.

Do not infer Telum II execution behavior from the z/Architecture instruction set. Do not convert system-level virtual-cache terminology into a conventional private/shared-cache model without following IBM's description.

## Telum II chip

IBM Research documents Telum II as:

- Samsung 5 nm bulk technology;
- approximately 600 mm²;
- 43 billion transistors;
- eight high-performance processor cores;
- 5.5 GHz operation for the z17 high-end processor;
- an integrated AI accelerator;
- an on-chip data processing unit (DPU) for I/O acceleration;
- two PCIe Gen5 ×16 interfaces per processor chip;
- M-BUS to the second processor chip in the dual-chip module;
- X-BUS links to processor chips in the other DCMs of a drawer;
- A-BUS connectivity across drawers.

A full high-end topology can use four dual-chip modules per drawer, two processor chips per DCM, and up to four drawers, yielding up to 32 coherently connected processor chips.

## Core execution

The current IBM z17 Technical Introduction describes each processor unit as superscalar and out-of-order:

- up to six instructions decoded per clock;
- up to ten instructions run per clock;
- out-of-order instruction issue;
- out-of-order operand fetching;
- high-frequency, low-latency pipelines whose depth varies by instruction class.

Telum II also adds or improves:

- branch prediction;
- instruction-cache prefetching;
- the number of rename registers;
- TLB behavior.

That is enough to reject a naive "CISC instruction = one expensive serial operation" model. Instruction count, decoded width, operation expansion, dependencies, and memory behavior need to be measured separately.

## L1 and L2 structure

Per core, IBM documents:

- 128 KiB L1 instruction cache;
- 128 KiB L1 data cache.

Each processor chip contains ten 36 MiB L2-cache instances.

IBM Research's Telum II description is more specific than the shorthand "36 MiB L2 per core":

- each of the eight CPU cores has a private 36 MiB L2;
- the DPU has a private 36 MiB L2;
- one additional floating 36 MiB L2 participates in the hierarchy;
- the ten L2 instances are fully connected by a 352 GB/s on-chip ring.

This hierarchy is deliberately unusual. Do not flatten it into an x86-like per-core-L2-plus-shared-L3 picture when reasoning about data placement.

## Virtual L3 and L4

IBM describes the on-chip L2 system as providing a **360 MiB virtual L3** on each processor chip.

At full drawer scale, the coherent hierarchy provides about **2.88 GiB of virtual L4**.

These are virtualized cache levels assembled from lower-level cache resources and cross-chip topology, not additional monolithic SRAM blocks called L3 and L4. Any latency/bandwidth model must preserve that distinction.

## DCM and drawer topology

A z17 dual-chip module contains two processor chips. A drawer contains four DCMs.

Interconnect roles include:

- M-BUS between the two chips of a DCM;
- X-BUS among processor chips in the other DCMs in a drawer;
- A-BUS across drawers.

A result measured within one core, one chip, one DCM, one drawer, or across drawers is therefore a different locality case. High-dimensional work that shares state among threads must record placement rather than reporting only a thread count.

## DPU and accelerators

Telum II integrates a DPU into the processor chip for I/O acceleration and retains/enhances IBM's on-chip AI accelerator.

Those blocks should be documented as real machine resources, but they are not presumed useful for general Idriç transforms until there is a programming interface and execution path relevant to the intended operation.

Do not route an ordinary rotation/matrix operation through an accelerator merely because the block exists.

## What matters for future transform work

Before designing a z17-specific lowering, obtain measurements for the exact Linux partition and operation:

- scalar integer and floating dependency-chain latency;
- vector instruction latency/throughput for the relevant element widths;
- packed conversion and permutation cost;
- branch and condition-code behavior in realistic loops;
- L1-resident, local-L2, remote-cache, and memory-sized working sets;
- single-core versus multi-core scaling;
- placement within chip/DCM/drawer topology;
- translation/TLB sensitivity;
- interference from SMT or partition sharing if applicable;
- sustained rather than burst throughput.

The benchmark should include intent-level kernels such as "apply many independent small rotations" rather than only library APIs such as GEMM or QR.

## Access priority

For the current project:

1. run experiments first on already-available x86;
2. rent Graviton5/4 or POWER10/11 when their machine properties are relevant;
3. use z17/Telum II as a documented big-iron comparison until a practical Linux on Z/LinuxONE execution path is established.

This is a logistical priority, not a claim that z17 is less interesting architecturally.

## Sources

IBM Research:

- Telum II microprocessor, IEEE JSSC:
  https://research.ibm.com/publications/ibm-telum-ii-microprocessor-55-ghz-with-on-die-ai-and-data-processing-and-design-technology-co-optimizations-for-power-area-and-reliability
- Telum II ISSCC 2025 processor paper:
  https://research.ibm.com/publications/ibm-telum-ii-next-generation-55ghz-microprocessor-with-on-die-data-processing-unit-and-improved-ai-accelerator
- Enterprise-class on-chip accelerator integration, HPCA 2026:
  https://research.ibm.com/publications/enterprise-class-on-chip-accelerator-integration

IBM Redbooks:

- IBM z17 Technical Introduction:
  https://www.redbooks.ibm.com/redbooks/pdfs/sg248580.pdf
- IBM z17 (9175) Technical Guide:
  https://www.redbooks.ibm.com/redbooks/pdfs/sg248579.pdf
- z17 Redbooks landing page/current revisions:
  https://redbooks.ibm.com/feature/z17

IBM product material:

- Telum on IBM Z:
  https://www.ibm.com/products/z/telum
- IBM z17:
  https://www.ibm.com/products/z17

## Boundary

This note describes the processor and system machinery. It does not claim an executable Idriç s390x backend, and it does not treat vector-library conventions or established decomposition APIs as the hardware abstraction.
