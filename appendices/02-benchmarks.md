[Monograph index](../README.md) · [Full text](../door-ipc-implementation-for-freebsd_full.md)

<a id="appendix-2"></a>

<a id="contents-appendix-2-benchmark-methodology-and-results"></a>

# Appendix 2. Benchmark methodology and results

## A2.1. Measurement scope and evidence

The recorded platform is FreeBSD 16.0-CURRENT on an Intel Core i5-10400 at 2.90 GHz, with six physical cores and twelve logical CPUs. Results characterize this processor and the measured implementations; they are not a scheduler-specific performance claim. The corrected Door JSON identifies GENERIC-NODEBUG. Its `invariants` field is null, so that field alone does not establish the individual kernel option state. No measurements were executed by the document editor.

There are two IPC evidence sets. The earlier complete matrix reports 7335 successful cases, no failures/skips, 1467 aggregate configurations and five repetitions each. It includes sockets, pipes, Door invocation, single-channel SHM, M:N Pair and direct ping. The later corrected Door M:N runs each contain 330 verified cases and 66 five-repeat aggregates; the SHM + futex/umtx summary supplies another 330 successful cases and 66 aggregates. The 132 old M:N configurations are superseded, not averaged with the correction. The retained tables therefore contain 1335 original non-M:N-Pair configurations plus 198 current SHM configurations: **1533 aggregate rows**. This is not a count of independent correctness tests.

The earlier intermediate global JSON was a running checkpoint with a 3000-record retention limit; it is not the final campaign record. The final earlier Markdown supplies every aggregate but not all raw JSON semantics and telemetry. New Door records retain verification, per-case CPU snapshots and wait diagnostics. The supplied futex file is an aggregate Markdown export, not its full raw JSON. These distinctions limit reanalysis.

## A2.2. Workload, timing, and comparison rules

The corrected SHM comparison uses stream/fixed payloads of 64, 1024 and 16384 bytes, 128 slots per client, windows 1 and 32, 100000 measured operations and 1000 warmup operations **per client**, five repetitions, eleven M:N layouts: 1:1, 1:2, 1:4, 2:1, 2:2, 2:4, 4:1, 4:4, 8:1, 8:2 and 8:4. Door client/server CPU lists are empty: placement is unbound. No disjoint CPU-set or reserved telemetry-core experiment is claimed for this run. In original matrix rows the worker field is a requested configuration value; not every engine instantiates that many server workers. In particular the Unix socket implementation uses one server event loop. Do not read a matrix label such as 8:4 as proof of four active socket-server threads. The IPC matrix operation count is total; direct-ping and SHM M:N counts are per client. Different counts can change run duration and scheduling exposure.

In the common SHM workload the client fills each payload, the server transforms it and the client checks every response byte. Queue counters and slots bound in-flight work. Ring and reused-chain descriptors are prepared before measurement; reuse does not include allocating/releasing every operation. Dynamic pool mode includes reserve/release costs and clearing. Zero-copy means no intermediate IPC payload copy; filling, transforming, verification, cache traffic and clearing still consume memory bandwidth.

Single-channel batch flow publishes a group and drains it; stream flow overlaps admission and completion. The synthetic mixed profile selects 50% min(256,max), 35% min(1024,max), 13% min(16384,max), and 2% max bytes. Max payload is not the average message size. Fixed slots/reused chains retain maximum capacity, while dynamic chains reserve actual length; memory budgets differ. Single-channel SHM has no separate warmup phase in this harness.

Throughput is completed operations divided by the measurement wall interval, including the slowest client and start skew. Aggregate ns/op is its inverse and is not request RTT. SHM latency is **fill-to-verified-response**; this includes the time a record spends behind other records. Native async measures its own completion boundary. Direct ONEWAY may report submit latency separately from eventual completion; a submit percentile must never be ranked against request/reply completion. Direct-ping SHM and full-byte SHM transport workloads also do not perform identical work. Original engine/test labels are retained in the detailed tables.

RPS, ns/op and p50/p95/p99/p99.9 are medians of repetition-level values; percentiles are not a merged histogram. Quantized histogram percentiles have approximately 1.6% bucket resolution. Original campaign `Max` is the median of repetition maxima, as emitted by its runner; new Door/futex Max is the largest repetition maximum. The two Max conventions must not be mixed. Ranges in detailed tables are repetition min/max throughput. Five repetitions give descriptive repeatability, not a confidence interval.

Socket, pipe, Door invocation and SHM engines differ in framing, dispatch, payload checks and buffering. Their ratios characterize end-to-end test implementations, not pure syscall overhead. Match flow/profile, payload, topology, outstanding window and completion boundary before comparison. M:N throughput is not one-operation latency, PPS on a NIC, or HTTP/PostgreSQL capacity.

CPU snapshots from `kern.cp_times` cover the entire host and case boundaries can include setup/warmup; backend RSS is not total physical memory for the chain. The original planned VNET cross-jail topology, reserved physical telemetry core, full process counters and complete histograms are not established by these supplied artifacts. They remain possible future campaigns, not measured features here.

## A2.3. Unix socket versus corrected SHM: matched 1:1, window 32

| Payload B | Unix stream Mops/s | Unix seqpacket Mops/s | Door Ring Mops/s | Door Pool reuse Mops/s | SHM futex Mops/s | Ring / stream | Futex / stream |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 64 | 0.598 | 0.587 | 3.748 | 3.771 | 3.149 | 6.27× | 5.27× |
| 1024 | 0.383 | 0.381 | 1.205 | 1.207 | 1.226 | 3.15× | 3.20× |
| 16384 | 0.055 | 0.058 | 0.104 | 0.104 | 0.105 | 1.91× | 1.91× |

The shared-memory implementations exceed the measured socket implementation in these configurations. This supports choosing persistent memory and batched notification for this workload; it does not identify the isolated cost of socket copying or prove the same ratio in an application.

## A2.4. Corrected Pair versus futex: throughput and scaling

### Payload 64 bytes; window 32

| Clients:workers | Ring Mops/s | Pool reuse Mops/s | Futex Mops/s | Ring / futex | Pool / futex |
| --- | --- | --- | --- | --- | --- |
| 1:1 | 3.748 | 3.771 | 3.149 | 1.190 | 1.198 |
| 1:2 | 3.428 | 3.437 | 3.463 | 0.990 | 0.993 |
| 1:4 | 3.072 | 3.060 | 3.078 | 0.998 | 0.994 |
| 2:1 | 5.977 | 6.443 | 7.939 | 0.753 | 0.812 |
| 2:2 | 6.346 | 6.341 | 6.714 | 0.945 | 0.945 |
| 2:4 | 5.137 | 5.311 | 5.227 | 0.983 | 1.016 |
| 4:1 | 16.003 | 15.858 | 14.972 | 1.069 | 1.059 |
| 4:4 | 8.061 | 7.637 | 7.803 | 1.033 | 0.979 |
| 8:1 | 14.085 | 14.094 | 7.522 | 1.873 | 1.874 |
| 8:2 | 16.243 | 16.383 | 20.480 | 0.793 | 0.800 |
| 8:4 | 11.996 | 12.576 | 13.103 | 0.915 | 0.960 |

### Payload 1024 bytes; window 32

| Clients:workers | Ring Mops/s | Pool reuse Mops/s | Futex Mops/s | Ring / futex | Pool / futex |
| --- | --- | --- | --- | --- | --- |
| 1:1 | 1.205 | 1.207 | 1.226 | 0.982 | 0.984 |
| 1:2 | 1.160 | 1.162 | 1.148 | 1.010 | 1.012 |
| 1:4 | 1.091 | 1.095 | 1.036 | 1.053 | 1.057 |
| 2:1 | 2.277 | 2.277 | 2.279 | 0.999 | 0.999 |
| 2:2 | 2.257 | 2.255 | 2.207 | 1.023 | 1.022 |
| 2:4 | 2.012 | 2.001 | 1.924 | 1.046 | 1.040 |
| 4:1 | 3.037 | 2.999 | 3.049 | 0.996 | 0.983 |
| 4:4 | 3.416 | 3.476 | 3.412 | 1.001 | 1.019 |
| 8:1 | 2.912 | 2.914 | 3.010 | 0.968 | 0.968 |
| 8:2 | 5.030 | 5.017 | 5.005 | 1.005 | 1.002 |
| 8:4 | 4.799 | 4.776 | 4.795 | 1.001 | 0.996 |

### Payload 16384 bytes; window 32

| Clients:workers | Ring Mops/s | Pool reuse Mops/s | Futex Mops/s | Ring / futex | Pool / futex |
| --- | --- | --- | --- | --- | --- |
| 1:1 | 0.104 | 0.104 | 0.105 | 0.996 | 0.994 |
| 1:2 | 0.098 | 0.097 | 0.098 | 1.000 | 0.993 |
| 1:4 | 0.097 | 0.096 | 0.097 | 1.005 | 0.998 |
| 2:1 | 0.216 | 0.216 | 0.217 | 0.997 | 0.996 |
| 2:2 | 0.207 | 0.207 | 0.207 | 1.000 | 0.999 |
| 2:4 | 0.190 | 0.189 | 0.191 | 0.992 | 0.986 |
| 4:1 | 0.421 | 0.420 | 0.376 | 1.120 | 1.115 |
| 4:4 | 0.422 | 0.313 | 0.322 | 1.311 | 0.974 |
| 8:1 | 0.425 | 0.421 | 0.430 | 0.987 | 0.978 |
| 8:2 | 0.505 | 0.494 | 0.511 | 0.988 | 0.967 |
| 8:4 | 0.525 | 0.531 | 0.523 | 1.004 | 1.015 |

At 1 KiB and 8:2 all three paths reach approximately 5 million operations/s; the difference between their medians is below 1%. At 16 KiB they remain close. At 64 B and 8:2 futex reaches 20.480 Mops/s versus 16.243/16.383 Mops/s for Door Ring/Pool reuse, about 26%/25% higher. At 1:1 the Door median is higher, but the measured 64-byte throughput ranges overlap: Ring 3.226–3.854 Mops/s and futex 3.094–4.002 Mops/s. That case does not establish a stable universal advantage.

At 64 B adding workers from 8:2 to 8:4 lowers throughput and worsens tails for all three paths. These measurements expose a topology-sensitive limit; they do not separate scheduler, cache-coherence, wakeup or lock-contention costs. Ordinary SHM+umtx does not implement Door participant authorization or generation-bound resource management, so equal throughput is not equal functionality.

## A2.5. Tail latency and notification regression

| Payload B | Method, 8:2/window 32 | p50 µs | p95 µs | p99 µs | p99.9 µs | Max µs |
| --- | --- | --- | --- | --- | --- | --- |
| 64 | Door Ring + Pair | 8.128 | 21.760 | 62.464 | 458.752 | 2009.502 |
| 64 | Door Pool reuse + Pair | 8.128 | 23.040 | 64.000 | 471.040 | 1986.505 |
| 64 | SHM + futex/umtx | 7.872 | 13.184 | 18.432 | 62.976 | 912.225 |
| 1024 | Door Ring + Pair | 38.912 | 50.176 | 59.904 | 72.704 | 269.931 |
| 1024 | Door Pool reuse + Pair | 38.912 | 51.712 | 61.952 | 73.728 | 740.840 |
| 1024 | SHM + futex/umtx | 39.424 | 50.176 | 59.392 | 71.680 | 141.339 |
| 16384 | Door Ring + Pair | 368.640 | 507.904 | 520.192 | 540.672 | 796.204 |
| 16384 | Door Pool reuse + Pair | 368.640 | 512.000 | 524.288 | 540.672 | 735.788 |
| 16384 | SHM + futex/umtx | 368.640 | 507.904 | 520.192 | 540.672 | 866.567 |

Both corrected Door files report 330/330 verified cases, no failures/skips, SC sleep handshakes and zero client wait timeouts in all 660 records. Their largest observed delays across all configurations were 6.942 ms (Ring) and 6.896 ms (Pool reuse), rather than the former approximately one-second fallback. This is evidence for the tested correction, not a proof covering all interleavings. Futex raw timeout diagnostics were not supplied.

## A2.6. Application-level experiments

These application measurements are retained from the reported console results and are distinct from the IPC matrix. No missing raw runs or server CPU measurements are reconstructed. They demonstrate the scope in which a transport advantage does or does not remain visible.

### HTTP RAM backend: 64-byte response, no database

Three 30-second runs per variant, four load-generator threads, 384 connections, HTTP/1.1 over loopback. All requests succeeded. Door uses the backend RAM path; FastCGI uses `/fcgi-ram`. Values below are medians of the three console summaries; client CPU is percent of one logical core.

| Path | Successful requests/s | Mean ms | p95 ms | p99 ms | Client CPU % |
| --- | --- | --- | --- | --- | --- |
| Door Pair RAM | 194831.979 | 1.948 | 2.584 | 3.140 | 348.5 |
| Unix socket FastCGI RAM | 149198.732 | 2.565 | 3.040 | 3.615 | 389.4 |

Door throughput is approximately 30.6% higher in this setup. This comparison includes nginx, application dispatch and the client; it is not an isolated IPC transport ratio. Reconnects occurred in both variants; their cause is not established by the console summaries.

<a id="http-ram-follow-up"></a>

#### Follow-up HTTP RAM campaign: kqueue notification bridge

The follow-up report uses the same stated RAM workload shape: HTTP/1.1 on
loopback, 64-byte replies, 384 connections, four generator threads and
30-second runs. These are the reported summary values; there are no
additional raw samples or server-side CPU counters in this record.

| Path | Successful RPS | Mean ms | p50 ms | p95 ms | p99 ms | Client CPU % | Errors |
|---|---:|---:|---:|---:|---:|---:|---:|
| Door Pair RAM | 218528.197 | 1.741 | 1.675 | 2.474 | 2.943 | 344.237 | 0 |
| Unix socket FastCGI RAM | 149169.402 | 2.566 | 2.556 | 3.007 | 3.858 | 390.056 | 0 |

Door completed **46.5% more requests/s** than FastCGI in this comparison,
and **12.2% more requests/s** than in the earlier Door RAM series. This
is a follow-up after the bridge change, not a controlled isolation of
its cost. The loopback generator competes for host CPU and reports
client-side CPU only. Do not use these percentages to infer the effect
on the SQL-backed feed or on an external generator.

### HTTP PostgreSQL read workload: offered rate 8000/s

Three runs per variant: 32 users, four client threads, depth 12 (384 connections), 30 seconds, read/feed mix, seed 1. Every offered request succeeded; drops and errors were zero. These are open-loop rate-limited tests, not maximum-throughput tests.

| Path | Successful RPS | Mean ms | p95 ms | p99 ms | Transfer/processing mean ms | Scheduler mean ms | Client CPU % |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Door HTTP + PGDoor direct | 7999.737 | 1.169 | 1.494 | 1.619 | 1.026 | 0.137 | 307.766 |
| Door HTTP + libpq | 7999.736 | 1.307 | 1.587 | 1.702 | 1.143 | 0.158 | 318.003 |
| FastCGI + libpq | 7999.398 | 3.123 | 7.630 | 11.383 | 2.936 | 0.181 | 309.059 |

The nearly equal RPS follows from the fixed offered rate. The observed advantage is lower latency, not demonstrated spare capacity. `worker_elapsed` in the load generator is a client-side transfer/completion interval, not application-worker or PostgreSQL CPU time. In a separate single run offered at 10000/s, PGDoor completed 9389.791/s, Door+libpq 9163.850/s and FastCGI+libpq 9158.613/s with user-window drops; these three single runs are not a five-repeat capacity estimate.

### PGDoor versus libpq: direct encoding and larger results

One client and one database worker/backend, full byte verification, median of three repetitions. The ordinary Door path uses SPI_getvalue; direct encoding writes supported Datum representations into the destination and falls back for other types. Both expose textual cells. These compare Door+SHM+SPI against libpq and PostgreSQL protocol processing over a Unix socket, not the transport alone.

| Case | Door ops/s | Door direct ops/s | libpq ops/s | Direct / libpq | Door mean µs | Direct mean µs | libpq mean µs |
| --- | --- | --- | --- | --- | --- | --- | --- |
| SELECT 1 | 154750 | 159373 | 67803 | 2.35× | 6.42 | 6.23 | 14.70 |
| 64 B | 150317 | 152664 | 67786 | 2.25× | 6.61 | 6.50 | 14.71 |
| 1 KiB | 147180 | 149300 | 64246 | 2.32× | 6.74 | 6.65 | 15.52 |
| 16 KiB | 66125 | 79522 | 32848 | 2.42× | 15.07 | 12.53 | 30.39 |
| 1 MiB | 1806 | 2610 | 1399 | 1.87× | 553.54 | 383.04 | 714.57 |
| 8 MiB | 120 | 193 | 84 | 2.30× | 8304.93 | 5182.92 | 11904.02 |
| 21 rows × 128 B | 32394 | 35895 | 23633 | 1.52× | 30.82 | 27.81 | 42.26 |

Earlier equal-worker scaling, before the direct-encoding campaign: SELECT 1 reached 567744 versus 215502 ops/s at 4:4 and 598412 versus 264853 ops/s at 8:8 (Door versus libpq). The supplied real feed SQL reached 12105 versus 11837 ops/s at 4:4, but 15304 versus 16588 at 8:8. Query execution therefore can dominate or reverse a microbenchmark transport advantage. The direct-encoding and equal-worker campaigns are separate measurements.

## Continue reading

- [A2.7. Complete corrected SHM data](benchmark-data/02-07-corrected-shm.md)
- [A2.8. Retained original campaign data](benchmark-data/02-08-original-campaign.md)
- [A2.9. Source artifacts and repeatability](benchmark-data/02-09-provenance.md)
