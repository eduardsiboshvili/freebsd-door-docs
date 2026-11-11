[Monograph index](../README.md) · [Full text](../door-ipc-implementation-for-freebsd_full.md)

<a id="appendix4-9"></a>

## A4.9. Local demo installation and application routes

The bundled Door Stack is a local development demo. Install the matching
Door-enabled FreeBSD kernel, headers and libdoor, then boot that kernel.
From the extracted stack, run as root on a development host or VM:

```sh
make install
sh ./demo.sh status
curl -fsS http://127.0.0.1:8080/health
```

`make install` invokes `install-demo.sh`, checks that a Door can be created,
builds a private nginx and dynamic module, builds the application and benchmark
binaries, and starts the demo. PostgreSQL 18 is provisioned in a private cluster
by default; use `make install DEMO_ARGS=--no-pgsql` for the SQLite-only demo
and IPC/HTTP benchmarks. The generated configuration is
`conf/demo-application.toml` and `conf/demo-nginx.conf`. The installer keeps
its token, SQLite state, uploads and benchmark artifacts separate from the
manual stack. It prints the administrative token on an interactive terminal;
the default file is `state/demo-admin-token`. Keep the demo on loopback or a
trusted local network.

| Route | Application responsibility |
|---|---|
| `/`, `/filemanager/` | Application index and file manager; the app owns routing and the configured file root. |
| `/blog/` | Choose the default or a named SQLite database, or the configured PostgreSQL database. The choice is carried by the URL; each database keeps separate data and sessions. The admin token can create a named SQLite database, optionally with sample posts. |
| `/benchmark/` | HTTP workloads, Direct ping and SHM test controls and results. |
| `/ipcbench/` | IPC matrix, individual and extended plans, CPU telemetry, export and cleanup. |
| `/pgbench/` | Prepare dedicated SQL data, apply the worker profile, run transport comparisons, and download results. |
| `/health` | Returns `ok` from the application through nginx. |

An alternative to the blog's creation form is
`sh ./demo.sh sqlite demo-notes --sample`. Creating a named database does
not overwrite an existing name. In the default PostgreSQL demo the blog
uses `door_blog` over libpq; the SQL benchmark uses `door_pgbench` and its
own non-superuser role. The preload extension has one worker pool per
cluster, bound to the configured database/role. A benchmark profile would
displace a PGDoor blog profile in the same cluster, which is why the
managed demo keeps the blog on libpq. An external deployment can use a
separate cluster or select its own single pool profile.

Initial provisioning and the UI's **Create/refill the test database**
and **Apply the PGDoor profile to PostgreSQL** apply the profile, restart
only the installer-managed cluster, and probe the Door workers before
reporting readiness. Active connections to that cluster may be
interrupted. An ordinary benchmark job does not restart PostgreSQL;
a custom installation must restart its own cluster after applying settings
that show `pending_restart`. The generated profile's actual endpoint
prefix and worker count are authoritative.

After installation or reboot use `sh ./demo.sh start`; the supervisor
starts the app and its nginx together. Use `stop`, `restart`, `status`, or
`doctor` through the same script. A successful health probe confirms
that the application answered; it does not prove that a benchmark can
create a result directory or open every PGDoor endpoint. If a request
fails, check the exact listener and response body, then
`state/application.log` and `state/demo-nginx/nginx-error.log`. If the API
accepted the job, inspect `job-*/output.log` below the configured
`[benchmarks].results` path (default `state/demo-results`). A system nginx
on port 80 and the demo nginx on port 8080 are different listeners.

<a id="appendix4-10"></a>

## A4.10. Benchmark control, reproducibility and validation

| Suite | What is measured | Useful controls |
|---|---|---|
| Direct ping | Separate no-data and payload cases: synchronous ping-pong, ONEWAY with inline or SHM data, and native async cases supported by the runner. | Payload, clients, servers, CPU placement, in-flight window and warmup. Keep ONEWAY submission distinct from verified completion. |
| IPC matrix | Unix stream/datagram/seqpacket, loopback TCP/UDP, pipe and SHM/kqueue baselines, plus Door CALL, ONEWAY, async and SHM variants. | Single transport or matrix; report the effective worker topology and window, not just requested values. |
| SHM | Ring + batched CALL; Ring + Pair; Pool + Pair reserve/release; Pool reuse + Pair; SHM CALL/async; fixed-stream M:N Ring/Pool reuse + Pair; POSIX SHM + process-shared umtx/futex baseline. | Payload, flow, profile, slots, window, client and worker counts and CPU lists. M:N Pair and the futex baseline use fixed-stream payloads; unsupported selections are disabled. |
| SQL | `door`, `door-direct` and libpq over a Unix socket. SQL ping-pong is `SELECT 'pong'::text`; other cases are SELECT 1, text up to 8 MiB, 21 rows × 128 bytes, and a deterministic feed. | Run one scenario or the suite; match client and endpoint counts where possible. Identity and result-equivalence probes are outside timing. |
| HTTP | Closed-loop HTTP RAM capacity, or scheduled-rate HTTP blog traffic with PostgreSQL. | Match payload, generator threads, connections and benchmark duration. For the SQL-backed HTTP test, also match the account set, depth, mix and offered rate. |

The finite full plan has no overall deadline. The continuous plan repeats
until stopped. The monograph plan expands valid combinations of payloads,
windows, supported profiles and client/worker layouts; a selected transport
can be run alone with the same plan. Report p50/p95/p99/p99.9 and the timing
boundary, alongside throughput. Do not use aggregate ns/op as RTT. On
FreeBSD the UI samples logical CPU counters from `kern.cp_times`, including
other host activity. The first sample sets the baseline.

A download during a run is a checkpoint. The retained raw-record list can
roll over at 3,000 entries; `results.jsonl` on disk preserves the full run,
and completed finite monograph plans write `monograph.md` with all
aggregates. Check the current case and passed/failed/skipped counts before
treating a completed runner as a verified series. Job cleanup is disabled
while a managed run is active; it removes only benchmark results under the
manager's ownership rules.

`make check` exercises the router, SQLite database selection and creation,
the nginx notification model, the stack and demo lifecycle, IPC/futex and
benchmark job management, fixture creation and PGDoor wire handling. Use
native FreeBSD checks in addition to the portable suite:

```sh
make -C benchmarks/door-direct-ping check
make -C benchmarks/shm-transport-bench check-mt
make -C benchmarks/shm-transport-bench check-futex
python3 app/pgsql/smoke.py http://127.0.0.1:8080
```

The blog smoke command assumes the demo is running on its default port
with a configured blog database. Test doubles establish runner behavior;
they do not exercise native kernel Door delivery or PostgreSQL worker
execution. For the latest installation the reported `demo.sh status` and
HTTP 200 `/health` confirm service readiness on FreeBSD. That observation
and the separately reported benchmark runs are narrower than a complete
native integration matrix.
