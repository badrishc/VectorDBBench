# Garnet Vector Search — Quantization and In-Memory vs Disk-Tiered Results

VectorDBBench measurements for Garnet [Vector Sets](https://github.com/microsoft/garnet)
(DiskANN graph index) on the **Wikipedia-10M + Cohere 768-dim** dataset
(`Performance768D10M`, COSINE).

Two axes are covered:

1. **Quantization** — `NOQUANT` (raw FP32 traversal) vs **`Q8`** (8-bit quantized
   traversal with FP32 rerank).
2. **Tiering** — served **entirely from memory** vs **from the NVMe storage tier**
   (Linux native **O_DIRECT** device, OS page cache bypassed) after the log has been
   evicted.

All runs use `max_degree=16`, `l_build=300`, `l_search=192`, `k=100`, 1,000 held-out
query vectors, and the concurrency sweep **5, 10, 20, 40, 60, 80, 120, 140**
(30 s per level). The client is co-located with the server on a single host.

> **The headline result is the Q8 disk-tiered number.** It started at 86 ms/query and is
> now **3.3 ms** — a **26x** improvement at unchanged recall — putting disk-tiered
> serving within ~20 % of *in-memory* Q8 latency. See
> [The Q8 disk-tiered optimization arc](#the-q8-disk-tiered-optimization-arc).

---

## TL;DR

| Metric (10M, md=16, l_build=300, l_search=192, k=100) | NOQUANT in-mem | **Q8 in-mem** | NOQUANT disk | **Q8 disk (current)** |
|---|--:|--:|--:|--:|
| **Recall@100** | 0.8382 | 0.8408 | 0.8382 | **0.8408** |
| **Serial avg latency** | 2.7 ms | ~2.7 ms | 84.5 ms | **3.3 ms** |
| **Serial p99 latency** | 3.9 ms | 3.6 ms | 186 ms | **4.4 ms** |
| **Peak QPS** | 17,259 (C140) | **33,597** (C120) | 223 (C40) | **2,821** (C20) |
| **Disk reads / query** | — | — | ~1,100 | **193** |

**Headlines**

- **Recall is identical across every tier and every optimization** — **0.8408** for Q8
  in memory, on disk, and after checkpoint/recover. Everything below changed *speed only*.
- **In memory, Q8 is a ~1.95x throughput win** over NOQUANT (33,597 vs 17,259 QPS) at
  equal recall: the 768 B quantized distance computation is far cheaper than 3,072 B FP32.
- **On disk, Q8 now delivers ~12.6x NOQUANT's throughput** (2,821 vs 223 QPS) and
  **~26x lower latency than where Q8-on-disk started** (3.3 ms vs 86 ms).
- **Disk-tiered Q8 latency is now near-memory**: 3.3 ms vs ~2.7 ms. The remaining
  *throughput* gap (2.8K vs 33.6K QPS) is **purely NVMe IOPS**, not software — the device
  runs at **584K IOPS / 100 % utilization** at the plateau.

---

## Test environment

| | |
|---|---|
| **CPU** | 2x Intel Xeon Platinum 8380 — 80 physical cores / **160 threads** |
| **RAM** | 503 GB (Garnet limited to `--memory 64g`) |
| **NVMe** | Dell Ent NVMe P5600 MU U.2 3.2 TB (`/dev/nvme0n1`, mounted `/DATA2`, 512-byte sectors), **single drive** (no RAID) |
| **Kernel** | Linux 6.8.0 (io_uring, incl. unprivileged SQPOLL) |
| **Garnet** | `main` @ `71e34c133`, .NET 10 (SDK 10.0.302), Release build |
| **DiskANN** | `diskann-garnet` **4.0.4** |
| **Dataset** | Cohere `cohere_large_10m` (Wikipedia + Cohere 768-dim), 44 GB checkpoint, COSINE |
| **Client** | VectorDBBench, `Performance768D10M`, co-located on the same host |

Raw FP32 vector = 768 x 4 = **3,072 B** (record ~3,104 B with framing, fetched in a
**single** sector-aligned device read). Q8 quantized vector = **768 B**.

---

## What `Q8` actually means here

`Q8` is Redis-compatible **8-bit scalar quantization** (`VectorQuantType.Q8`). Vectors are
**ingested as FP32** (`VADD <set> FP32 <bytes> <id> Q8 EF <l_build> M <md> ...`) and Garnet
stores **both** representations:

| Representation | Size (768-dim) | Used for | Where it lives when disk-tiered |
|---|--:|---|---|
| **Quantized vector** (8-bit) | 768 B | graph traversal / candidate selection | **memory** (copied back on read) |
| **Full vector** (FP32) | 3,072 B | final **rerank** of the candidate set | **disk** (read on demand) |

This split is exactly why the disk-tiered path issues **~193 reads/query** at
`l_search=192`: traversal is served entirely from memory, and only the FP32 rerank vectors
hit NVMe. Search is `VSIM <set> FP32 <query> COUNT 100 EF 192`, **unfiltered**.

### Per-namespace read-copy (caching) policy

On a disk read, each Vector-Set namespace is either copied back into memory (copy-to-tail on
the main log — or into the read cache when `--read-cache` is enabled) or served from disk
every time. From `libs/server/Resp/Vector/VectorManager.Callbacks.cs`:

| Namespace | Record | Copied back to memory on read? |
|---|---|:--:|
| 0 | **FullVector** (raw FP32, 3,072 B) | NO — **served from disk by design** (rerank only) |
| 1 | NeighborList (adjacency) | YES — cached |
| 2 | QuantizedVector | YES — cached |
| 3 | Attributes | NO — served from disk (read only by *filtered* search) |
| 4 | **Metadata** | YES — cached  <- **this was the bug; see Phase 1** |
| 5 | InternalIdMap | YES — cached |
| 6 | ExternalIdMap | YES — cached |

> **Correction to earlier revisions of this document.** Metadata (ns 4) was previously
> documented as "set-level lifecycle only," never read per query. That was **wrong**, and it
> was the root cause of the original 86 ms Q8-disk result — see
> [Phase 1](#phase-1--the-metadata-discovery).

---

## The Q8 disk-tiered optimization arc

Warm, steady-state, serial (1,000 queries), `l_search=192`, **recall 0.8408 at every step**:

| Phase | Change | Serial avg | Serial p99 | Peak QPS | disk rd/q |
|---|---|--:|--:|--:|--:|
| 0 | Q8 on disk, as originally measured | 86 ms | 184 ms | ~223 | 386 |
| 1 | **Cache the Metadata namespace** (copy-to-tail) | 46.5 ms | 80.7 ms | 1,142 | **193** |
| 2 | **DiskANN 4.0.4** — batched (single multi-read) rerank | 5.0 ms | 6.1 ms | 2,417 | 193 |
| 3 | **io_uring + device IOPS work** (garnet #2018) | 4.1 ms | 5.2 ms | 2,755 | 193 |
| 4 | **+ `--device-uring-sqpoll`** | **3.3 ms** | **4.4 ms** | 2,798 | 193 |
| 4b | + `--device-throttle-limit 512` (tuning check) | 3.3 ms | 4.6 ms | **2,821** | 193 |

**Net: 86 ms -> 3.3 ms (26x) and 223 -> 2,821 QPS (12.6x), recall unchanged at 0.8408.**

### Phase 1 — the Metadata discovery

**86 ms -> 46.5 ms.** Per-namespace read instrumentation (counting vector read-callback invocations by
`NamespaceBytes[0] & 7`, split `DiskLogRecord` vs in-memory `LogRecord`) on a warm pass:

| namespace | disk rd/q | mem rd/q | disk KB/q | avg read |
|---|--:|--:|--:|--:|
| QuantizedVector (2) | 0 | 2,117 | 0 | — (cached) |
| NeighborList (1) | 0 | 215 | 0 | — (cached) |
| ExternalIdMap (6) | 0 | 100 | 0 | — (cached) |
| FullVector (0) | 193 | 0 | 594 | 3,076 B |
| **Metadata (4)** | **193** | 0 | **1,582** | **8,196 B** |
| **TOTAL disk** | **386** | | **2,176** | |

Traversal was already 100 % in memory — the quantization design worked as intended. But
**Metadata, an 8 KB record, was being read once per rerank candidate, uncached, accounting
for 73 % of all disk bytes/query** — more than the vectors themselves.

A cold pass proved it is a small *shared* structure, not per-node data:
`disk=154, mem=193154` (~154 unique records re-read ~193x/query), versus FullVector's
genuinely per-node `disk=193001, mem=0`.

**Fix:** add `DiskANNService.Metadata` to the copy-to-tail `ReadCopyOptions` branch — a
few-line change that removed **73 % of disk bytes/query** for ~1.3 MB of memory. Merged
upstream as **microsoft/garnet#2007**.

### Phase 2 — DiskANN 4.0.4 batched rerank (46.5 -> 5.0 ms)

With Metadata cached, the remaining 193 FullVector reads were still issued **serially per
query**, so the NVMe sat at queue depth ~1-30 while CPU was ~60 % idle — the workload was
*IO-parallelism-bound*, not device-bound. DiskANN **4.0.4** issues the rerank set as a
**single multi-read**, taking serial-pass queue depth to ~24 and latency **46.5 -> 5.0 ms
(~9x)**, and peak QPS to 2,417. Throughput then became **device-IOPS-bound**.

### Phase 3 — device IOPS work (5.0 -> 4.1 ms)

garnet **#2018 "[Storage] Optimize IOPS for RAID-0 NVMe disks"** reworked the native device
(ring sharding, deeper defaults) and shipped new tuning flags. On `main` with plain io_uring
at **default** parameters this gives 4.1 ms / 2,755 QPS — the new defaults match or beat the
previously hand-tuned configuration, so **no tuning is required**.

### Phase 4 — io_uring SQPOLL (4.1 -> 3.3 ms)

A microsecond-level breakdown of the per-read cost on the serial path found **7.43 us/read**,
of which **6.06 us (82 %) was the P/Invoke + native submit**; within that, the
`io_uring_submit` **syscall alone was 5.03 us**. For O_DIRECT NVMe reads the kernel performs
block-layer submission *inline* in `io_uring_enter`, once per read — so this was
CPU-in-syscall time, **not** IO wait (the device-throttle spin measured only 0.17 us,
proving the drive was idle).

`IORING_SETUP_SQPOLL` moves submission to a kernel poller thread, making submits
syscall-free. Enabled with **`--device-uring-sqpoll`** (shipped in #2018; 64 rings -> 64
`iou-sqp` kernel threads):

| | default submit | **SQPOLL** | delta |
|---|--:|--:|--:|
| serial avg | 4.1 ms | **3.3 ms** | **-19 %** |
| serial p99 | 5.2 ms | **4.4 ms** | -15 % |
| QPS @ C5 | 1,198 | 1,301 | +9 % |
| QPS @ C10 | 1,970 | 2,257 | **+15 %** |
| peak QPS | 2,755 | 2,798 | ~0 % (device-bound) |

**SQPOLL is a latency / low-concurrency optimization only.** At the plateau the device is
already saturated, so submission cost is irrelevant; the busy-polling threads even cost a
little p99 at C120-140. That is why the flag is **opt-in**.

---

## Results

### 1. In-memory concurrency sweep (QPS)

| Concurrency | 5 | 10 | 20 | 40 | 60 | 80 | 120 | 140 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **NOQUANT** (recall 0.8382) | 1,368 | 2,539 | 5,472 | 10,890 | 12,927 | 15,075 | 16,774 | **17,259** |
| **Q8** (recall 0.8408) | 1,113 | 2,217 | 4,337 | 11,437 | 15,864 | 20,246 | **33,597** | 32,326 |

NOQUANT peaks at C140 = **17,259 QPS**; Q8 peaks at C120 = **33,597 QPS** (**1.95x**). Below
C20 NOQUANT is slightly ahead — quantization adds a rerank step that only pays off once the
cheaper distance math dominates — after which Q8 pulls away decisively.

### 2. Disk-tiered concurrency sweep (QPS), current `main`

| Concurrency | 5 | 10 | 20 | 40 | 60 | 80 | 120 | 140 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **NOQUANT** (recall 0.8382) | 78 | 132 | 192 | **223** | 219 | 217 | 205 | 196 |
| **Q8**, io_uring default (0.8408) | 1,198 | 1,970 | **2,755** | 2,750 | 2,739 | 2,728 | 2,721 | 2,714 |
| **Q8**, + SQPOLL (0.8408) | 1,301 | 2,257 | **2,798** | 2,780 | 2,767 | 2,757 | 2,749 | 2,739 |
| **Q8**, + SQPOLL + throttle 512 | 1,283 | 2,251 | **2,821** | 2,792 | 2,773 | 2,763 | 2,755 | 2,740 |

Q8 average latency by concurrency (SQPOLL): 3.8 / 4.4 / 7.1 / 14.4 / 21.6 / 29.0 / 43.5 /
51.0 ms.

The Q8 curve **saturates at C20 and then stays flat** out to C140 — the signature of a
device-IOPS ceiling rather than a software collapse. NOQUANT-on-disk is ~12.6x slower
because it must read a 3,072 B FP32 vector *at every hop of the traversal*, not just for
rerank.

### 3. Why `l_search` sets the disk ceiling

With Metadata cached, FullVector rerank reads are the *only* disk cost, so throughput
follows a clean relationship (measured on the Phase-1 build):

> **peak_QPS ~= device_IOPS_ceiling / reads_per_query**, where
> **reads/query = max(k, l_search)**

| l_search | recall | FullVector rd/q | peak QPS | ~ device IOPS |
|--:|--:|--:|--:|--:|
| 32 | 0.7548 | 101 | 2,203 | 222K |
| 64 | 0.7548 | 101 | 2,243 | 227K |
| 128 | 0.7927 | 129 | 1,747 | 225K |
| 192 | 0.8408 | 193 | 1,162 | 224K |
| 256 | 0.8684 | 257 | 859 | 221K |

`peak_QPS x reads/q` is nearly constant, confirming the model. (l_search 32 ~= 64 because
both floor at ~101 reads/q — `k=100` dominates the rerank set.)

---

## Device saturation analysis

`iostat -x` during the C40 Q8 disk sweep on current `main` (steady-state samples):

| metric | value |
|---|--:|
| r/s | **584,226** |
| read bandwidth | 1.96 GB/s |
| avg read size | 3.36 KB (= the FP32 record, single IO) |
| avg queue depth (`aqu-sz`) | 2,390 |
| `r_await` | 4.10 ms |
| `%util` | **100 %** |
| w/s | 0 |

Device ceiling measured independently with `fio` (O_DIRECT, 16 jobs, iodepth 64):

| fio profile | IOPS | Bandwidth |
|---|--:|--:|
| 4 KB random read | **758K** | 3.06 GB/s |
| 3,072 B random read, **4 KB-aligned** (`ba=4096`) | 757K | 2.33 GB/s |
| 3,072 B random read, 512-aligned (= Garnet's access) | **648K** | 1.99 GB/s |

Garnet's 3,072 B vectors land at 512-byte alignment, so each read straddles the device's
4 KB NAND page and the ceiling for *that* access pattern is **~648K IOPS** (a ~15 %
alignment penalty; the same read forced to 4 KB alignment recovers the full 757K).

At **584K IOPS** the workload now sits at **~90 % of that size-and-alignment-matched
ceiling** — up from ~35 % before the optimization arc. Throughput is genuinely
**NVMe-bound**: no software change lifts the plateau, only fewer reads per query or more
devices.

`--device-throttle-limit 512` vs the new default 4096 changed nothing beyond noise
(2,821 vs 2,798 peak), confirming the deeper default is harmless even though it drives
queue depth to ~2,390.

---

## Key conclusions

1. **Recall is invariant** — 0.8408 for Q8 across in-memory, disk-tiered, post-recover, and
   every optimization phase. Tiering and quantization trade speed, not accuracy.
2. **Q8 in memory ~= 2x NOQUANT** (33,597 vs 17,259 QPS) at equal recall.
3. **Q8 disk-tiered latency is now near-memory**: 3.3 ms vs ~2.7 ms in-memory — a 26x
   improvement over the original 86 ms, from four independent fixes (Metadata caching,
   batched rerank, device IOPS work, SQPOLL).
4. **Disk throughput is now genuinely device-bound** at 584K IOPS / 100 % util (~90 % of
   the drive's size-matched ceiling). The path to higher QPS is **fewer reads per query**
   (a shallower rerank depth R << l_search, or an fp16 in-memory rerank surrogate at
   ~15 GB) or **more NVMe devices** (RAID-0, which #2018 explicitly targets) — *not* more
   IO tuning. 4 KB-aligning the stored vectors would also lift the per-read ceiling
   ~648K -> ~758K.
5. **Defaults are good now.** The #2018 device defaults match or beat the previously
   hand-tuned settings; only `--device-uring-sqpoll` is worth opting into, and only for
   latency-sensitive / low-concurrency serving.

### Open question

During the `iostat` window the device served **584,226 r/s** against **2,734.8 QPS**, i.e.
**~214 device reads per query**, versus the **193** logical FullVector reads instrumented at
`l_search=192` (~11 % more). The earlier Phase-1/2 builds tracked ~193 closely. Worth
re-running the per-namespace instrumentation on current `main` to confirm no extra IO crept
in with the newer storage changes (garnet #2062 record framing / #2063 buffer pool are the
likeliest candidates).

---

## Reproduction

Full step-by-step build/load/measure instructions are in
[README.md](./README.md#quantized-in-memory-vs-disk-tiered-benchmark-static-load).
Condensed, assuming an already loaded + checkpointed 10M Q8 index:

```bash
# 1. Start the disk-tiered server, recovering the existing checkpoint
LOGDIR=/DATA2/badrishc/garnet_bench_Q8
dotnet ~/git/garnet/main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --bind 127.0.0.1 --enable-vector-set-preview \
  --index 2g --memory 64g --page 64m \
  --storage-tier --logdir "$LOGDIR" --checkpointdir "$LOGDIR" \
  --device-type Native --device-io-backend Uring \
  --device-uring-sqpoll \
  --enable-debug-command local --recover
# Recovery blocks ~1-2 min; poll DBSIZE until it returns.

# 2. Force disk serving: evict the whole log (Head advances to Tail)
redis-cli DEBUG FLUSHANDEVICT          # -> OK head=<tail> tail=<tail>

# 3. One cold pass to rehydrate the graph stubs (~32 ms/query; discard)
export DATASET_LOCAL_DIR=/path/to/vectordb_dataset
uv run vectordbbench garnet --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
  --skip-drop-old --skip-load --skip-search-concurrent --db-label cold

# 4. Warm serial passes (recall + latency); stable by pass 2-3
for i in 1 2 3; do
  uv run vectordbbench garnet --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
    --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
    --skip-drop-old --skip-load --skip-search-concurrent --db-label warm$i
done

# 5. Concurrency sweep (QPS)
uv run vectordbbench garnet --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
  --skip-drop-old --skip-load --skip-search-serial \
  --num-concurrency 5,10,20,40,60,80,120,140 --concurrency-duration 30 --db-label sweep

# 6. Device view (run alongside step 5)
iostat -x 5 /dev/nvme0n1       # watch r/s, rareq-sz, aqu-sz, r_await, %util
```

**Expected on the hardware above:** cold pass ~32 ms/query; warm serial **3.3 ms avg /
4.4 ms p99**; peak **~2,800 QPS at C20**, flat to C140; **recall 0.8408**; device at
~584K IOPS / 100 % util.

Use `--quantization NOQUANT` with a separate `--logdir` for the NOQUANT comparison, and drop
`--storage-tier` / `FLUSHANDEVICT` for the in-memory numbers.
