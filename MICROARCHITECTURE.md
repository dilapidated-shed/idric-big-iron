# Rentable AArch64 microarchitecture: Graviton5/V3 and Graviton4/V2

Status: processor/microarchitecture research. The complete A64/NEON/SVE/SVE2/SME ISA catalog remains in INSTRUCTIONS.md; this file describes concrete cores that can actually be rented and measured.

## Priority order

As of 2026-09-24, the most useful cloud targets are:

1. **AWS Graviton5 / Neoverse V3** — current C9g, M9g, and R9g families are generally available in multiple regions and can be purchased On-Demand or Spot.
2. **AWS Graviton4 / Neoverse V2** — widely available C8g/M8g/R8g-class systems with excellent public microarchitecture documentation.

This branch should not optimize for an abstract "Arm server." Use the concrete CPU profile.

## AWS Graviton5

AWS's current Graviton technical guide identifies Graviton5 as:

- Arm Neoverse V3 cores;
- Armv9.2-A class;
- 3.3 GHz nominal core frequency, no turbo;
- four 128-bit Neon/SVE vector execution resources in the AWS profile;
- 192 cores in the documented processor;
- 64 KiB L1 instruction + 64 KiB L1 data per core;
- 2 MiB private L2 per core;
- shared LLC whose exact size/topology depends on instance size/NUMA configuration;
- DDR5 memory;
- CMN-S3 interconnect;
- one hardware thread per vCPU/core in EC2's Graviton instance presentation.

AWS recommends tuning specifically with a Neoverse V3 CPU target when the compiler supports it.

### Neoverse V3 execution model

Arm's V3 Software Optimization Guide describes an in-order fetch/decode/rename/dispatch front end feeding an out-of-order backend. Instructions decode into internal macro-operations; a macro-operation may later split into two micro-operations.

The published guide describes a very wide backend with more than twenty issue pipelines covering:

- branch work;
- six simple integer pipes plus multicycle integer resources;
- four FP/Advanced-SIMD/SVE resources;
- multiple load/address-generation paths;
- store address and store-data resources.

Arm's current top-down performance methodology uses **ten rename slots per cycle** for Neoverse V3. That number is more useful for frontend pressure than treating A64 instruction count as equivalent to backend work.

Do not hard-code an LLVM scheduling-model number unless it is also supported by the Arm guide: compiler models sometimes carry placeholders for details not publicly specified.

### V3 caches and translation

Arm's V3 TRM documents:

- 64 KiB, 4-way L1 instruction cache;
- 64 KiB, 4-way L1 data cache;
- 64-byte cache lines;
- private unified L2 configurable as 2 MiB or 3 MiB in the generic IP, with four banks;
- a 256-bit CHI connection from the private L2 toward the system;
- implementation-specific system-level cache outside the core.

The AWS Graviton5 profile fixes the private L2 at 2 MiB. Do not replace AWS's silicon configuration with another legal V3 configuration from the generic Arm TRM.

Arm's current V3 optimization material also documents alignment cases that can reduce load/store bandwidth, including cache-line-crossing loads and stores crossing certain internal boundaries. These are exactly the details that matter for compact packed representations.

## AWS Graviton4

AWS identifies Graviton4 as:

- Neoverse V2;
- Armv9.0-A;
- 2.8 GHz nominal frequency, 2.7 GHz for the largest 48xlarge profile;
- no turbo;
- four 128-bit Neon/SVE execution resources in the AWS profile;
- SVE2 plus SVE integer, BF16, bit-permutation, and crypto facilities;
- 96 cores per socket, with a two-socket/192-core largest profile;
- 64 KiB L1 instruction + 64 KiB L1 data per core;
- 2 MiB private L2 per core;
- 36 MiB shared LLC in the documented profile;
- 12 DDR5 channels per socket, doubled on the largest two-socket profile;
- CMN-700 interconnect.

The important point for vector work is that **SVE is 128-bit on this concrete V2 implementation**. "SVE" is not permission to assume an arbitrarily wide physical vector datapath.

### Neoverse V2 execution model

Arm's V2 Software Optimization Guide is unusually useful for backend work. It describes:

- macro-operations after decode;
- register rename and dispatch;
- possible split into two micro-operations;
- out-of-order issue;
- 17 issue pipelines, each accepting one micro-operation per cycle:
  - 2 branch;
  - 4 simple integer;
  - 2 integer simple/multicycle;
  - 4 FP/Advanced-SIMD;
  - 2 load/store address;
  - 1 additional load;
  - 2 store-data.

The same guide publishes instruction-family latency, throughput, and pipeline use. For example, it distinguishes basic FP arithmetic, multiply, multiply-accumulate, reductions, vector loads, and SVE operations rather than giving a single "vector throughput" number.

That should be the first scheduling reference before we write any hand-shaped V2 sequence.

### V2 front-end changes

Arm reports substantial V2 front-end growth relative to V1, including much larger branch-target structures, larger TAGE prediction structures, doubled instruction-TLB/cache bandwidth, and a doubled fetch queue.

The practical lesson is not "branches are free." It is that branch shape, code footprint, and instruction supply need to be measured separately from vector arithmetic.

## What this means for compact rotations and low precision

For both V2 and V3:

- 128-bit physical vector width makes packing choices concrete;
- BF16/FP16/dot-product support may be useful as widening or accumulation waypoints even when E3M2/E4M3/E5M2/E5M3 are stored more compactly;
- conversion and unpack/shuffle cost must be measured, not treated as bookkeeping;
- small-rotation work should be tested as resident-state kernels, not only through matrix-library benchmarks;
- SVE predication may remove scalar cleanup/control overhead even when vector width is only 128 bits;
- the private 2 MiB L2/core on both Graviton4 and Graviton5 is a meaningful working-set boundary;
- system-level cache and NUMA topology differ by EC2 size, so the rented instance type belongs in every result.

## First rental measurements

For a first C9g/M9g or C8g/M8g instance, record:

- exact instance type and region;
- /proc/cpuinfo, lscpu, getauxval/HWCAP feature mask;
- SVE vector length returned by the machine;
- cache and NUMA topology;
- PMU access available to the guest;
- latency/throughput of the exact FP, integer, conversion, permute, predicate, and reduction operations considered for a lowering;
- sustained L1/L2/LLC/DRAM bandwidth;
- aligned and boundary-crossing packed loads/stores;
- scaling with vector width and unroll factor;
- branch/predicate alternatives;
- packed low-precision decode -> compute -> encode cost.

## Sources

AWS:

- AWS Graviton Technical Guide:
  https://aws.github.io/graviton/
- AWS EC2 current general-purpose instance specifications:
  https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html
- AWS EC2 current compute-optimized instance specifications:
  https://docs.aws.amazon.com/ec2/latest/instancetypes/co.html
- M9g/M9gd general availability:
  https://aws.amazon.com/about-aws/whats-new/2026/06/ec2-m9g-m9gd-instances-graviton5-processors-available/
- C9g/C9gd general availability:
  https://aws.amazon.com/about-aws/whats-new/2026/06/ec2-c9g-c9gd-instances-graviton5-processors-available/
- R9g/R9gd general availability:
  https://aws.amazon.com/about-aws/whats-new/2026/08/amazon-ec2-r9g-and-r9gd-memory-optimized-instances-are-now-available/

Arm:

- Neoverse V3 Software Optimization Guide:
  https://documentation-service.arm.com/static/6734eb2627eda361ad4da4f4
- Neoverse V3 Technical Reference Manual:
  https://documentation-service.arm.com/static/67361bd2c7fc0d1f211da382
- Neoverse V3 telemetry/top-down documentation:
  https://documentation-service.arm.com/static/66f71ac61669c0388dca6d9b
- Neoverse V2 Software Optimization Guide:
  https://documentation-service.arm.com/static/668bc0a369e89f01e39c4668
- Arm's Neoverse V2 microarchitecture overview:
  https://developer.arm.com/community/arm-community-blogs/b/servers-and-cloud-computing-blog/posts/arm-neoverse-v2-platform-best-in-class-cloud-and-ai-ml-performance

## Boundary

These notes describe concrete rentable microarchitectures. They do not imply that the generic A64 backend emits SVE/SVE2 today, and they do not collapse V2, V3, or any future Neoverse core into one scheduling model.
