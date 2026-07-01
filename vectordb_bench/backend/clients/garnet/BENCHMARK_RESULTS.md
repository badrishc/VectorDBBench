# Garnet Vector Search — In‑Memory vs Disk‑Tiered Benchmark Results

This document reports a full set of VectorDBBench measurements for Garnet
[Vector Sets](https://github.com/microsoft/garnet) (DiskANN graph index, **NOQUANT / FP32**)
on the **Wikipedia‑10M + Cohere 768‑dim** dataset (`Performance768D10M`, COSINE).

The goal is to quantify what happens when the same graph is served **entirely
from memory** versus **served from disk** (Linux native **O_DIRECT** device with
libaio, OS page cache bypassed) after the raw vectors have been evicted.

All runs use the same parameters: `max_degree=16`, `l_build=300`, `l_search=192`,
`k=100`, and the concurrency sweep **5, 10, 20, 40, 60, 80, 120, 140**
(30 s per level). The client is co‑located with the server on a single host.

---

## TL;DR

| Metric (10M, md=16, l_build=300, l_search=192, k=100) | In‑memory | Disk (libaio, O_DIRECT) |
|---|--:|--:|
| **Recall@100** | 0.838 | **0.8393** |
| **Serial p99 latency** | 3.8 ms | 96 ms |
| **Peak QPS** | **16,906** (C140) | 215 (C40) |
| **Peak read IOPS** | — | ~519K |
| **Peak read bandwidth** | — | ~1.69 GB/s |

**Headlines**

- **Recall is preserved off disk** — 0.838 in‑memory vs **0.8393** disk‑served.
  Serving raw vectors from disk changes *speed*, not *which* neighbors are found.
- Moving raw vectors to disk costs **~80× throughput** (16,906 → ~215 QPS) and
  **~25× serial latency** (3.8 → ~96 ms). The graph adjacency ("stub") stays
  memory‑resident; only the 3,072‑byte FP32 vectors are read from NVMe.
- The disk workload reaches **~519K read IOPS / ~1.7 GB/s** — about **80 % of the
  device's ceiling for this access pattern** (see [SSD saturation](#ssd-saturation-analysis)).
  The workload is **not** SSD‑bound; the limiter is DiskANN's per‑query serial
  IO chain plus co‑locating the client with the server on one box.

---

## Test environment

| | |
|---|---|
| **CPU** | 2× Intel Xeon Platinum 8380 — 80 physical cores / 160 threads |
| **RAM** | 503 GB |
| **NVMe** | Dell Ent NVMe P5600 MU 3.2 TB (`/dev/nvme0n1`, mounted `/DATA2`, 512‑byte sectors) |
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
so every vector is resident in the mutable log region. RSS ≈ 32 GB.

### Disk‑tiered server (native O_DIRECT device)

```
GarnetServer --enable-vector-set-preview \
  --index 1g --memory 64g --page 64m \
  --storage-tier --logdir /DATA2/badrishc/garnet_ckpt \
  --checkpointdir /DATA2/badrishc/garnet_ckpt \
  --device-type Native --device-io-backend Libaio \
  --device-completion-threads 16 \
  --enable-debug-command local --recover
```

`--device-type Native` uses the Linux **O_DIRECT** device (bypasses the OS page
cache), confirmed by `libnative_device.so` + `libaio.so` in `/proc/<pid>/maps`.
To move the graph to disk after loading:

```
DEBUG FLUSHANDEVICT      # requires --enable-debug-command local
# -> OK head=<tail> tail=<tail>   (Head advances to Tail: all records evicted)
```

**Methodology for the disk run:** recover the 10M checkpoint → `FLUSHANDEVICT`
→ a warm‑up pass (C60, discarded) to populate the in‑memory stub cache →
then measure serial recall + the concurrency sweep. This reports **steady‑state
(warm‑stub)** disk performance, the realistic production case.

### Caching policy (what stays in memory vs goes to disk)

Garnet keeps the graph **stub** memory‑resident and serves only the bulk raw
vectors from disk. On a disk read, each Vector‑Set namespace is either copied to
the main‑log tail (cached) or left on disk:

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

**Verified after the sweep:** `Log.HeadAddress` unchanged (all 10M raw vectors
stay on disk) and the warm stub region (`Log.TailAddress − Log.HeadAddress`) was
only ~20 MiB for this query set — the adjacency/id‑map stubs actually touched by
beam search. Confirms "cache the adjacency, read the raw vectors from disk."

---

## Results

### 1. In‑memory concurrency sweep

`recall = 0.838`, `serial p99 = 3.8 ms`, `p95 = 3.3 ms`, pure in‑memory.

| Concurrency | 5 | 10 | 20 | 40 | 60 | 80 | 120 | 140 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **QPS** | 1,345 | 2,474 | 5,478 | 10,054 | 13,359 | 14,748 | 16,526 | **16,906** |
| **p99 (ms)** | 5.6 | 5.8 | 5.3 | 6.0 | 7.4 | 9.8 | 14.5 | 15.9 |

Throughput scales almost linearly to C40 (~10k QPS) and reaches the **knee at
C140 = 16,906 QPS** (p99 15.9 ms), the CPU‑bound in‑memory ceiling for this host.
(A wider sweep in earlier testing confirmed the plateau: C220 ≈ 17,059 and C260 ≈
17,077 add only ~1 % QPS for +2–3× the p99, so C140 is the effective peak.)

### 2. Disk‑served concurrency sweep — libaio (O_DIRECT)

`recall = 0.8393`, `ndcg = 0.8537`, `serial avg = 60.6 ms`, `p99 = 96 ms`, `p95 = 84 ms`.

| Concurrency | 5 | 10 | 20 | 40 | 60 | 80 | 120 | 140 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **QPS** | 71 | 121 | 181 | **215** | 211 | 209 | 197 | 189 |
| **p99 (ms)** | 108 | 125 | 165 | 278 | 441 | 584 | 919 | 1151 |
| **p95 (ms)** | 88 | 101 | 134 | 229 | 357 | 504 | 781 | 932 |

Peak **215 QPS @ C40**, then a gentle, graceful decline as latency climbs — the
device is saturated at C40 and extra client concurrency only deepens the queue.
Peak device: **~519K read IOPS, ~1.69 GB/s, r_await 0.22 ms, QD ≈ 115, %util 100 %**
(request size ≈ 3.3 KB = the FP32 vector).

---

## SSD saturation analysis

Device ceiling measured with `fio` (O_DIRECT, libaio, 16 jobs, iodepth 64 ≈ QD
1024) on a 20 GB file on `/dev/nvme0n1`:

| fio profile | IOPS | Bandwidth |
|---|--:|--:|
| 4 KB random read | **758K** | 3.06 GB/s |
| 3,072 B random read, **4 KB‑aligned** (`ba=4096`) | **757K** | 2.33 GB/s |
| 3,072 B random read, 512‑aligned (= Garnet's access) | **648K** | 1.99 GB/s |

**Reading the ceiling correctly.** The device's absolute random‑read ceiling is
**~758K IOPS** (4 KB). Garnet's raw vectors are **3,072 bytes** and land at
512‑byte alignment, so each random read **straddles the device's 4 KB NAND page**
and the ceiling for *that access pattern* drops to **~648K IOPS** (a ~15 %
alignment penalty — the same 3,072‑byte read, forced to 4 KB alignment, recovers
the full 757K).

Garnet's disk‑served workload reaches **~519K IOPS**:

- **≈ 80 % of the size‑matched 648K ceiling** (≈ 68 % of the 758K 4 KB peak).
- `%util = 100 %` on NVMe means "never idle," **not** "max IOPS" — the device
  still has headroom at our achieved queue depth (~115 vs fio's 1024).
- Raising `--device-completion-threads` from the default 4 to 16 made no
  difference to IOPS, so the completion‑drain path is not the limit.
- The real limiter is DiskANN's **per‑query serial dependency chain** — each
  query issues on the order of a thousand mostly‑sequential vector reads
  (`l_search=192`), giving low per‑query IO parallelism — compounded by the
  benchmark client sharing the 160‑thread box with the server.

**Actionable:** 4 KB‑aligning (or padding) the stored vectors would lift the
per‑read device ceiling ~648K → ~758K (≈ 15 % more IOPS headroom). Reducing reads
per query (quantization / a hot‑vector read cache) or adding per‑query IO
parallelism (wider beam) would push closer to that ceiling; running the client on
a separate host would remove the co‑location CPU contention.

---

## Key conclusions

1. **Recall is unaffected by tiering.** 0.838 (memory) ≈ 0.8393 (disk). Disk
   tiering trades latency/throughput, not accuracy.
2. **Cost of serving raw vectors from disk:** ~80× lower QPS (16,906 → ~215) and
   ~25× higher serial latency (3.8 → ~96 ms) at these parameters.
3. **The stub is tiny and stays hot.** The adjacency + id‑map stubs actually
   touched by the query set occupy only ~20 MiB of RAM; the ~31 GiB of raw
   vectors live on NVMe and are read on demand.
4. **~80 % of the NVMe's ceiling for this access pattern.** The workload is
   bounded by DiskANN's serial per‑query IO and client/server co‑location, not by
   the SSD or the completion threads. The device's absolute ceiling is ~758K
   IOPS; 3,072‑byte 512‑aligned reads cap at ~648K, and Garnet reaches ~519K of that.

---

*Reproduce with the disk‑tiered configuration above, then run
`vectordbbench garnet --case-type Performance768D10M --max-degree 16
--l-build 300 --l-search 192 --k 100 --skip-drop-old --skip-load
--num-concurrency 5,10,20,40,60,80,120,140`. See [README.md](./README.md) for the
full build‑and‑run walkthrough.*
