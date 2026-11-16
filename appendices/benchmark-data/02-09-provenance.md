[Monograph index](../../README.md) · [Full text](../../door-ipc-implementation-for-freebsd_full.md)

## A2.9. Source artifacts and repeatability

For corrected Pair reruns use `tools/ipc-suite.py --scope 3 --transport 25` (Ring) or `26` (Pool reuse), and `34` for ordinary SHM+futex. Supply `--config`, a new `--directory`, and `--clients 8 --workers 4 --slots 128 --window 32 --max-bytes 16384 --operations 100000 --warmup 1000 --repeats 5`. Record matching installed kernel/libdoor/headers, build options, placement, and the full `results.jsonl` and `summary.json` files. Run variants sequentially without concurrent benchmark load. A future confidence-oriented campaign should alternate/randomize methods and retain complete raw timing and process counters.

---
