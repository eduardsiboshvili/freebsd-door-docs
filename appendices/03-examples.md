[Monograph index](../README.md) · [Full text](../door-ipc-implementation-for-freebsd_full.md)

<a id="appendix-3"></a>

<a id="contents-appendix-3-example-programs-and-application-patterns"></a>

# Appendix 3. Example programs and application patterns

The example tree is part of the implementation deliverable. It demonstrates
how the public ABI, libdoor runtime, NameFS, DoorCTL, descriptor transport,
persistent SHM, rings, security controls, and lifecycle rules compose into
complete applications. Examples are explanatory integration programs, not a
replacement for Appendix 1: a successful demonstration does not prove hostile
raw-ABI validation or close a race-coverage gap.

Every example has its own README with exact build, server, client, helper, and
automatic-run commands. Every executable validates malformed command lines
through `usage()` and reports runtime failures with the failed operation and
`errno`.

<a id="contents-a3-1-reading-and-execution-rules"></a>

## A3.1. Reading and execution rules

The examples are cumulative:

- 01–06 establish synchronous calls, descriptors, ONEWAY, SHM, rings, and
  credentials;
- 07–11 compose asynchronous data paths with explicit completion;
- 12–15 cover worker pools, limits, replacement, and shutdown;
- 16–23 cover reciprocal transports, isolated sessions, capability pipelines,
  security domains, signals, process transitions, raw ABI, and adaptive
  transport selection.

Build or run the complete set from `freebsd-doors-guide/examples`:

```sh
make
make run
```

An individual directory is independently buildable. Automatic runs use unique
temporary paths and names, start the required server processes, run the client
matrix, request orderly shutdown, and remove their publications. A README may
also provide interactive commands for inspecting intermediate state.

All 26 examples are implemented. Earlier runs and the latest source audit
have different scope; see A3.7 for the remaining runtime check.

<a id="contents-a3-2-master-example-matrix"></a>

## A3.2. Master example matrix

| No. | Example | Status | Endpoint architecture | Main contract |
|---:|---|---|---|---|
| 01 | `01-echo` | Implemented; run verified | One synchronous Door | A successful `door_call()` includes reply delivery. |
| 02 | `02-descriptor-transfer` | Implemented; run verified | One synchronous Door | COPY and MOVE transfer capabilities transactionally. |
| 03 | `03-oneway` | Implemented; run verified | One ONEWAY Door | Submission is kernel acceptance, not callback completion. |
| 04 | `04-persistent-shm` | Implemented; run verified | One synchronous Door | A checked persistent-SHM reference replaces copied payload data. |
| 05 | `05-shm-ring` | Implemented; run verified | Synchronous control/notification over one Door | Ring slots have explicit reserve, publish, acquire, and release ownership. |
| 06 | `06-client-credentials` | Implemented; run verified | One synchronous Door | Authorization uses kernel-captured credentials. |
| 07 | `07-oneway-shm` | Implemented; run verified | ONEWAY data Door plus synchronous control Door | Asynchronous SHM needs readiness and completion barriers. |
| 08 | `08-oneway-fd` | Implemented; run verified | Synchronous capability Door plus ONEWAY data Door | Transfer an fd before publishing asynchronous work that refers to it. |
| 09 | `09-doorctl-matrix` | Implemented; run verified | Named synchronous and ONEWAY Doors | DoorCTL discovers capabilities; the Door remains the transport. |
| 10 | `10-mixed-service` | Implemented; run verified | One ordinary Door accepting synchronous and no-reply operations | Application opcode selects semantics while one worker pool handles both classes. |
| 11 | `11-oneway-completion` | Implemented; run verified | ONEWAY data Door plus synchronous acknowledgement Door | Explicit completion, flow control, and safe input reuse. |
| 12 | `12-concurrent-pools` | Implemented; run verified | Two protocol-compatible servers with different worker models and a shared client set | Compare legacy/shared workers with private `door_xcreate()` pools under identical loads. |
| 13 | `13-limits-and-introspection` | Implemented; run verified | One configurable synchronous Door | Limits, endpoint information, attributes, and expected rejection paths. |
| 14 | `14-generation-recovery` | Implemented; run verified | DoorCTL identities plus stable NameFS publication | Reconnect to a replacement generation without retargeting old capabilities. |
| 15 | `15-graceful-shutdown` | Implemented; run verified | Mixed CALL, ONEWAY, SHM, and ring service | Stop admission, drain work, detach resources, revoke, and destroy state. |
| 16 | `16-duplex-rings` | Implemented; run verified | Reciprocal client-owned and server-owned Doors, two generation-bound SPSC rings, and ONEWAY notifications | Build a bidirectional record channel without violating owner-only SHM attachment or pretending one SPSC ring is duplex. |
| 17 | `17-multiclient-sessions` | Implemented; run verified | Bootstrap Door plus per-client control/data capabilities, callback Door, and reciprocal SHM state | Isolate clients, mappings, limits, completion counters, and teardown. |
| 18 | `18-capability-pipeline` | Implemented; run verified | Synchronous fd/session setup plus ONEWAY ring work | Compose descriptor capabilities, SHM records, generations, and asynchronous execution. |
| 19 | `19-security-boundaries` | Implemented; run verified | Capsicum client and optional cross-jail named service | Demonstrate rights reduction, credential checks, local/global discovery, and denied operations. |
| 20 | `20-signals-recursion` | Implemented; run verified | Interruptible and non-cancellable Doors plus recursive service calls | Handle `EINTR`, cancellation, recursion, and retry without duplicating side effects. |
| 21 | `21-unref-fork-exec` | Implemented; run verified | Referenced/unreferenced endpoint plus inherited and transferred capabilities | Explain `DOOR_UNREF`, process-local callback registry, fork/exec, and received fds. |
| 22 | `22-raw-abi` | Split peers; target rerun pending | Direct `SYS_door` operations; libdoor publication helpers | Show kernel validation and the minimum work a language runtime must implement. |
| 23 | `23-adaptive-transport` | Implemented; run verified | Control, synchronous, ONEWAY, persistent-SHM, and ring paths | Select transport by semantics and payload lifetime rather than size alone. |
| 24 | `24-async-call` | Implemented; earlier run reported; audit rerun required | Ordinary server Door plus client-owned private completion Door | Kernel async CALL with copied payload, application SHM references and SPSC ring records. |
| 25 | `25-shm-page-view` | Implemented; namefs run reported; audit rerun required | Namefs request Door; composed read-only page view | Publish/attach/release/destroy, contiguous VA from disjoint source pages, release before reuse. |
| 26 | `26-shm-pair` | Standalone peers; native rerun pending | Named ordinary Door; independent producer and consumer; one active session | Synchronous REGISTER/DETACH; single SHM slot or batched SPSC ring with Pair notifications |

Historical run labels describe earlier checks. All revised examples need a
fresh top-level run; the current source audit is not runtime qualification.

<a id="contents-a3-3-foundational-examples"></a>

## A3.3. Foundational examples

### A3.3.1. `01-echo`

**Purpose:** Establish the smallest complete server/client lifecycle.

**Topology and sequence:** The server creates an ordinary Door, publishes it at
a unique NameFS path, and waits in its managed worker. The client opens that
path, submits one copied string, receives the transformed reply, and closes its
capability. Shutdown detaches publication, revokes the endpoint, and releases
the owner descriptor.

**Contracts demonstrated:** A successful `door_call()` includes callback and
reply delivery; request and reply are byte ranges rather than implicit C
strings; callback state outlives every active callback; NameFS publication and
the Door capability have separate references.

**Related tests:** `basic:echo_call`, `basic:multiple_calls`, API close/revoke,
and NameFS basic publication.

### A3.3.2. `02-descriptor-transfer`

**Purpose:** Demonstrate capability transfer in both directions and persistent
server-side slots.

**Topology and sequence:** Clients submit temporary, retained-COPY, or
retained-MOVE file descriptors. The server reads a requested range, may retain
the received capability in a numbered slot, and can later export a slot by
COPY or MOVE. Separate prepared files make slot identity observable.

**Contracts demonstrated:** COPY preserves the sender capability; MOVE closes
the matching source only after receiver installation commits; an application
slot has its own generation and ownership; uncertain MOVE must not be blindly
retried; every returned descriptor is closed or retained explicitly.

**Related tests:** descriptor validation, graph, limits, ownership, types,
mixed-payload, and fault-injection suites.

### A3.3.3. `03-oneway`

**Purpose:** Isolate true asynchronous submission from synchronous RPC.

**Topology and sequence:** A producer submits a copied message to a
`DOOR_ONEWAY` endpoint. The client reports kernel acceptance while the server
independently reports callback processing.

**Contracts demonstrated:** success is not callback completion; there is no
reply channel or descriptor payload; copied bytes may be reused after submit,
but external application state still needs its own completion protocol.

**Related tests:** all ONEWAY snapshot, blocking, revoke, burst, and completion
distinction cases.

### A3.3.4. `04-persistent-shm`

**Purpose:** Show a mapped region whose bytes are not recopied through the
ordinary call payload.

**Topology and sequence:** The client creates and maps a generation-bound SHM
region, writes data, constructs a checked range, and invokes the server. The
owner attaches a read-only consumer view, validates the logical range, reads
it, acknowledges completion, detaches, and permits destroy/unmap.

**Contracts demonstrated:** addresses are process-local; offset and length are
validated; a numeric region id is meaningful only with its Door generation;
handle destruction, mapping lifetime, and active-call references are distinct.

**Related tests:** shared-memory and persistent-SHM suites, including bounds,
permissions, active references, and destroy-versus-mapping lifetime.

### A3.3.5. `05-shm-ring`

**Purpose:** Demonstrate the ten-function checked SPSC ring API.

**Topology and sequence:** One producer reserves and commits records; one
consumer acquires immutable views and consumes them. Door notification/control
coordinates readiness and final drain.

**Contracts demonstrated:** one producer and one consumer; exact writer/view
identity; release/acquire publication; ring-full backpressure; abort of an
unpublished reservation; committed length hides reused slot tails; mapping
teardown follows outstanding state resolution.

**Related tests:** all fourteen `shm_ring` cases.

### A3.3.6. `06-client-credentials`

**Purpose:** Authorize from credentials captured by the kernel rather than
identity fields supplied in payload data.

**Topology and sequence:** A client invokes an ordinary Door; the callback
queries the current caller and applies a simple policy before returning the
observed identity and decision.

**Contracts demonstrated:** credentials are callback-local snapshots; PID,
UID, GID, groups, and jail information are policy inputs, not proof supplied by
the client; `DOOR_NO_CRED` changes availability explicitly.

**Related tests:** basic credentials, attributes, Capsicum, access statistics,
and NameFS jail cases.

<a id="contents-a3-4-asynchronous-composition-and-service-operation"></a>

## A3.4. Asynchronous composition and service operation

### A3.4.1. `07-oneway-shm`

**Purpose:** Make SHM-backed ONEWAY lifetime safe.

**Topology and sequence:** A synchronous control Door establishes readiness and
reports completion; an ONEWAY Door accepts strong SHM references. The producer
waits for the acknowledged sequence before reusing or unmapping storage.

**Contracts demonstrated:** submit does not mean the callback has read the
mapping; active calls retain the region object; revoke and drain are separate;
generation replacement invalidates old strong references.

**Related tests:** persistent-SHM ONEWAY and DoorCTL ONEWAY-SHM cases.

### A3.4.2. `08-oneway-fd`

**Purpose:** Combine descriptor capabilities with a data-only ONEWAY path.

**Topology and sequence:** A synchronous control transaction installs an fd in
a server slot and returns its generation. Later ONEWAY messages refer to that
slot/generation without carrying descriptors. A control operation observes
completion or removes the slot.

**Contracts demonstrated:** capability installation commits before work is
published; stale slot generations are rejected; MOVE uncertainty remains on
the synchronous control plane; ONEWAY remains descriptor-free.

**Related tests:** descriptor ownership/limits, mixed payload, ONEWAY, and
generation replacement.

### A3.4.3. `09-doorctl-matrix`

**Purpose:** Demonstrate logical-name discovery independently from transport.

**Topology and sequence:** The server registers ordinary and ONEWAY generations
with DoorCTL. Clients LOOKUP or CONNECT and then exercise copied CALL, copied
ONEWAY, synchronous SHM, ONEWAY SHM, credentials, and completion control.

**Contracts demonstrated:** a name resolves an exact generation; CONNECT
returns an ordinary capability; unregister hides future discovery but does not
revoke existing fds; transport and authorization remain Door operations.

**Related tests:** DoorCTL, DoorCTL ONEWAY, DoorCTL SHM, and DoorCTL ONEWAY-SHM.

### A3.4.4. `10-mixed-service`

**Purpose:** Show how application semantics can share one ordinary endpoint.

**Topology and sequence:** One managed pool dispatches typed QUERY,
UPDATE_SYNC, application no-reply, and STATS opcodes. Every invocation is still
an ordinary synchronous Door transaction; selected operations merely return an
empty application result.

**Contracts demonstrated:** application no-reply is not true ONEWAY; short
operations may share a pool; long asynchronous work should move to an explicit
queue or separate ONEWAY endpoint; opcode and reserved-field validation precede
state mutation.

**Related tests:** basic, API, attributes `DOOR_NO_RETURN`, mixed payload, and
pool lifecycle.

### A3.4.5. `11-oneway-completion`

**Purpose:** Unify completion across copied, persistent-SHM, and ring ONEWAY
submissions.

**Topology and sequence:** A data endpoint accepts all three transport forms;
a synchronous control Door negotiates resources and exposes a monotonically
increasing accepted/completed watermark. Producers bound in-flight work and
wait for the target sequence before reuse.

**Contracts demonstrated:** completion belongs to the application protocol;
ring-full and in-flight limits provide backpressure; acknowledgement may batch
several operations; shutdown waits for the final accepted sequence.

**Related tests:** ONEWAY, persistent SHM, ring, revoke-active, and resource
limit suites.

### A3.4.6. `12-concurrent-pools`

**Purpose:** Compare two server-thread architectures under the same protocol
and client triggers.

**Topology and sequence:** The shared server uses `door_create()` and the
legacy process-wide pool with depletion notification. The private server uses
`door_xcreate()`, setup hooks, and endpoint-specific pools. Clients run ping,
parallel, deterministic block/release, ONEWAY, mixed, overload, and disconnect
modes and compare exact counters.

**Contracts demonstrated:** the callback is one-request dispatch, not a
listening loop; the initial worker is ready before create succeeds; depletion
may add workers; private pools isolate latency and throughput roles; bounded
overload must backpressure or reject rather than deadlock; detached workers do
not transfer callback-state lifetime to libdoor.

**Related tests:** depletion, legacy pool, xcreate, pool lifecycle, worker
hardening, concurrency stress, and final stress.

### A3.4.7. `13-limits-and-introspection`

**Purpose:** Make utility APIs and expected policy failures observable.

**Topology and sequence:** The server and client inspect `door_info()`, query
and set endpoint parameters, exercise attributes, run exact-boundary requests,
and deliberately exceed data, descriptor, SHM, and ring limits by one unit.

**Contracts demonstrated:** INFO is a snapshot; attributes are normalized;
expected rejection is not a crash; failed MOVE/data operations preserve
ownership; per-object limits are distinct from administrator-controlled global
limits.

**Related tests:** ABI, API, attributes, descriptor limits, huge payload,
resource limits, and ring bounds.

### A3.4.8. `14-generation-recovery`

**Purpose:** Demonstrate stable discovery across server replacement without
confusing capability generations.

**Topology and sequence:** Generation one is registered with DoorCTL and
published through NameFS. The owner is revoked or exits; a replacement Door is
atomically registered and rebound to the stable pathname. The client compares
uniquifiers, observes failure on the old fd, reconnects, and rebuilds
generation-bound state.

**Contracts demonstrated:** an old descriptor never silently addresses the
replacement; equal numeric SHM ids across generations are unrelated; recovery
is detect, discard old resources, reconnect, renegotiate, and republish; the
stable vnode is a rendezvous identity, not a mutable fd alias.

**Related tests:** all DoorCTL replacement families, lifetime owner-exit, and
NameFS recovery/rebind cases.

### A3.4.9. `15-graceful-shutdown`

**Purpose:** Compose the full deterministic service teardown protocol.

**Topology and sequence:** The server stops admission, removes DoorCTL/NameFS
discovery, stops producers, drains accepted ONEWAY work and active callbacks,
detaches consumer mappings, destroys handles, revokes endpoints, closes
capabilities, and destroys synchronization state last.

**Contracts demonstrated:** close, unregister, detach, revoke, drain, and join
are not synonyms; `door_return()` may finish a reply and then leave its fused
wait with `EBADF`/`ENXIO`; application state remains alive until callback and
completion counters reach their predicates.

**Related tests:** lifetime, lifetime_extended, pool lifecycle, worker
hardening, ONEWAY revoke, persistent-SHM revoke, and final stress.

<a id="contents-a3-5-advanced-compositions"></a>

## A3.5. Advanced compositions

### A3.5.1. `16-duplex-rings`

**Purpose:** Build full-duplex records from two correctly owned SPSC channels.

**Topology and sequence:** The server owns a request Door and consumes a
client-produced request ring. The client owns a callback Door and consumes a
server-produced response ring. A synchronous bootstrap transfers the callback
capability and negotiates both generation-bound regions before either producer
publishes data.

**Contracts demonstrated:** one SPSC ring is not duplex; consumer attachment is
owner-controlled; each direction has an independent sequence and completion
watermark; half-close and teardown proceed per direction; equal region ids do
not cross Door generations.

**Related tests:** persistent SHM, rings, descriptor transfer, recursive calls,
and generation rejection.

### A3.5.2. `17-multiclient-sessions`

**Purpose:** Prevent one client from becoming authority over another client's
state.

**Topology and sequence:** A bootstrap Door authenticates clients and creates
session-specific capabilities, callback Door relations, mappings, quotas,
sequence counters, and cleanup state. Several client processes operate
concurrently and disconnect independently.

**Contracts demonstrated:** session identity is not a reusable global integer;
generation and credentials qualify lookup; per-client quota and completion
state are isolated; client death and server shutdown release only references
owned by that session.

**Related tests:** concurrent clients, credentials, access stats, resource
limits, lifetime, and DoorCTL generation cases.

### A3.5.3. `18-capability-pipeline`

**Purpose:** Compose synchronous authority establishment with high-rate
asynchronous work.

**Topology and sequence:** A control CALL installs or replaces an fd and
returns `(session, slot, generation)`. Ring/SHM work records carry that tuple;
an ONEWAY notification schedules processing; a watermark reports completion.

**Contracts demonstrated:** queued work cannot use a replaced slot; capability
MOVE commits before publication; ring data is untrusted and bounds-checked;
completion governs slot/record reuse.

**Related tests:** descriptor transaction suites, SHM/ring corruption tests,
ONEWAY completion, and generation replacement.

### A3.5.4. `19-security-boundaries`

**Purpose:** Separate discovery, descriptor possession, operation rights, and
server authorization.

**Topology and sequence:** The client acquires required capabilities, limits
them with Capsicum, enters capability mode, and demonstrates allowed and
denied operations. An optional privileged setup contrasts jail-local and
global DoorCTL names and NameFS paths projected through NullFS.

**Contracts demonstrated:** the server trusts kernel credentials, not payload
identity; a visible pathname is not sufficient authority; global discovery
does not imply cross-jail CALL/ADMIN/SHM permission; transferred descriptors
retain only their rights.

**Related tests:** Capsicum, access stats, credentials, DoorCTL validation, and
NameFS host/jail and NullFS composition cases.

### A3.5.5. `20-signals-recursion`

**Purpose:** Make interruption and nested invocation behavior deterministic.

**Topology and sequence:** One operation is interrupted and reconciled using an
idempotent operation id; a `DOOR_NO_CANCEL` endpoint demonstrates continued
server work; bounded recursive calls use enough independently runnable workers
and avoid holding application locks across the nested CALL.

**Contracts demonstrated:** `EINTR` does not prove that the server did nothing;
non-idempotent retry needs reconciliation; signals cannot be lost before
private park; self-recursion with an exhausted pool deadlocks and is therefore
bounded or rejected.

**Related tests:** signals, cancellation, recursive, process-control, depletion,
and worker hardening.

### A3.5.6. `21-unref-fork-exec`

The example demonstrates inherited capabilities, process-local callback
registry behavior, UNREF, and Door descriptor COPY. A child-owned NameFS
relay receives a copied service fd, invokes the service, closes the received
capability, and replies. The bootstrap CALL reply confirms relay completion.
The child closes its inherited original service fd before the transfer;
UNREF is verified with the service's explicit completion barrier.

### A3.5.7. `22-raw-abi`

**Purpose:** Expose the work normally hidden by libdoor and prove that the
kernel validates untrusted callers independently.

**Topology and sequence:** A compact runtime invokes `SYS_door` selectors
directly, creates an endpoint, binds a worker, executes the fused return/wait
loop, performs CALL/INFO basics, and sends malformed versions, flags, reserved
fields, pointers, or sizes through selected negative paths.

**Contracts demonstrated:** syscall records are versioned; all user pointers
and counts are hostile; the runtime owns buffer growth, descriptor conversion,
TLS-like current-call state, and cleanup; fast selectors cannot change the
generic result.

**Related tests:** ABI/API/fault-injection suites and raw DoorFS tests. The
example is educational and is not complete raw-selector coverage.

### A3.5.8. `23-adaptive-transport`

**Purpose:** Select transport from semantics rather than advertise one path as
universally fastest.

**Topology and sequence:** One application operation is exercised through
copied synchronous CALL, synchronous persistent SHM, copied ONEWAY, ONEWAY SHM,
and bounded ring records. Negotiated limits, reply need, mutability, repetition,
and completion lifetime select the path; every mode produces the same logical
result or completion record.

**Contracts demonstrated:** small synchronous values favor copied CALL; stable
large ranges can use persistent SHM; repeated bounded records fit rings;
asynchronous paths require explicit completion; size is considered only after
ownership and completion are fixed.

**Related tests:** basic/mixed/huge payload, ONEWAY, persistent SHM, rings,
limits, replacement, and stress suites.

### A3.5.9. `24-async-call`

An ordinary request Door serves sync and native async clients. The async
client registers pending IDs and receives correlated completions on its
private ONEWAY Door. The example exercises copied payload sizes 0, 1, 64,
4096, and 65536; SHM bounds; ring wraparound; and 16 outstanding async calls.
Copied commands containing SHM/ring references use an application ownership
protocol, not kernel async CALL_SHM. Submit acceptance permits no early reuse
of referenced mutable storage. Stop admission and drain callback state before
teardown (§10.7, §14.14).

### A3.5.10. `25-shm-page-view`

The server publishes a local NameFS request Door; the client opens the anchor
file and consumes a page-composite view. The producer loans A page 3, B page 1,
and A page 1, preserving order without copying payload. Three rounds verify
release and reuse. The receiver stops readers and releases before RETURN; the
producer observes RELEASED and destroys before recycling. The read-only view
does not remove the producer's writable alias (§10.8, §14.15).

No example uses a socket. Example 21 transfers a capability with Door COPY
through a child-owned relay obtained by descriptor COPY over a parent-owned bootstrap Door.
Every example has a local `run.sh` and `make run`. NameFS anchors are created
if absent and existing regular files retain contents and modes; publication
and detach do not require deleting a pre-existing anchor.

<a id="contents-a3-6-cross-example-coverage-matrices"></a>

## A3.6. Cross-example coverage matrices

### A3.6.1. Transport coverage

Legend: **P** is the primary teaching responsibility; **S** is supporting use.

| Example | Sync CALL | App no-reply | True ONEWAY | Persistent SHM | Ring | Descriptor | Completion control | Native async | Page view | Pair notifications |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:| :---: |
| 01 | **P** | — | — | — | — | — | — | — | — | — |
| 02 | **S** | — | — | — | — | **P** | — | — | — | — |
| 03 | — | — | **P** | — | — | — | — | — | — | — |
| 04 | **S** | — | — | **P** | — | — | **S** | — | — | — |
| 05 | **S** | — | — | **S** | **P** | — | **S** | — | — | — |
| 06 | **P** | — | — | — | — | — | — | — | — | — |
| 07 | — | — | **P** | **P** | — | — | **P** | — | — | — |
| 08 | **S** | — | **P** | — | — | **P** | **P** | — | — | — |
| 09 | **P** | — | **P** | **P** | — | — | **P** | — | — | — |
| 10 | **P** | **P** | comparison | optional | — | — | internal | — | — | — |
| 11 | **S** | comparison | **P** | **P** | **P** | — | **P** | — | — | — |
| 12 | **P** | **P** | **P** | optional | optional | — | **P** | — | — | — |
| 13 | **P** | **P** | **P** | limits | limits | limits | — | — | — | — |
| 14 | **P** | — | **P** | **P** | optional | optional | **P** | — | — | — |
| 15 | **P** | **P** | **P** | **P** | **P** | optional | **P** | — | — | — |
| 16 | **S** | — | **P** | **S** | **P** | bootstrap | **P** | — | — | — |
| 17 | **P** | — | **P** | **P** | **P** | optional | **P** | — | — | — |
| 18 | **S** | — | **P** | **P** | **P** | **P** | **P** | — | — | — |
| 19 | **P** | — | **P** | optional | optional | setup | **S** | — | — | — |
| 20 | **P** | optional | — | — | — | optional | reconciliation | — | — | — |
| 21 | **P** | — | — | — | — | transfer | lifecycle barriers | — | — | — |
| 22 | raw | raw | raw | — | validated records | raw | raw worker loop | — | — | — |
| 23 | **P** | comparison | **P** | **P** | **P** | qualified handles | **P** | — | — | — |
| 24 | **S** | — | — | **S** | **S** | — | **P** | **P** | — | — |
| 25 | **S** | — | — | source pages | — | — | release acknowledgement | — | **P** | — |
| [26](#example-26-door-bootstrap) | **S** | — | — | **P** | **P** | — | **S** (REGISTER/DETACH) | — | — | **P** |

Example 26 uses DoorCTL discovery. Pair lanes signal publication and
completion in both modes; the copied control CALLs only register and
detach the mappings.

### A3.6.2. Concurrency and lifecycle coverage

| Concern | Primary examples |
|---|---|
| Initial worker readiness and ordinary managed callback | 01, 10 |
| Shared pool, depletion, and private xcreate pools | 12 |
| ONEWAY acceptance and explicit completion | 03, 07, 08, 11 |
| Multiple clients and per-session isolation | 12, 17 |
| Signals, cancellation, and recursion | 20 |
| Fork, exec, transfer, and UNREF | 21 |
| Generation replacement and reconnect | 14 |
| Full graceful shutdown and drain | 15 |
| Raw fused RETURN/wait-next lifecycle | 22 |
| Native submit/callback lifetime, completion accounting, and drain | 24 |
| Page leases, read-only mapping, release before reuse | 25 |
| Independent named peers; one active session, with readmission after detach | 26 |
| Pair event snapshot and recheck; record or batch completion before reuse | 26 |
| Worker final access and server unmap before the client destroys its handles | 26 |

### A3.6.3. Security and ownership coverage

| Concern | Primary examples |
|---|---|
| Kernel credentials | 06, 19 |
| Capsicum rights | 19 |
| Jail scope and NullFS-visible NameFS publication | 19 |
| Descriptor COPY/MOVE | 02, 08, 18 |
| Generation-qualified SHM | 04, 07, 09, 14, 16–18, 23–26 |
| Ring ownership and corruption-resistant API | 05, 11, 16–18, 23, 24, 26 |
| Same-user Pair session, Door-owner attach, and PID-bound detach | 26 |
| Raw untrusted ABI | 22 |
| Limits and denial-of-service boundaries | 12, 13, 17, 19 |

<a id="contents-a3-7-example-qualification-record"></a>

## A3.7. Example qualification record

An example qualification run records:

| Field | Required evidence |
|---|---|
| Source identity | Date, FreeBSD build, ABI, and source identity where useful |
| Installed identity | Matching kernel, world, headers, libdoor, and examples |
| Build | Top-level `make clean all` result |
| Automatic execution | Top-level `make run` result and captured output |
| Manual-only modes | Commands executed or explicit reason they are not automatic |
| Environment | Scheduler, CPU topology, jail/Capsicum setup where used |
| Cleanup | No stale NameFS mounts, DoorCTL names, processes, mappings, or temporary paths |
| Result | Per-example pass/fail; a later success does not erase an earlier failure |

Examples 24 and 25 have earlier successful reported runs; example 25 was
also checked as ELF32 and ELF64. The latest full example audit verified source
syntax, script syntax, local runners, and absence of socket APIs. It requires
a new complete FreeBSD run, particularly the revised example 21 relay and
the standalone example 26 peers using DoorCTL.
Appendix 1 remains the correctness record; Appendix 2 contains performance
methodology and the reported measurements.

<a id="pair-examples"></a>
<a id="example-26-door-bootstrap"></a>

## A3.8. Example 26: Pair client and server

The maintained `examples/26-shm-pair` example builds two independently
launched programs: `pair_demo_server` and `pair_demo_client`. The server
publishes an ordinary Door with `door_named_register()`. The client connects
by service name with `door_named_connect()`, creates the Pair region and,
in ring mode, a separate SPSC ring. The peers run under the same user in the
same jail.

| Stage | What the example demonstrates |
|---|---|
| REGISTER | A synchronous CALL carries region IDs. The server attaches as the Door owner, validates the mapping layout, and binds the session to the calling PID before replying. |
| `shm` mode | The producer writes one 64-byte slot and waits for the consumer to verify it before reuse. |
| `ring` mode | The producer reserves and commits eight records per batch. The consumer acquires, verifies, and consumes all eight before acknowledging the batch. |
| Pair notification | Lane 0 signals publication; lane 1 signals completion. Each waiter snapshots its event sequence before checking the acquire-loaded predicate. The counters determine when bytes may be read or reused. |
| DETACH | After the last completion, a synchronous CALL waits for the worker's final access, unmaps the server regions, and returns the verified count. The client then destroys its handles. |

CALL carries setup and teardown messages only. Payload bytes stay in SHM or
the ring. The server accepts one active session and can accept another after
a successful detach. Restart it to clear a session abandoned by its client.

On FreeBSD with matching kernel, headers, and libdoor, run both modes with:

```sh
make -C examples/26-shm-pair run
```

The runner starts one server and two successive clients; each must verify
1,024 records. See §14.22 for separate launches. The two modes and invalid
control requests were checked in a local model; the native FreeBSD run of
the standalone peers is still pending.

---
