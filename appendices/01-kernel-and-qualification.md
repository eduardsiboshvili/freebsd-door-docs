[Monograph index](../README.md) · [Full text](../door-ipc-implementation-for-freebsd_full.md)

<a id="contents-a1-4-doorfs-raw-kernel-tests"></a>

## A1.4. DoorFS raw-kernel tests

These entries were checked against the current kernel-test source; the case names and counts match exactly.

### A1.4.1. Program: `doorfs_test`

**Source:** `tests/sys/fs/doorfs/doorfs_test.c`

**Build/install:** The program is built as `ATF_TESTS_C+=doorfs_test`, links
against `libdoor` and `libpthread`, and is installed below
`/usr/tests/sys/fs/doorfs`.

The program creates a managed echo Door and concentrates on the file object
returned by CREATE. `call_echo()` submits the fixed byte string
`"doorfs-file"`, requires a same-sized reply, and compares the bytes exactly.
That helper turns a successful call into an independent liveness probe for a
duplicated or inherited file capability.

#### Coverage summary

| Area | Current coverage |
|---|---|
| File attributes through `fstat()` | `file_stat` |
| Rejection of stream `read()`/`write()` | `non_stream_io_rejected` |
| Duplicate file-reference lifetime | `duplicate_holds_file` |
| Revoke visibility through duplicate fds | `revoke_visible_to_duplicate` |
| Fork-inherited capability | `inherited_file_capability` |
| Raw `SYS_door` selectors | **Not covered by this program** |
| Historical ioctl transport | **Not covered by this program** |
| Malformed kernel ABI blocks/pointers | **Not covered by this program** |
| Capsicum rights | **Not covered by this program**; covered at the libdoor test layer only |
| Owner exit and worker loss | **Not covered by this program** |
| Kernel object accounting/baselines | **Not covered** |
| Partial allocation/copyin/copyout rollback | **Not covered** |

The missing rows are explicit coverage gaps. A successful run of these five
cases must not be cited as proof of complete raw-selector hardening, kernel
object accounting, or failure rollback.

#### `doorfs/doorfs_test:file_stat`

**Purpose:** Verify the externally visible `struct file` metadata of a newly
created Door descriptor.

**Setup:** Create one managed echo Door with `door_create()`. No pathname,
duplicate, child process, descriptor transfer, or mapping is involved.

**Sequence:**

1. Create the Door and retain its returned fd.
2. Call `fstat()` on that fd.
3. Check file type, mode, link count, and close the only visible descriptor.

**Expected result:** `fstat()` succeeds. `st_mode` reports a regular-file
presentation with permissions `0600`; `st_nlink` is exactly `1`. Closing the fd
succeeds. The regular-file presentation is an observability/compatibility
choice and does not authorize stream I/O.

**Concurrency:** The creating thread and the initial libdoor-managed worker are
present. No callback is invoked. Creation waits for worker readiness, so no
delay is used as proof.
incorrect vnode/file reporting in `fstat`/`procstat`, and changes that expose
broader permissions than intended.

**Cleanup:** Close the Door fd. Final library/kernel references must unwind
without requiring an RPC or pathname publication.

#### `doorfs/doorfs_test:non_stream_io_rejected`

**Purpose:** Verify that a Door capability cannot be treated as a byte stream.

**Setup:** Create one managed echo Door and a one-byte userspace buffer.

**Sequence:**

1. Call `read(fd, &byte, 1)` and record failure.
2. Clear `errno`, call `write(fd, &byte, 1)`, and record failure.
3. Close the Door normally.

**Expected result:** Both `read()` and `write()` return `-1` and set a nonzero
`errno`. The test intentionally does not freeze one exact errno; its ABI
invariant is that ordinary stream operations never consume, expose, or mutate
Door RPC state.

**Concurrency:** No RPC is dispatched. The test thread performs both file
operations sequentially while the managed worker remains idle.
handlers, data disclosure through an unimplemented read path, or corruption of
call queues by stream writes.

**Cleanup:** Close the Door fd. The failed operations must not create a call,
server assignment, buffer, or retained reference.

#### `doorfs/doorfs_test:duplicate_holds_file`

**Purpose:** Verify that `dup()` creates an independent file-descriptor
reference that keeps the exact Door generation callable after the original fd
is closed.

**Setup:** Create one echo Door, duplicate its fd with `dup()`, and retain both
numeric descriptors initially.

**Sequence:**

1. Close the original descriptor.
2. Perform the fixed echo RPC through the duplicate.
3. Compare reply bytes and close the duplicate.

**Expected result:** Original close succeeds and does not revoke or destroy the
endpoint. `door_call()` through the duplicate succeeds and echoes the exact
request. The final duplicate close succeeds and releases its file reference.

**Concurrency:** One client thread synchronizes with one managed callback by
the synchronous CALL/RETURN completion point. No timing assumption is used.
the underlying file reference, premature revoke on non-final close, and
use-after-free through a surviving duplicate.

**Cleanup:** Close both descriptors in ownership order. No publication or child
exists; after the final close, all test-owned file references must be gone.

#### `doorfs/doorfs_test:revoke_visible_to_duplicate`

**Purpose:** Verify that revoke is endpoint-wide and is observed through every
descriptor referring to that generation.

**Setup:** Create an echo Door and duplicate its fd. Both descriptors reference
the same `struct door` generation.

**Sequence:**

1. Invoke `door_revoke()` through the original fd.
2. Attempt the fixed echo call through the duplicate.
3. Check the allowed lifetime error and close both descriptors.

**Expected result:** Revoke succeeds. The duplicate does not remain a hidden
live endpoint: its call fails with `EBADF`, `ECANCELED`, or `ENXIO`, depending
on the precise call/revoke observation point. It must never call the callback,
succeed against a replacement, or hang.

**Concurrency:** Revoke returns before the follow-up call is issued, so the
ordering is deterministic. The worker may be unwinding, which accounts for the
enumerated error set; no delay selects an outcome.
silent generation retargeting, and workers remaining indefinitely blocked
after revoke.

**Cleanup:** Close the duplicate and original fd after revoke. All workers and
file references must be eligible for ordinary asynchronous teardown.

#### `doorfs/doorfs_test:inherited_file_capability`

**Purpose:** Verify that a Door file capability inherited across `fork()`
remains callable even though the child does not inherit the owner's callback
registry or worker threads.

**Setup:** The parent creates an echo Door and forks. The child inherits the
client fd; the owner and managed server worker remain in the parent.

**Sequence:**

1. Child calls the inherited fd using the fixed echo transaction.
2. Child exits with status zero only on a correct reply.
3. Parent uses `waitpid()` and verifies normal successful exit.

**Expected result:** The child call succeeds through the inherited kernel file
capability. `waitpid()` returns the child, `WIFEXITED` is true, and exit status
is zero. The child is a client only; it does not acquire a callback pointer by
inheritance.

**Concurrency:** Parent owner/worker and child client are distinct processes.
The synchronous RPC and `waitpid()` are the two completion barriers. No sleep
is used.
calls, lost file references during fork, and confusion between inheriting a
capability and inheriting server runtime state.

**Cleanup:** Child exits; parent reaps it and closes the original fd. No zombie,
active call, or child-held file reference may remain.

<a id="contents-a1-5-namefs-vfs-tests"></a>

## A1.5. NameFS VFS tests

These entries were checked against the current kernel-test source; the case names and counts match exactly.

### A1.5.1. Program: `namefs_test`

**Source:** `tests/sys/fs/namefs/namefs_test.c`

**Build/install:** Built as `ATF_TESTS_C+=namefs_test`, linked with `libdoor` and
`libpthread`, and installed below `/usr/tests/sys/fs/namefs`. All cases require
root because they mount/unmount NameFS, compose mounts, or create jails.

The common fixture creates a private directory and an existing regular-file
placeholder named `door`. `fattach()` performs the NameFS file mount on that
target. `marker_server()` returns its generation-specific cookie string, so a
successful call distinguishes old and replacement endpoints even though the
filesystem pathname and NameFS vnode identity remain stable.

Every case with `ATF_TC_WITH_CLEANUP` has best-effort cleanup that attempts
`fdetach()`, forced unmount of a fixture directory when necessary, unlink, and
`rmdir`. Cleanup is recovery after a failed assertion; its use of `MNT_FORCE`
is not itself a successful forced-unmount test.

#### Coverage summary

| Area | Current cases |
|---|---|
| Basic mount, lookup/open, call, detach | `basic_publish` |
| Rebind with old/new open generations | `rebind_preserves_open_generation` |
| Owner-exit recovery and rebind | `owner_exit_recovery_rebind` |
| Detach with an already-open capability | `detach_preserves_open_capability` |
| Revoked target rejection | `dead_door_open_and_attach_rejected` |
| Invalid non-Door target fd | `invalid_target_rejected` |
| Mounted-on vnode unlink/rename exclusion | `mounted_path_mutation_rejected` |
| NullFS above NameFS | `nullfs_over_namefs` |
| NameFS above NullFS | `namefs_over_nullfs` |
| Default host-to-jail denial | `host_to_jail_denied` |
| Explicit cross-jail permission | `host_to_jail_cross_allowed` |
| Direct malformed `nmount()` option matrix | **Not covered** |
| Successful direct ordinary/forced `unmount(2)` of NameFS | **Not covered** |
| Concurrent open/rebind/unmount/reclaim stress | **Not covered** |

#### `namefs/namefs_test:basic_publish`

**Purpose:** Verify the complete ordinary NameFS publication lifecycle and the
filesystem identity visible to applications.

**Setup:** Create directory `basic`, create its `door` placeholder, create a
marker Door returning `"basic"`, and retain the server fd.

**Sequence:**

1. Attach the Door to the placeholder with `fattach()`.
2. Use `statfs()` to identify the mounted filesystem as `namefs`.
3. Open the path, call the endpoint, and require marker `"basic"`.
4. Close the client, detach, close the server, and remove the fixture.

**Expected result:** Attach, lookup/open, synchronous call, close, and detach all
succeed. `f_fstypename` is exactly `namefs`; the opened fd is a usable Door
capability, not the underlying placeholder file.

**Concurrency:** Test process and managed worker synchronize by CALL/RETURN.
Mount and detach operations are sequential. No delay is used.
vnode/file, open failing to acquire the Door reference, or detach leaking the
mount/root vnode.

**Cleanup:** Normal path closes client and server, detaches, unlinks, and removes
the directory. ATF cleanup repeats safe best-effort detach/forced-unmount and
path removal after an early failure.

#### `namefs/namefs_test:rebind_preserves_open_generation`

**Purpose:** Verify stable NameFS pathname/vnode identity while open file
descriptors remain pinned to the generation selected at their own open.

**Setup:** Create two marker Doors returning `"first"` and `"second"`. Attach
the first, then open `old_client` before rebind.

**Sequence:**

1. Reattach the second Door at the already mounted path.
2. Open `new_client` after rebind.
3. Call both clients and compare their distinct markers.

**Expected result:** Rebind succeeds atomically. `old_client` continues to call
`"first"`; `new_client` calls `"second"`. Existing capabilities never silently
retarget, and new opens never obtain a half-updated target.

**Concurrency:** Operations are deliberately ordered in one test thread;
synchronous replies prove generation selection. The case tests temporal
identity without relying on scheduling delay.
generation confusion, dropping the old target reference too early, and
non-atomic `nm_target` replacement.

**Cleanup:** Close both clients, detach the stable publication, close both
server fds, and remove the placeholder/directory. Both generation references
must drain independently.

#### `namefs/namefs_test:owner_exit_recovery_rebind`

**Purpose:** Verify recovery of the same named vnode after the original Door
owner exits, while preserving stale-capability isolation.

**Setup:** Create `recovery/door`, two synchronization pipes, and fork an owner
child. The child creates/attaches a Door returning `"departed"`, signals READY,
and waits for a control byte. The parent opens both a Door client and an
`O_PATH` recovery anchor before permitting owner exit.

**Sequence:**

1. Parent releases the child and observes actual exit with `waitpid()`.
2. Verify a new `open(O_RDWR)` of the dead publication fails with `EBADF` or
   `ENXIO`.
3. Create a replacement Door returning `"replacement"` and rebind it at the
   same path while the recovery anchor preserves namespace reachability.
4. Verify the old client fails with a lifetime error.
5. Open a new client and require the replacement marker.

**Expected result:** Owner exit makes the old generation dead; merely retaining
the NameFS identity does not keep calls alive. Rebind succeeds without creating
a new pathname identity. The stale client fails with `EBADF`, `ECANCELED`, or
`ENXIO`; the new client calls only the replacement.

**Concurrency:** Parent and owner child synchronize with READY/control pipes.
`waitpid()` proves the owner entered and completed process exit; sending a
signal or sleeping is not used as an exit barrier. Synchronous calls prove old
and new generation behavior.

**Regression:** Protects the recovery path that previously risked losing the
mounted vnode during owner teardown, rebinding an arbitrary lower/upper vnode,
or retargeting stale descriptors to the replacement generation.

**Cleanup:** Close recovery anchor and stale/new clients, detach, close the
replacement fd, remove paths, and reap the child. Cleanup must release old and
new Door references and the stable NameFS vnode exactly once.

#### `namefs/namefs_test:detach_preserves_open_capability`

**Purpose:** Verify that unpublishing a pathname affects future lookup but does
not revoke a capability opened before detach.

**Setup:** Attach a Door returning `"held"` and open a client through NameFS.

**Sequence:**

1. Call `fdetach()` while the client fd remains open.
2. Open the restored underlying placeholder and call `door_call(fd, NULL)`,
   requiring only that the invalid call fails.
3. Call the pre-detach client and require `"held"`.

**Expected result:** Detach succeeds and a new open of the pathname succeeds.
The pre-existing client remains a valid generation-pinned Door capability and
completes its call. The `door_call(fd, NULL)` check is not sufficient to prove
the new fd's type because `libdoor` rejects a NULL `door_arg_t` before it needs
to exercise that fd; a future revision should use `fstat()` plus a valid CALL
argument and require `ENOTTY` from the restored regular file.

**Concurrency:** Sequential detach and probes remove timing ambiguity. The
managed callback participates only in the final old-capability call.
Door files. Continued lookup/namecache exposure is only partially checked by
the current new-open probe and should be strengthened as described above.

**Cleanup:** Close underlying-file fd, old Door client, and server; unlink the
placeholder and remove the directory. No NameFS mount remains after detach.

#### `namefs/namefs_test:dead_door_open_and_attach_rejected`

**Purpose:** Verify that revoked Door generations cannot be newly opened from
an existing publication or attached at another pathname.

**Setup:** Attach a live Door at `dead-open/door`, revoke it, and prepare a
second placeholder at `dead-attach/door`.

**Sequence:**

1. Attempt to open the first published path after revoke.
2. Detach the first publication.
3. Attempt `fattach()` of the same revoked server fd to the second path.

**Expected result:** Opening the revoked publication fails with `EBADF` or
`ENXIO`. Attaching the revoked Door fails with exactly `EBADF`. No new client or
publication reference is acquired on either failure.

**Concurrency:** Revoke completes before both checks. No worker callback or
timing race is required.
revoked capability, and reference leaks during failed open/attach validation.

**Cleanup:** Detach first mount, close server fd, remove both placeholders and
directories. ATF cleanup handles either fixture after partial failure.

#### `namefs/namefs_test:invalid_target_rejected`

**Purpose:** Verify that NameFS accepts only a valid live Door fd as `doorfd`.

**Setup:** Create `invalid/door` and open `/dev/null` read-only as a deliberately
wrong target object.

**Sequence:**

1. Call `fattach(/dev/null-fd, path)`.
2. Check failure and exact errno.
3. Close the invalid fd and remove the untouched fixture.

**Expected result:** `fattach()` returns `-1` with `EINVAL`. No mount is
published, the placeholder remains an ordinary file, and the unrelated
`/dev/null` file is not consumed or modified.

**Concurrency:** Single-threaded validation before any mount publication.
objects as Doors, and partial mount/reference publication before target
validation completes.

**Cleanup:** Close `/dev/null`, best-effort detach, unlink, and remove directory.
There must be no mount or Door reference from the rejected attempt.

#### `namefs/namefs_test:mounted_path_mutation_rejected`

**Purpose:** Verify VFS file-mount protection of the mounted-on pathname.

**Setup:** Attach a marker Door at `mutation/door`.

**Sequence:**

1. Attempt `unlink(path)` while NameFS is mounted.
2. Attempt `rename(path, "mutation/renamed")`.
3. Detach and then remove the original placeholder normally.

**Expected result:** Both unlink and rename fail with `EBUSY` while mounted.
After detach, unlink succeeds. No alternate renamed entry is created.

**Concurrency:** Sequential namespace mutations under ordinary VFS locking; no
callback is needed.
active file mount, dangling `nm_mountvp`, and namecache references to an
unreachable publication.

**Cleanup:** Detach, close server, unlink original and any unexpected renamed
path, and remove the directory.

#### `namefs/namefs_test:nullfs_over_namefs`

**Purpose:** Define behavior when NullFS covers a directory containing a NameFS
file mount: the lower hidden file mount must not leak through the upper NullFS
namespace, while directly opened capabilities remain valid.

**Setup:** Create source and target directories, attach a Door returning
`"null-over"` at the source path, then mount NullFS from source onto target.

**Sequence:**

1. Open `target/door` through NullFS and attempt a Door call.
2. Require `ENOTTY`, demonstrating that the covered lower NameFS file mount was
   not traversed through the new upper mount.
3. Open the direct source NameFS path and call successfully.
4. Unmount NullFS and call again through the already-open direct client.

**Expected result:** The NullFS view opens a non-Door object and Door call fails
with exactly `ENOTTY`. Direct NameFS open returns the correct marker before and
after NullFS unmount. Unmount does not invalidate the already-open direct Door
capability.

**Concurrency:** Sequential mount/open/call/unmount operations; synchronous
calls are the endpoint barriers.
file mount through NullFS, and prevents upper-mount teardown from dropping an
unrelated directly opened NameFS capability.

**Cleanup:** Unmount NullFS, close direct client, detach NameFS, close server,
and remove both directory trees. Cleanup may force-unmount a residual fixture.

#### `namefs/namefs_test:namefs_over_nullfs`

**Purpose:** Verify that NameFS can be mounted on a pathname reached through a
NullFS view and can be detached through the same composed namespace.

**Setup:** Mount NullFS from `under-source` onto `under-target`; create the
placeholder in the source and attach a Door returning `"name-over"` to the
corresponding target view.

**Sequence:**

1. Open `under-target/door` and call the Door.
2. Detach NameFS through the NullFS view.
3. Unmount NullFS and remove the underlying fixture.

**Expected result:** Attach/open/call succeeds and returns `"name-over"`.
`fdetach(view)` succeeds while NullFS remains mounted; ordinary NullFS unmount
then succeeds. Reference ownership between the upper NameFS file mount and
lower NullFS vnode remains balanced.

**Concurrency:** Sequential VFS composition and synchronous callback. No delay.
the host filesystem directly, incorrect FSID/mount lookup during detach, and
lower-vnode leaks after layered unmount.

**Cleanup:** Detach NameFS, unmount NullFS, close server, unlink lower
placeholder, and remove source/target directories.

#### `namefs/namefs_test:host_to_jail_denied`

**Purpose:** Verify default jail isolation for a host-owned NameFS Door.

**Setup:** Host creates and attaches a Door without `DOOR_CROSS_JAIL`, resolves
the publication to an absolute path, then forks a child that enters a new jail
rooted at `/`.

**Sequence:**

1. Jailed child attempts `open(path, O_RDWR)`.
2. Child exits zero only when open fails with `EACCES`.
3. Parent reaps the child and checks its encoded step result.

**Expected result:** Cross-prison open is denied with exactly `EACCES`; no Door
fd or client reference is installed in the jailed process. Parent-side
publication remains intact until normal detach.

**Concurrency:** Host parent/server and jailed child are separate processes.
`waitpid()` is the completion barrier; child exit codes identify the failing
step without timing assumptions.
boundaries and ensures NameFS open performs Door jail policy checks rather than
trusting filesystem visibility alone.

**Cleanup:** Parent detaches, closes server, removes fixture, and reaps child.
The child holds no successful capability.

#### `namefs/namefs_test:host_to_jail_cross_allowed`

**Purpose:** Verify the explicit `DOOR_CROSS_JAIL` exception and ensure it grants
only the intended Door capability behavior.

**Setup:** Same jail fixture as the denial case, but create the host Door with
`DOOR_CROSS_JAIL` and marker `"jail"`.

**Sequence:**

1. Jailed child opens the absolute NameFS path.
2. Child calls the endpoint and requires marker `"jail"`.
3. Child closes the fd and exits; parent verifies success.

**Expected result:** Open and synchronous call succeed only because the Door
was explicitly created with cross-jail policy. The child receives a normal
generation-pinned capability; the flag does not transfer owner/admin authority.

**Concurrency:** Host owner/worker and jailed child client synchronize by
CALL/RETURN and `waitpid()`.
cross-jail service and overly broad bypasses that would make all host Doors
visible/callable from a jail.

**Cleanup:** Child closes client and exits; parent reaps it, detaches NameFS,
closes server, and removes fixture.

<a id="contents-a1-6-scheduler-and-race-qualification"></a>

## A1.6. Scheduler and race qualification

#### Ordinary and instrumented libdoor builds

| Build | Registered libdoor cases | Kernel requirement |
|---|---:|---|
| Default tests | 332 | Functional Doors support |
| `WITH_DOOR_TESTS=yes` | 349 | Built-in DOORFS, DOORFS_TEST, INVARIANTS; four available CPUs for phase tests |

The extra 17 cases are 16 phase/invariant cases and one raw async publication
drain case. They are conditionally built, not missing failures or skipped
results. Release kernels must omit DOORFS_TEST; it deliberately requires
INVARIANTS. Build the full tests with matching private headers:

```sh
make -C /usr/src/lib/libdoor/tests WITH_DOOR_TESTS=yes clean all install
kyua test -k /usr/tests/lib/libdoor/Kyuafile
```

### A1.6.1. Scheduler matrix

Public semantics must be qualified under both supported schedulers. Record the
active scheduler from `kern.sched.name`; a configured kernel name is not a
substitute for the runtime value.

| Dimension | Required coverage | Relevant mechanisms |
|---|---|---|
| Scheduler | 4BSD and ULE | Global `sched_lock` versus per-CPU TDQ ownership |
| Placement | Unbound, same CPU, different core, SMT sibling where available | Local reservation, affinity fallback, remote wake |
| Call state | Pending, assigned, active, interrupted, revoked | Park publication, abort, resume, ordinary fallback |
| Workload | Synchronous, native async, ONEWAY, SHM, page views, descriptor-bearing sync calls, recursive | Different wake, completion, and ownership paths |
| Process state | Signal, stop/continue, worker exit, owner exit, fork/exec | Lifecycle integration |

CPU placement is required for scheduler-sensitive focused tests and benchmark
workloads, not for structure-only ABI validation. Functional results must be
identical across schedulers even though lock topology and latency differ.

### A1.6.2. Focused race protocol

For each corrected race:

1. identify the unsafe interleaving and invariant;
2. make the precondition observable with a barrier or protected counter;
3. repeat the focused case beyond its previously observed failure horizon;
4. run the complete three-suite iteration repeatedly;
5. preserve the first failure and do not combine partial campaigns;
6. execute under both 4BSD and ULE when scheduler interaction is possible;
7. verify resource state returns to baseline.

Priority focused cases include:

```text
signals/signals_test:repeated_interrupt_and_recovery
lifetime_extended/lifetime_extended_test:owner_exit_during_pending_call
lifetime_extended/lifetime_extended_test:worker_exit_aborts_active_call
doorctl_oneway/doorctl_oneway_test:replacement_generation
doorctl_oneway_shm/doorctl_oneway_shm_test:replacement_generation_shm
race_hardening/race_hardening_test:revoke_close_same_door_race
```

### A1.6.3. Verified development milestone

The recorded project milestone reported the following complete mass-test
campaigns:

| Scheduler | Complete iterations | Suites per iteration | Result |
|---|---:|---|---|
| 4BSD | 1000 | `lib/libdoor`, `sys/fs/doorfs`, `sys/fs/namefs` | Successful |
| ULE | 1000 | `lib/libdoor`, `sys/fs/doorfs`, `sys/fs/namefs` | Successful |

This record establishes the reported development milestone, not the final
release certificate. The final publication must attach the exact source
revision, kernel configuration, host identity, dates, test logs, and Kyua
result databases. Numerical performance observations are not part of
this correctness record.

<a id="contents-a1-7-requirements-coverage-matrix"></a>

## A1.7. Requirements coverage matrix

The matrix is a review index linking the normative identifiers in Chapter 3 to
the implementation and the detailed cases above. `Covered` means a focused
case directly checks the critical transition; `Layered` means several cases
jointly establish it; `Partial` leaves a named lower-layer or variant gap.

| Requirements/invariants | Primary implementation | Principal tests | Status/variants |
|---|---|---|---|
| `REQ-F-001`–`REQ-F-011`; `INV-001`–`INV-005`, `INV-020`, `INV-025` | `door_core.c`, `door_rpc.c`, `door_syscall.c`, `door_file.c`; `door_create.c`, `door_call.c`, `door_server.c` | `abi`, `api`, `basic`, `mixed_payload`, `huge_payload`; DoorFS file cases | Layered; fast-selector forced equivalence remains partial |
| `REQ-F-012`–`REQ-F-017`; `INV-003`–`INV-005`, `INV-008`, `INV-018` | cancel/revoke RPC paths; `kern_exit.c`, `kern_thread.c`, binding registry | `cancellation`, `signals`, `lifetime`, `lifetime_extended`, `worker_hardening`, `process_control` | Covered under 4BSD/ULE; resource baselines remain release evidence |
| `REQ-F-018`–`REQ-F-022`; `INV-019`–`INV-025` | `door_xfer.c`, descriptor capture/install/MOVE acknowledgement, filedesc integration | `mixed_payload`, `huge_payload`, all descriptor suites, `fault_injection` | Layered; focused fd-number-reuse MOVE barrier is a gap |
| `REQ-F-023`–`REQ-F-026`; `INV-026`, `INV-027`, `INV-030` | ONEWAY submit, queue/server reference transfer, revoke paths | `oneway`, `doorctl_oneway`, `doorctl_oneway_shm`, stress suites | Covered; application completion remains protocol-defined |
| `REQ-F-027`–`REQ-F-031`; `INV-006`, `INV-028`–`INV-030` | `door_shm.c`, mapping and strong-reference paths | `shared_memory`, `persistent_shm`, `doorctl_shm`, `doorctl_oneway_shm` | Covered for current API; accounting rollback matrix partial |
| `REQ-F-032`; `INV-031` | `door_shm_ring.c` ten-operation SPSC API | all fourteen `shm_ring` cases | Covered for documented SPSC model |
| `REQ-F-033`–`REQ-F-039`; `INV-002`, `INV-007`, `INV-032`–`INV-035` | `door_ctl.c`, NameFS VFS/vnode operations, NullFS composition | DoorCTL families; all eleven NameFS cases | Layered; malformed mount/reclaim stress gaps remain |
| `REQ-F-040`, `REQ-F-041`; `INV-040` | libdoor registry, pools, TLS, factory, fork handlers | `pool_lifecycle`, `legacy_pool`, `xcreate`, `depletion`, `fork_exec`, `worker_hardening` | Covered at public runtime layer |
| `REQ-F-042`–`REQ-F-045` | Native async kernel dispatch, `door_async.c`, reply workers and close | `async_test`, `async_kernel_limits_test`, ordered invariant and phase cases | Native/ELF32 complete runs reported; raw drain distinguishes publication from library deduplication |
| `REQ-F-046`–`REQ-F-049` | Kernel page loans and VM mapping; `door_shm_view.c` handles | All ten `shm_view` cases; fd-reuse and live-object cycles | Native/ELF32 runs reported; exhaustive injected-failure coverage remains separate |
| `INV-008`–`INV-013` | scheduler-neutral shuttle API, `sched_4bsd.c`, `sched_ule.c`, sleepqueue/signal paths | `signals`, cancellation, recursive/process-control/stress cases; scheduler campaign | Layered under 4BSD/ULE; focused affinity assertion and UP qualification remain gaps |
| `INV-014`–`INV-019`, `INV-036`–`INV-040` | Door/call/binding/file/SHM/vnode refs and lock order | lifetime, worker, descriptor, SHM, NameFS, fault and stress suites | Layered; complete kernel object-accounting test remains partial |
| `REQ-S-001`–`REQ-S-009` | raw syscall validation, binding authority, credential snapshots, Capsicum lookup | `api`, `fault_injection`, `capsicum`, `basic:credentials`, DoorFS cases | Partial: complete raw selector/right matrix not yet present |
| `REQ-S-010`–`REQ-S-018` | jail policy, descriptor transaction, SHM generation/range validation | NameFS jail cases; descriptor, Capsicum, SHM, DoorCTL replacement suites | Layered; cross-jail ADMIN/SHM denial matrix remains partial |
| `REQ-S-019`–`REQ-S-024` | object quotas, provisional charging, rollback and copyout hygiene | `resource_limits`, `fault_injection`, stress, access statistics | Layered; authoritative all-object baseline case remains a gap |
| `REQ-N-001`–`REQ-N-006` | public headers/symbol map, versioned ABI, scheduler fallback | `abi:public_layout`, API suites, 4BSD/ULE campaigns | Layered; non-LP64 compatibility requires explicit qualification |
| `REQ-N-007`–`REQ-N-014` | generic/fast convergence, bounded hints, production source hygiene | broad functional suites; Chapter 16 review records | Measured configurations in Appendix 2; unmeasured scenarios remain outside the claim |
| `REQ-N-015`–`REQ-N-020` | Chapter 15 method and this appendix | source inventory, 358 case entries including the Pair table, historical scheduler campaign | Implemented method; actual run metadata and saved results required |

Coverage never flows upward merely because a broader test passes. For example,
`libdoor` success cannot close a hostile raw-selector gap, and repeated stress
cannot replace a deterministic state barrier.

<a id="contents-a1-8-failure-classification-and-diagnostics"></a>

## A1.8. Failure classification and diagnostics

### A1.8.1. Classification

| Class | Evidence | Qualification result |
|---|---|---|
| ATF `failed` | Assertion message and expected/actual state | Failed; investigate implementation, specification, and test premise |
| Kyua `broken` | Empty/malformed result, harness error, or abrupt program exit | Not passed; classify underlying crash or harness failure |
| Timeout/hang | Kyua timeout or blocked process with no predicate progress | Failed; capture stacks and ownership before termination |
| Userspace crash | Signal, core, invalid memory or allocator failure | Failed unless the exact signal is the asserted result |
| Kernel panic | Trap, assertion, lock-order violation, corrupt scheduler state | Campaign invalid; preserve dump and exact build |
| Resource instability | Post-test count differs from defined baseline | Correctness failure even if assertions passed |
| Semantic false success | Success at the wrong completion point or ownership transfer | ABI failure; common in ONEWAY, MOVE, owner exit, and replacement |

### A1.8.2. Kyua evidence

```sh
kyua report --results-file='<result-id>'
kyua report-junit --results-file='<result-id>' > /var/tmp/door-failure.xml
kyua db-exec --results-file='<result-id>' 'SELECT * FROM test_results;'
```

Always name the result database. An unqualified `kyua report` may select a
later run and hide the original failure.

### A1.8.3. Blocked-process evidence

```sh
procstat -kk <pid>
procstat -t <pid>
procstat -f <pid>
ps -H -o pid,tid,state,wchan,command -p <pid>
```

Record every thread, not only the thread named in the assertion. A client in a
private scheduler wait, a server blocked on a pipe, and a coordinator waiting
on an umtx describe different ownership failures.

### A1.8.4. Kernel and resource evidence

Preserve `dmesg`, console output, crash dumps, scheduler identity, topology,
CPU placement, `vmstat -m` DoorFS allocations, `procstat` file observations,
and production Door/NameFS accounting interfaces. Temporary instrumentation
belongs to a separate reproducer build and cannot silently become release or
benchmark evidence.

<a id="contents-a1-9-verified-qualification-record"></a>

## A1.9. Verified qualification record

### A1.9.1. Actual check record

Record date, FreeBSD release/build, scheduler, CPU/affinity topology, kernel
configuration and instrumentation, ABI, installed component consistency,
commands, Kyua databases, and resource baselines. Continuous development does
not require a final repository revision or a source snapshot. A benchmark may
retain source identity for its particular measurement independently.

### A1.9.2. Reported results and scope

| Evidence | Result and limit |
|---|---|
| Current libdoor inventory | 47 programs, 349 registered cases; 365 with the 5 DoorFS and 11 NameFS cases |
| Synchronous forwarding, 8 November 2026 | 7/7 reported passed, including a twelve-relay chain and client/backend/relay process death; see [A1.3.18](test-catalog/18.md#forwarding-tests) |
| Pair extension evidence | Seven registered ATF cases; standalone model checks and 660 corrected M:N benchmark cases are separate evidence, not a complete 349-case Kyua run |
| Earlier complete libdoor run | 335/335 reported successful, including both new MOVE/revoke gates |
| Scheduler cleanup run, 21 October 2026 | Author reported successful tests after switch unification and unused-method removal; no separate case count, scheduler/configuration or Kyua database supplied with this report |
| Subsequent SHM view lock fix | Reported successful after correction; no separate full result database supplied |
| Earlier native and ELF32 runs | 333/333 each; not a separately reported ELF32 335-case run |
| Reported earlier ELF environment | FreeBSD 16.0 (1600027), amd64/i386 ELF builds; latest base/configuration not supplied |
| Earlier scheduler campaigns | 1000 full iterations under each scheduler for an earlier inventory; not qualification of every later case |
| Editorial verification | Current source/case inventory, internal links, Mermaid render; no new FreeBSD execution here |
| Latest scheduler identity | Not specified with the scheduler-cleanup test report |
| Performance | Appendix 2 records IPC and local application workloads separately from ATF qualification |

A passing suite can coexist with WITNESS warnings. The page-view attach
reversal was found in console output despite 335/335. Qualification requires
review of kernel diagnostics as well as Kyua results. SIGSEGV/SIGBUS from
the read-only persistent SHM child and the non-inherited page-view child are
intentional protection checks when their parent test verifies the signal;
they are distinct from kernel panic or lock-order diagnostics.

<a id="contents-a1-10-known-coverage-gaps"></a>

## A1.10. Known coverage gaps

The following gaps remain visible even when all current cases pass:

| Gap ID | Missing proof | Required addition |
|---|---|---|
| `GAP-RAW-001` | Exhaustive raw selector/version/reserved-field/pointer matrix beyond the added async and compatibility checks | DoorFS case family covering every selector class |
| `GAP-RAW-002` | Forced generic/empty/scalar CALL and RETURN equivalence | Raw plus libdoor forced-path cases |
| `GAP-CAP-001` | Every `CAP_DOOR_*` right at the raw syscall boundary | Raw Capsicum rights matrix without registry assistance |
| `GAP-COUNT-001` | Exhaustive live-resource baseline after every injected failure (cycle counters now cover IPC and NameFS) | Kernel accounting rollback case |
| `GAP-MOVE-001` | Deterministic fd-number reuse during MOVE acknowledgement/rollback | Barrier-controlled reuse case |
| `GAP-NMOUNT-001` | Complete malformed/conflicting NameFS mount-option matrix | Raw `nmount()` validation cases |
| `GAP-VNODE-001` | Concurrent open/rebind/detach versus inactive/reclaim/deferred GC | NameFS VFS stress with reference baseline |
| `GAP-DETACH-001` | Strong proof that post-detach lookup reaches the underlying non-Door object | Strengthen detach case with `fstat()` and CALL rejection |
| `GAP-AFF-001` | Focused proof that affinity mismatch never performs cross-CPU direct switch | Pinned scheduler regression under 4BSD and ULE |
| `GAP-UP-001` | UP compile/runtime fallback qualification | UP build plus core, signal, lifetime suites |
| `GAP-JAIL-001` | Cross-jail CALL permission does not confer ADMIN/INFO/SHM owner authority | Cross-jail operation-right denial matrix |
| `GAP-ABI-001` | Architectures beyond the asserted amd64/i386 LP64/ILP32 pair | Add architecture builds and runtime evidence |
| `GAP-EXAMPLE-001` | Fresh complete 26-example runtime qualification after the latest audit | Example build/run qualification record |
| `GAP-BENCH-001` | Complete production-equivalent performance qualification beyond the recorded configurations | Appendix 2 supplies measured baselines; retain full provenance, alternate/randomize methods, and qualify missing workload/environment cells |

Gaps describe missing proof, not necessarily known implementation defects.
They must not be silently closed by a broad stress pass. Closing a row requires
the focused test, its source link, and a recorded focused result.

---
