# DOOR IPC Implementation for FreeBSD

Technical monograph by Eduard Siboshvili · FreeBSD 16-CURRENT · 9 November 2026.

Read the chapters below or open the [full monograph](door-ipc-implementation-for-freebsd_full.md).
The [detailed contents](detailed-contents.md) links to individual sections.

## Start here

- [Programming guide](chapters/14.md) and [Example 26: independent Pair client and server](appendices/03-examples.md#example-26-door-bootstrap).
- [Forwarding and crash tests](appendices/test-catalog/18.md#forwarding-tests) and [qualification record](appendices/01-kernel-and-qualification.md#contents-a1-9-verified-qualification-record).
- [Benchmark methodology and results](appendices/02-benchmarks.md), including the [follow-up HTTP RAM results](appendices/02-benchmarks.md#http-ram-follow-up).
- [nginx and PostgreSQL modules](appendices/04-modules.md) and [local demo setup and benchmark controls](appendices/04-local-demo.md).

## Chapters

- [Glossary and notation](front-matter.md)

### [Part I. Problem statement and requirements]

- [1. Introduction](chapters/01.md)
- [2. Doors model and FreeBSD context](chapters/02.md)
- [3. Requirements and threat model](chapters/03.md)

### [Part II. Architecture and implementation]

- [4. System architecture](chapters/04.md)
- [5. FreeBSD kernel integration](chapters/05.md)
- [6. DoorFS architecture](chapters/06.md)
- [7. Scheduler shuttle protocol](chapters/07.md)
- [8. NameFS, DoorCTL, and service discovery](chapters/08.md)
- [9. libdoor architecture](chapters/09.md)

### [Part III. Observable semantics and correctness]

- [10. ABI and operation semantics](chapters/10.md)
- [11. Locks, references, and lifetime](chapters/11.md)
- [12. Race safety and failure protocols](chapters/12.md)
- [13. Security model](chapters/13.md)

### [Part IV. Programming and verification]

- [14. Programming guide](chapters/14.md)
- [15. Verification methodology](chapters/15.md)

### [Part V. Discussion and conclusions]

- [16. Design decisions and limitations](chapters/16.md)
- [17. FreeBSD and illumos implementation differences](chapters/17.md)
- [18. Conclusions](chapters/18.md)

## Appendices and references

1. [Appendix 1. Test specification and qualification](appendices/01-tests.md) — installation, test catalog and qualification.
2. [Appendix 2. Benchmark methodology and results](appendices/02-benchmarks.md) — methodology and comparisons on Intel Core i5-10400, with separate tables for corrected SHM and the original campaign.
3. [Appendix 3. Example programs and application patterns](appendices/03-examples.md) — examples and application patterns.
4. [Appendix 4. Real nginx and PostgreSQL modules](appendices/04-modules.md) — architecture, configuration and [demo operation](appendices/04-local-demo.md).

- [References and normative sources](references.md)

© 2026 Eduard Siboshvili. The monograph text and diagrams are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Source excerpts retain their original licenses.
