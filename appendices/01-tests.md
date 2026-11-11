[Monograph index](../README.md) · [Full text](../door-ipc-implementation-for-freebsd_full.md)

<a id="appendix-1"></a>

<a id="contents-appendix-1-test-specification-and-qualification"></a>

# Appendix 1. Test specification and qualification

This appendix treats the installed tests as executable ABI and lifetime
specifications. The source tree is authoritative. In the current source it
contains **47 libdoor programs with 349 cases**, **one DoorFS program with five
cases**, and **one NameFS program with eleven cases**: **49 programs and 365
ATF cases in total**.

A passing case establishes only the behavior named in its entry. It does not
promote a partial or missing lower-layer check into coverage. Scheduler and
race-sensitive cases require repeated execution as specified below.

<a id="contents-a1-1-test-environment-and-execution-rules"></a>

## A1.1. Test environment and execution rules

### A1.1.1. Installation identity

Tests execute from `/usr/tests`. Kernel, world, public headers, `libdoor`,
manual pages, examples, and installed tests must come from the same selected
source revision. Record before every qualification campaign:

```sh
cd /usr/tests || exit 1
uname -a
freebsd-version -ku
sysctl -n kern.sched.name
sysctl kern.smp.cpus
sysctl kern.sched.topology_spec
mount
```

DoorFS and NameFS are integrated kernel facilities in the system described by
this monograph. Their configured presence and the expected ABI must be verified
before the first case; dynamic component removal is not a test dimension.

### A1.1.2. Required test layers

Run the three suites independently from the test root:

```sh
cd /usr/tests || exit 1
kyua test lib/libdoor
kyua test sys/fs/doorfs
kyua test sys/fs/namefs
```

| Layer | Establishes | Does not establish alone |
|---|---|---|
| `lib/libdoor` | Public API, runtime, pools, RPC, descriptors, ONEWAY, SHM, DoorCTL, stress | Complete hostile raw-selector validation or VFS reclamation |
| `sys/fs/doorfs` | Raw Door file operations and kernel-facing ABI behavior represented by its cases | Worker-runtime or NameFS behavior |
| `sys/fs/namefs` | VFS publication, open, rebind, recovery, NullFS, and jail interaction | Raw selector or callback-pool safety |

A qualification result is the conjunction of all three layers. Preserve the
Kyua result id and report for each invocation.

### A1.1.3. Selecting and repeating cases

Programs and individual cases are selected by installed path:

```sh
kyua test lib/libdoor/signals/signals_test
kyua test \
  lib/libdoor/signals/signals_test:repeated_interrupt_and_recovery
kyua test \
  lib/libdoor/lifetime_extended/lifetime_extended_test:owner_exit_during_pending_call
kyua test sys/fs/namefs/namefs_test:owner_exit_recovery_rebind
```

Use `kyua list` when a selector is uncertain. Recently corrected races are
first repeated in isolation and then exercised in the complete-suite loop.
A timeout bounds a run but never proves that a state transition occurred.
Tests must wait for an observable predicate: pending/active counts, a pipe or
CV barrier, callback acknowledgement, child exit, generation change, or mount
state. Arbitrary delay is not evidence.

### A1.1.4. Mass execution

The established `door_mass_test.sh` runner defaults to `/usr/tests`, records
the scheduler, runs all three suites in order, stops on the first nonzero
status, and writes a cumulative log plus an atomically replaced progress file.

```sh
chmod 755 /root/door_mass_test.sh
/root/door_mass_test.sh 1000
cat /var/tmp/door-tests-progress
```

The progress record must include state, scheduler, current and total iteration,
suite, exit status, timestamp, and log path. A complete iteration means that
all three suites succeeded once. Partial iterations and isolated repetitions
are recorded separately and never added to the complete-iteration baseline.

### A1.1.5. Result retention

On first failure, preserve:

- exact test selector, scheduler, CPU placement, iteration, and timestamps;
- Kyua result id, `kyua report`, and `kyua report-junit` output;
- the Kyua work directory when available;
- userspace cores, kernel log, and crash dump where applicable;
- process/thread stacks and wait channels before killing a hung test;
- relevant production object/resource counters before and after cleanup.

Do not rerun first and investigate later. An intermittent failure remains a
failure even if a subsequent campaign succeeds.

<a id="contents-a1-2-test-case-description-template"></a>

## A1.2. Test-case description template

Every case in A1.3–A1.5 is described by the following contract:

```markdown
##### `suite/program:test_case`

**Purpose:** The exact invariant or public behavior being verified.

**Setup:** Doors, workers, processes, descriptors, mappings, names, and fault
state created by the test.

**Sequence:**
1. Observable operation or synchronization step.
2. State transition being forced.
3. Operation whose result is checked.

**Expected result:** Return value, errno, output data, descriptor ownership,
kernel state, and final object lifetime.

**Concurrency:** Participating processes/threads and every synchronization
point. State why no arbitrary delay is being used as proof.

**Regression:** The historic or plausible bug prevented by the case.

**Cleanup:** Revoke, unregister, unmap, descriptor close, worker drain, and
resource-accounting expectations.
```

Program entries additionally record source path, case count, shared fixtures,
and fixed workload constants. A source-line number is a review aid, not a
stable identifier; the suite/program/case tuple is authoritative.

### A1.2.1. Common concurrency and cleanup rules

Use readiness predicates, joins, completion barriers, and named kernel gates
to establish the intended order. A timeout bounds failure; it is not evidence
of the order. Request-callback last access and completion-endpoint drain are
different events. Revoke endpoints, join participants, remove each unique VA
range once, close every private fd, and compare live-resource baselines where
the case claims absence of leaks. Phase tests need four available CPUs: the
controller, gate-holding worker, and two clients have separate CPU assignments.

Existing case cards below retain specific operations and checked errors; the
new source-derived cards record assertions alongside scenario descriptions.

<a id="contents-a1-3-libdoor-api-tests"></a>

## A1.3. libdoor API tests

The entries below were compared with every `ATF_TC*` declaration and every
`ATF_TP_ADD_TC` registration in `lib/libdoor/tests` in the current implementation.
The source inventory registers 349 cases. The previous 335 case descriptions
are supplemented by seven Pair cases in §A1.3.17 and seven forwarding cases in
§A1.3.18; standalone models are
listed separately and are not counted as ATF programs.

### A1.3.1. Test-program inventory

| Directory | Test program | Cases |
|---|---|---:|
| `abi` | `abi_test` | 3 |
| `access_stats` | `access_stats_test` | 5 |
| `api` | `api_validation_test` | 7 |
| `async` | `async_kernel_limits_test` | 4 |
| `async` | `async_test` | 14 |
| `attributes` | `attributes_test` | 10 |
| `basic` | `basic_test` | 12 |
| `cancellation` | `cancellation_test` | 9 |
| `capsicum` | `capsicum_test` | 8 |
| `concurrency_stress` | `concurrency_stress_test` | 8 |
| `depletion` | `depletion_test` | 4 |
| `descriptor` | `descriptor_validation_test` | 8 |
| `descriptor_graph` | `descriptor_graph_test` | 6 |
| `descriptor_limits` | `descriptor_limits_test` | 7 |
| `descriptor_ownership` | `descriptor_ownership_test` | 9 |
| `descriptor_types` | `descriptor_types_test` | 8 |
| `doorctl` | `doorctl_test` | 5 |
| `doorctl_oneway` | `doorctl_oneway_test` | 5 |
| `doorctl_oneway_shm` | `doorctl_oneway_shm_test` | 7 |
| `doorctl_shm` | `doorctl_shm_test` | 5 |
| `fault_injection` | `fault_injection_test` | 7 |
| `final_stress` | `final_stress_test` | 2 |
| `fork_exec` | `fork_exec_test` | 16 |
| `huge_payload` | `huge_payload_test` | 12 |
| `legacy_pool` | `legacy_pool_test` | 4 |
| `lifetime` | `lifetime_test` | 9 |
| `lifetime_extended` | `lifetime_extended_test` | 9 |
| `mass_stress` | `mass_stress_test` | 2 |
| `mixed_payload` | `mixed_payload_test` | 9 |
| `oneway` | `oneway_test` | 11 |
| `persistent_shm` | `persistent_shm_test` | 13 |
| `phases` | `invariant_test` | 8 |
| `phases` | `phase_test` | 8 |
| `pool_lifecycle` | `pool_lifecycle_test` | 14 |
| `process_control` | `process_control_test` | 4 |
| `race_hardening` | `race_hardening_test` | 3 |
| `recursive` | `recursive_test` | 3 |
| `recursive` | `forward_test` | 7 |
| `resource_limits` | `resource_limits_test` | 6 |
| `shared_memory` | `shared_memory_test` | 11 |
| `shm_fd_reuse` | `shm_fd_reuse_test` | 2 |
| `shm_pair` | `shm_pair_test` | 7 |
| `shm_ring` | `shm_ring_test` | 14 |
| `shm_view` | `shm_view_test` | 10 |
| `signals` | `signals_test` | 4 |
| `worker_hardening` | `worker_hardening_test` | 3 |
| `xcreate` | `xcreate_test` | 7 |
| **Total** | **47 programs** | **349** |

## Continue reading

- [A1.3.2. ABI and API validation](test-catalog/02.md)
- [A1.3.3. Core RPC semantics](test-catalog/03.md)
- [A1.3.4. Cancellation, signals, and process control](test-catalog/04.md)
- [A1.3.5. Lifetime and worker pools](test-catalog/05.md)
- [A1.3.6. Descriptor semantics](test-catalog/06.md)
- [A1.3.7. ONEWAY](test-catalog/07.md)
- [A1.3.8. SHM and rings](test-catalog/08.md)
- [A1.3.9. Named service and recovery](test-catalog/09.md)
- [A1.3.10. Security and failure injection](test-catalog/10.md)
- [A1.3.11. Stress and race hardening](test-catalog/11.md)
- [A1.3.12. Remaining suites](test-catalog/12.md)
- [A1.3.13. Maintenance and review rules](test-catalog/13.md)
- [A1.3.14. Native async, deterministic phases, compatibility, and page views](test-catalog/14.md)
- [A1.3.15. Strengthened existing ownership cases](test-catalog/15.md)
- [A1.3.16. Reply MOVE commit versus revoke](test-catalog/16.md)
- [A1.3.17. Pair events, allocation chains, and stream-protocol invariants](test-catalog/17.md)
- [A1.3.18. Synchronous forwarding and process failure](test-catalog/18.md)
- [A1.4. DoorFS raw-kernel tests](01-kernel-and-qualification.md)
