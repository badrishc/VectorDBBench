# Garnet Vector Search — In‑Memory vs Disk‑Tiered Benchmark Results

This document reports a full set of VectorDBBench measurements for Garnet
[Vector Sets](https://github.com/microsoft/garnet) (DiskANN graph index, **NOQUANT / FP32**)
on the **Wikipedia‑10M + Cohere 768‑dim** dataset (`Performance768D10M`, COSINE).

The goal is to quantify what happens when the same graph is served **entirely
from memory** versus **served from disk** after the raw vectors have been
evicted, and to compare the two Linux native‑device IO backends
(**libaio** and **io_uring**) under O_DIRECT (OS page cache bypassed).

---

## TL;DR

| Metric (10M, max_degree=16, l_build=300, l_search=192, k=100) | In‑memory | Disk (libaio, O_DIRECT) | Disk (io_uring, O_DIRECT) |
|---|--:|--:|--:|
| **Recall@100** | 0.838 | **0.8393** | **0.8393** |
| **Serial p99 latency** | 3.8 ms | 98 ms | 121 ms |
| **Peak QPS** | **17,077** (C260) | 212 (C60) | 210 (C30) |
| **Peak read IOPS** | — | ~520K | ~508K |
| **Peak read bandwidth** | — | ~1.6 GB/s | ~1.65 GB/s |

**Headlines**

- **Recall is preserved off disk** — 0.838 in‑memory vs 0.839 disk‑served
  (identical for both IO backends). Serving raw vectors from disk changes
  *speed*, not *which* neighbors are found.
- Moving raw vectors to disk costs **~80× throughput** (17,077 → ~212 QPS) and
  **~25× serial latency** (3.8 → ~98 ms). The graph adjacency ("stub") stays
  memory‑resident; only the 3,072‑byte FP32 vectors are read from NVMe.
- Both IO backends reach **~510K read IOPS / ~1.6 GB/s** — about **80 % of the
  device's 3 KB random‑read ceiling** (641K IOPS). The workload is **not** SSD‑
  bound; the limiter is the per‑query serial dependency chain of DiskANN beam
  search plus co‑locating the benchmark client with the server on one box.

---

## Test environment

| | |
|---|---|
| **CPU** | 2× Intel Xeon Platinum 8380 — 80 physical cores / 160 threads |
| **RAM** | 503 GB |
| **NVMe** | Dell Ent NVMe P5600 MU 3.2 TB (`/dev/nvme0n1`, mounted `/DATA2`) |
| **Garnet** | `main` @ `f396fd257` (PR #1901 merged), .NET 10, Release build |
| **Dataset** | Cohere `cohere_large_10m` (Wikipedia + Cohere 768‑dim), 44 GB, COSINE |
| **Index** | DiskANN, NOQUANT (FP32); `max_degree=16`, `l_build=300`, `l_search=192`, `k=100` |
| **Client** | VectorDBBench, `Performance768D10M`, co‑located on the same host |

Raw FP32 vector = 768 × 4 = **3,072 bytes** (stored inline; a single device read
per vector, since `--value-overflow-threshold` default 16k > 3,072).

---

## Configuration

### In‑memory server (no storage tier)

```
GarnetServer --enable-vector-set-preview \
  --index 1g --memory 64g --page 64m
```

Verified pure in‑memory: `Log.HeadAddress == Log.BeginAddress == 64` (no spill),
RSS ≈ 32.4 GB (30.85 GiB log + 1.02 GiB index).

### Disk‑tiered server (native O_DIRECT device)

```
GarnetServer --enable-vector-set-preview \
  --index 1g --memory 64g --page 64m \
  --storage-tier --logdir /DATA2/badrishc/garnet_ckpt \
  --checkpointdir /DATA2/badrishc/garnet_ckpt \
  --device-type Native --device-io-backend {Libaio|Uring} \
  --device-completion-threads 16 \
  --enable-debug-command local --recover
```

`--device-type Native` uses the Linux **O_DIRECT** device (bypasses the OS page
cache), confirmed by `libnative_device.so` + `libaio.so` / `liburing.so` in
`/proc/<pid>/maps`. To move the graph to disk after loading:

```
DEBUG FLUSHANDEVICT      # requires --enable-debug-command local
# -> OK head=<tail> tail=<tail>   (Head advances to Tail: all records evicted)
```

### Caching policy (what stays in memory vs goes to disk)

Garnet keeps the graph **stub** memory‑resident and serves only the bulk raw
vectors from disk. On a disk read, each Vector‑Set namespace is copied to the
main‑log tail (or read cache when `--readcache` is on) or left on disk:

| Namespace | Record | Copied to memory on read? |
|---|---|:--:|
| 0 | **FullVector** (raw FP32, 3,072 B) | ✗ — served from disk each read |
| 1 | NeighborList (adjacency) | ✓ cached |
| 2 | QuantizedVector | ✓ cached |
| 3 | Attributes | ✗ (only read by *filtered* search) |
| 4 | Metadata | ✗ (set‑level lifecycle only) |
| 5 | InternalIdMap | ✓ cached |
| 6 | ExternalIdMap | ✓ cached |

This workload is **unfiltered**, so Attributes (ns 3) and Metadata (ns 4) are
never read per query — they are intentionally not cached.

**Verified after the query sweep:** `Log.HeadAddress` unchanged (all 10M raw
vectors stay on disk) and `Log.TailAddress` grew only **+45.2 MiB** = the
adjacency/id‑map stubs copied to the tail — just **0.14 %** of the 30.9 GiB of
data. Confirms "cache the adjacency, read the raw vectors from disk."

---

## Results

### 1. In‑memory concurrency sweep (box maximum)

`recall = 0.838`, `serial p99 = 3.8 ms`, pure in‑memory.

| Concurrency | 10 | 30 | 60 | 100 | 140 | 180 | 220 | 260 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **QPS** | 2,477 | 8,057 | 13,361 | 15,389 | 16,895 | 16,951 | 17,059 | **17,077** |
| **p99 (ms)** | 6.0 | 5.7 | 7.7 | 13.3 | 16.5 | 24.5 | 29.0 | 42.0 |

QPS saturates the box at **~17,000 QPS** around C140–180; beyond that only p99
grows. This is the CPU‑bound in‑memory ceiling for this host.

### 2. Disk‑served concurrency sweep — libaio (O_DIRECT)

`recall = 0.8393`, `ndcg = 0.8537`, `serial avg = 60.4 ms`, `p99 = 98 ms`, `p95 = 83 ms`.

| Concurrency | 10 | 30 | 60 | 100 | 160 | 220 |
|---|--:|--:|--:|--:|--:|--:|
| **QPS** | 107 | 209 | **212** | 203 | 184 | 182 |

Peak **212 QPS @ C60**, then a gentle decline — **graceful** under high client
concurrency. During the sweep the device sustained **~520K read IOPS, ~1.6 GB/s,
QD≈117, %util=100 %**, request size ~3.3 KB (= the 3,072‑byte FP32 vector).

### 3. Disk‑served concurrency sweep — io_uring (O_DIRECT)

`recall = 0.8393`, `ndcg = 0.8537`, `serial avg = 74.5 ms`, `p99 = 120.6 ms`, `p95 = 102.7 ms`.

| Concurrency | 10 | 30 | 60 | 100 | 160 | 220 |
|---|--:|--:|--:|--:|--:|--:|
| **QPS** | 122 | **210** | 183 | 101 | ✗ collapse | — |

Peak **210 QPS @ C30**. At **C160 the server became unresponsive** (mass client
socket‑read timeouts, `r/s = 0`, `PING` timed out) and did not recover — the run
had to be aborted. This was **reproduced twice** (on a long‑lived server and
again on a freshly‑recovered server), so it is not transient state corruption.
At moderate concurrency (≤ C100) io_uring matches libaio on IOPS (**~508K IOPS,
~1.65 GB/s, QD≈108, %util=100 %**).

> **Interpretation (fair framing).** The benchmark client (VectorDBBench's
> `ProcessPoolExecutor`) spawns *C* client processes on the **same 160‑thread
> host** as the server. At C160+, 160 client processes plus the server threads
> plus 16 device‑completion threads heavily oversubscribe the CPU. io_uring's
> completion path is more CPU‑active than libaio's blocking `io_getevents`, so
> under this contention it appears to starve and wedge, while libaio degrades
> gracefully. This is most likely a **co‑location / CPU‑oversubscription
> artifact**, not a fundamental io_uring throughput limit — at ≤ C100 both
> backends deliver identical IOPS and peak QPS. A **dedicated client host**
> would be needed to characterize io_uring beyond C100 cleanly.

### 4. libaio vs io_uring (disk‑served, O_DIRECT)

| | libaio | io_uring |
|---|--:|--:|
| Recall@100 | 0.8393 | 0.8393 |
| Serial p99 | 98 ms | 121 ms |
| Peak QPS | 212 (C60) | 210 (C30) |
| Peak read IOPS | ~520K | ~508K |
| Peak read BW | ~1.6 GB/s | ~1.65 GB/s |
| Peak queue depth | ~117 | ~108 |
| Behavior at C160+ | graceful decline (182–184 QPS) | server wedged (co‑located client) |

**On this shared‑box setup, libaio is the more robust choice.** Both backends
are equivalent at moderate concurrency; libaio holds up under extreme client
oversubscription where io_uring did not.

---

## SSD saturation analysis

Device ceiling measured with `fio` (O_DIRECT, libaio) on the same `/dev/nvme0n1`:

| fio profile | IOPS | Bandwidth |
|---|--:|--:|
| 4 KB random read, 8–16 jobs, QD512 | 748K | 3.06 GB/s |
| **3 KB random read, 8 jobs** (≈ our vector read) | **641K** | 1.88 GB/s |
| 4 KB random read, single job, QD128 | 285K | 1.17 GB/s |

Garnet's disk‑served workload reaches **~510–520K IOPS** = **~80 % of the 3 KB
random‑read ceiling (641K)**. It is heavily utilized but **not fully saturated**:

- `%util = 100 %` on NVMe means "never idle," **not** "max IOPS" — the device
  still has ~20 % IOPS headroom at our achieved queue depth (~110 vs fio's 512).
- Raising `--device-completion-threads` from the default 4 to **16 made no
  difference** (~520K either way), so the completion‑drain path is not the limit.
- The real limiter is the **per‑query serial dependency chain** of DiskANN beam
  search — each query issues on the order of a thousand mostly‑sequential vector
  reads (`l_search=192`), giving low per‑query IO parallelism — compounded by the
  benchmark client sharing the 160‑thread box with the server.

To push closer to the device ceiling you would reduce reads per query
(quantization / PQ, or a read cache for hot vectors) or add per‑query IO
parallelism (wider beam), and run the client on a separate host.

---

## Key conclusions

1. **Recall is unaffected by tiering.** 0.838 (memory) ≈ 0.839 (disk), identical
   for libaio and io_uring. Disk tiering trades latency/throughput, not accuracy.
2. **Cost of serving raw vectors from disk:** ~80× lower QPS (17,077 → ~212) and
   ~25× higher serial latency (3.8 → ~98 ms) at these parameters.
3. **The stub is tiny and stays hot.** Caching adjacency + id‑maps to memory
   added only 45 MiB (0.14 % of the data); the 30.9 GiB of raw vectors live on
   NVMe and are read on demand.
4. **~80 % of the NVMe's random‑read ceiling** is reached by both IO backends;
   the workload is bounded by DiskANN's serial per‑query IO and client/server
   co‑location, not by the SSD or the completion threads.
5. **libaio ≥ io_uring for robustness here.** Equivalent at ≤ C100; libaio
   degrades gracefully at high concurrency while io_uring wedged under CPU
   oversubscription from the co‑located client.

---

*Reproduce with the disk‑tiered configuration above, then run
`vectordbbench garnet --case-type Performance768D10M --max-degree 16
--l-build 300 --l-search 192 --k 100 --skip-drop-old --skip-load
--num-concurrency 10,30,60,100,160,220`. See [README.md](./README.md) for the
full build‑and‑run walkthrough.*
