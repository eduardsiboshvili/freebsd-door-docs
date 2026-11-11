[Monograph index](README.md) · [Full text](door-ipc-implementation-for-freebsd_full.md)

<a id="glossary"></a>

# Glossary and terminology

The following terms distinguish endpoint identity, discovery, transport, and
application ownership throughout the monograph.

| Term | Definition |
|---|---|
| Door | Kernel-managed IPC endpoint and its identity |
| DoorFS | Kernel subsystem implementing Door endpoint and call semantics |
| NameFS | VFS subsystem publishing a stable pathname identity for a Door service |
| DoorCTL | Named-service registry and generation-control interface |
| Endpoint | Publicly addressable Door capability, distinct from a pathname |
| Generation | One concrete service identity behind a stable service name |
| Shuttle | Scheduler optimization for direct successor handoff |
| ONEWAY | Call mode in which submission does not imply callback completion |
| Strong SHM reference | Region reference bound to an owning Door generation |
| Pair | Two independent SHM event sequences with participant-authorized kernel wait/wake |
| Lane | One event sequence in a Pair; not a thread or payload-buffer slot |
| Allocation chain | Generation-qualified lease descriptor containing bounded offset/length segments |
| Application stream | Bounded sequence of SHM records or chains with explicit publication, credits and final-access ACK |
| Linearization point | Operation event at which its public effect becomes observable |

<a id="notation"></a>

## Notation and cross-reference conventions

Functional requirements use REQ-F-nnn, security requirements REQ-S-nnn,
non-functional requirements REQ-N-nnn, and correctness invariants INV-nnn.
The Pair extension uses REQ-PAIR-nn and INV-PAIR-nn for its focused additions.
ABI rules use ABI-nnn, race hazards RACE-nnn, design decisions DEC-nnn,
library tests TEST-LIB-nnn, kernel tests TEST-KERN-nnn, and measurements
BENCH-nnn.

---
