[← Back to Index](../index.md)

# DEPLOYMENT 82 — Bare Metal → seL4 → Formally Verified Partitions

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│          Formally Verified Partitions          │
├──────────────────────────────────────────────────┤
│                      seL4                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTITIONED WORKLOADS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Partition A (Safety Critical)    Partition B (General)  │   │
│  │  ┌──────────────────────────┐     ┌──────────────────┐   │   │
│  │  │  Certified RTOS / app    │     │  Linux / apps    │   │   │
│  │  │  Medical device control  │     │  (UI, comms)     │   │   │
│  │  │  Flight control system   │     │                  │   │   │
│  │  └──────────────────────────┘     └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              seL4 MICROKERNEL HYPERVISOR                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  THE FORMALLY VERIFIED MICROKERNEL                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Kernel = ~10,000 lines of C (tiny — proven safe)  │  │   │
│  │  │                                                     │  │   │
│  │  │  Formal proofs (Isabelle/HOL theorem prover):       │  │   │
│  │  │  ✓ Functional correctness (does what spec says)    │  │   │
│  │  │  ✓ No buffer overflows                             │  │   │
│  │  │  ✓ No NULL pointer dereferences                   │  │   │
│  │  │  ✓ No arithmetic overflows                         │  │   │
│  │  │  ✓ Stack bounded                                   │  │   │
│  │  │                                                     │  │   │
│  │  │  Capability-based access control                   │  │   │
│  │  │  (objects accessed only via unforgeable caps)       │  │   │
│  │  │                                                     │  │   │
│  │  │  CAmkES / Microkit (component framework on seL4)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  VM Extensions (ARM SMMUv3 / x86 IOMMU for device iso)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HARDWARE (ARM / RISC-V / x86)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ARM Cortex-A / RISC-V SoC (preferred — smaller TCB)    │   │
│  │  IOMMU (required for device isolation guarantees)        │   │
│  │  RAM │ Flash │ Peripherals                               │   │
│  │              BARE METAL                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

FORMAL VERIFICATION PIPELINE:
┌─────────────────────────────────────────────────────────────────┐
│  seL4 C source → binary translation proof → Isabelle/HOL proofs │
│  → machine-checked proofs (billions of proof steps verified)    │
│  TSYS (verified) → Security Properties: confidentiality,       │
│  integrity, and availability proofs per partition               │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- seL4 is the world's only formally verified OS kernel — every line proven correct in Isabelle/HOL
- Capability-based access control prevents any partition from accessing another's memory without an explicit capability
- IOMMU extends isolation to DMA-capable devices — even peripherals can't violate partition boundaries
- CAmkES/Microkit provides component frameworks for building systems on seL4
- The formal proofs verify: no buffer overflow, no undefined behavior, stack bounded — in the kernel

## Use Case
Military systems, medical devices (pacemakers, insulin pumps), aviation, and any domain where a security/safety compromise is catastrophically unacceptable.

## Pros
- Only formally verified OS kernel in production
- Mathematical proof of security properties
- Minimal attack surface (~10K lines)
- Used in DARPA HACMS, Boeing aircraft, autonomous vehicle systems

## Cons
- Extremely specialized development (CAmkES, proof-aware)
- Very small developer pool
- Performance limited by microkernel IPC overhead
- Most commercial software cannot run directly