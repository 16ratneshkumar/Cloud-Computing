[← Back to Index](../index.md)

# DEPLOYMENT 22 — Embedded Hardware → INTEGRITY Multivisor

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              INTEGRITY Multivisor              │
├──────────────────────────────────────────────────┤
│               Embedded Hardware                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│           CERTIFIED SAFETY PARTITIONS                           │
│  ┌───────────────────────────┐  ┌──────────────────────────┐   │
│  │  INTEGRITY RTOS Partition │  │  Linux/Android Partition │   │
│  │  (DO-178C Level A)        │  │  (general purpose)       │   │
│  │  Flight control / avionics│  │  Mission systems UI      │   │
│  └───────────────────────────┘  └──────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              INTEGRITY Multivisor (Green Hills Software)         │
│  Formally certified to DO-178C Level A / IEC 62443              │
│  Proven zero kernel vulnerabilities in 30+ years                │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│    Safety-Critical Embedded Hardware (Aerospace / Defense)      │
│  CPU (PowerPC / ARM / x86) │ RAM │ Custom I/O                   │
│                  BARE METAL                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Commercial aircraft (Boeing 787 fly-by-wire), military systems, and medical devices requiring the highest safety certifications.

## Pros / Cons — See previous section (Deployment 22 detailed earlier)