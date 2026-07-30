# Running Garnet vector-set benchmarks

This document is a step-by-step guide to start a GarnetServer and run
VectorDBBench against it from the command line (no browser/UI). Two workflows are
covered:

- **Streaming** (`StreamingPerformanceCase`) — insert at a fixed rate while
  continuously measuring search **QPS**, **p99 latency**, and **recall** at
  successive ingestion checkpoints (Steps 1–6 below). This reproduces the workload
  behind the
  [DiskANN + Garnet perf comparison](https://github.com/microsoft/DiskANN/wiki/Perf:-Garnet-Providers-vs-other-Vector-DBs-(Zilliz,-Pinecone,-etc.))
  (Wikipedia-10M + Cohere 768-dim embeddings, 1000 inserts/sec).
- **Static quantized** (`Performance768D10M`) — bulk-load the dataset once, then
  compare **NOQUANT / Q8 / BIN** search served **in memory** vs **from the NVMe
  storage tier** (after eviction). See
  [Quantized in-memory vs disk-tiered benchmark](#quantized-in-memory-vs-disk-tiered-benchmark-static-load).

VectorDBBench's Garnet client connects to an **already-running** GarnetServer —
it does not start one. So there are two processes:

1. **GarnetServer** — the database (from the [Garnet](https://github.com/microsoft/garnet) repo).
2. **vectordbbench** — the benchmark client (this repo), which loads the dataset
   and drives inserts + searches.

> 📊 For measured results — **in‑memory vs disk‑tiered** vector search
> (libaio, O_DIRECT) and NVMe saturation analysis on the 10M Cohere dataset —
> see **[BENCHMARK_RESULTS.md](./BENCHMARK_RESULTS.md)**.

---

## Prerequisites

- **Garnet** checked out and buildable (requires the .NET SDK). This guide assumes
  it is at `~/git/garnet`.
- **uv** for Python dependency management ([install](https://docs.astral.sh/uv/getting-started/installation/)).
- The **Cohere** dataset (downloaded automatically on first run, or pre-staged —
  see Step 3). The 10M variant is ~44 GB.

Install this repo's dependencies including the Garnet client extra:

```bash
cd /path/to/VectorDBBench
uv sync --extra garnet
```

---

## Step 1 — Build GarnetServer

Vector sets are a preview feature, so build the server project that produces the
runnable DLL:

```bash
cd ~/git/garnet
dotnet build main/GarnetServer/GarnetServer.csproj -c Release
```

The runnable artifact is:

```
main/GarnetServer/bin/Release/net10.0/GarnetServer.dll
```

> The target framework folder (`net10.0`) may differ with your SDK version.

---

## Step 2 — Start GarnetServer (all in memory)

For a fully in-memory run, provision the hash index (`--index`) and the hybrid
log (`--memory`) large enough that the entire dataset stays resident (nothing
spills to disk). Vector sets require `--enable-vector-set-preview`.

```bash
cd ~/git/garnet
dotnet main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --bind 127.0.0.1 \
  --enable-vector-set-preview \
  --index 8g \
  --memory 128g \
  --page 64m
```

Flag rationale (for the 10M Cohere 768-dim, NOQUANT/FP32 dataset ≈ 31 GiB of data):

| Flag | Value | Why |
| --- | --- | --- |
| `--enable-vector-set-preview` | — | Required to enable `VADD`/`VSIM` vector-set commands. |
| `--index` | `8g` | Hash-index size. Sized above the working set so it is not the limit. 8 GiB is generous for 10M; `4g` also works. |
| `--memory` | `128g` | Hybrid-log capacity. Must exceed the resident data (~31 GiB) so nothing spills. This is a **cap**, not an upfront allocation. |
| `--page` | `64m` | Log page size. Large pages reduce page count for a big in-memory log. |

Adjust `--memory` / `--index` to your dataset size and available RAM. For the 1M
dataset, `--index 2g --memory 16g` is plenty.

> The default `--value-overflow-threshold` (`16k`) already keeps each 3 KB FP32
> vector inline in the log page, so it does not need to be set here. Only raise it
> if a single value would exceed 16 KB (e.g. FP32 vectors of dimension ≥ 4096).

### Optional: cap virtual memory (VSZ)

GarnetServer runs the .NET **Server GC**, which reserves virtual address space of
roughly **2× physical RAM** (e.g. ~1 TB VSZ on a 503 GB box). This reservation is
uncommitted — it consumes **no physical memory** (RSS is the real footprint) and
is harmless. If a smaller VSZ is desired (e.g. for monitoring/ulimits), cap the
managed heap:

```bash
# 32 GiB managed-heap hard limit (0x800000000 bytes)
DOTNET_GCHeapHardLimit=0x800000000 \
  dotnet main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --enable-vector-set-preview --index 8g --memory 128g --page 64m
```

This caps only the **managed** heap; Garnet's native vector storage (log pages +
index) is unaffected. Leave comfortable headroom (≈24–32 GiB) for network buffers
and per-session state under high concurrency — too tight risks a managed OOM.

### Optional: enable the read cache

For a larger-than-memory run, records spill to disk and a cold query pays a disk
read. The **read cache** keeps a separate, never-flushed, LRU copy of hot on-disk
records in memory so repeated queries stay fast. It is most useful when `--memory`
is smaller than the dataset.

The read cache requires storage tiering (so there is a disk tier to read from):

```bash
dotnet main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --bind 127.0.0.1 --enable-vector-set-preview \
  --index 8g --memory 16g --page 16m \
  --storage-tier --logdir /path/to/garnet-logs \
  --readcache --readcache-memory 8g --readcache-page 16m
```

| Flag | Meaning |
| --- | --- |
| `--storage-tier` | Enable on-disk tiering (records evicted below `Log.HeadAddress` go to `--logdir`). Required for the read cache. |
| `--logdir` | Directory for the on-disk hybrid-log segments. |
| `--readcache` | Enable the read cache. Fails at startup without `--storage-tier`. |
| `--readcache-memory` | Total read-cache size (inline + heap). Does not need to be a power of 2. |
| `--readcache-page` | Read-cache page size (rounds down to a power of 2; min 512). |

Confirm it is active and being populated via `INFO STORE` (see below): the
`ReadCache.*` fields appear (not `N/A`), and `ReadCache.TailAddress` grows past
`64` as on-disk records are read and cached.

---

## Step 3 — (Optional) Pre-stage the dataset

By default datasets download from a public S3 bucket on first use into
`/tmp/vectordb_bench/dataset`. To use a pre-downloaded copy, point
`DATASET_LOCAL_DIR` at the directory that contains the `cohere/` folder:

```bash
export DATASET_LOCAL_DIR=/path/to/vectordb_dataset
# expects: $DATASET_LOCAL_DIR/cohere/cohere_large_10m/...
```

---

## Step 4 — Run the streaming benchmark

With GarnetServer running, start the benchmark from this repo. The example below
reproduces the
[DiskANN wiki](https://github.com/microsoft/DiskANN/wiki/Perf:-Garnet-Providers-vs-other-Vector-DBs-(Zilliz,-Pinecone,-etc.))
configuration: Wikipedia-10M + Cohere (768-dim), inserting 1000 vectors/sec while
searching at every 10% of ingestion across concurrency levels 5/10/20/60.

```bash
cd /path/to/VectorDBBench
DATASET_LOCAL_DIR=/path/to/vectordb_dataset \
uv run vectordbbench garnet \
  --case-type StreamingPerformanceCase \
  --dataset-with-size-type "Large Cohere (768dim, 10M)" \
  --insert-rate 1000 \
  --search-stages 0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9 \
  --concurrencies 5,10,20,60 \
  --max-degree 16 \
  --l-build 128 \
  --l-search 128 \
  --k 100 \
  --host 127.0.0.1 \
  --port 6379 \
  --db-label stream-cohere10m
```

`--search-stages` and `--concurrencies` above are also the defaults, so they can be
omitted for a wiki-matching run. Drop `DATASET_LOCAL_DIR` to download to the
default location. The first run with `--drop-old` (the default) clears the `vs0`
collection on the server first.

> **Tip:** add `--dry-run` to print the resolved task config and exit without
> running — useful to confirm flags before a multi-hour run.

### Matching the wiki configuration

The wiki ("DiskANN3 + Garnet Providers v1.0.26") used Zilliz's VectorDBBench with
its standard streaming settings on an Azure D32v6 VM. The reproduction settings:

| Setting | Value | Source |
| --- | --- | --- |
| Dataset | Cohere 768-dim, 10M (Wikipedia + Cohere embeddings) | wiki text |
| Distance metric | COSINE | set automatically from the Cohere dataset |
| `--insert-rate` | `1000` rows/sec | wiki text ("inserts 1000 vector per sec") |
| `--search-stages` | `0.1 … 0.9` (every 10%) | VectorDBBench streaming default; matches the graph x-axis (a point per 10%) |
| `--concurrencies` | `5,10,20,60` | wiki uses `5,10,20`; `60` is added here to exercise higher concurrency on large (many-core) hosts |
| `--optimize-after-write` | on | default; the graphs' dashed "110%" point is the post-optimize search |
| `--read-dur-after-write` | `30` s | default |
| `--k` | `100` | default |

The graphs plot, per stage, the **max QPS** over the concurrency sweep, the serial
**p99 latency**, and an **adjusted recall** (`raw_recall / fraction_inserted`).

> The graph index parameters (`--max-degree`, `--l-build`, `--l-search`) are not
> stated in the wiki; they are recall/QPS tuning knobs (see [Tuning notes](#tuning-notes)).
> Absolute QPS also depends on hardware and the concurrency sweep, so it will differ
> from the wiki's 32-vCPU VM.

### Streaming options

| Flag | Default | Meaning |
| --- | --- | --- |
| `--insert-rate` | `500` | Background insert rate in rows/sec (rounded down to a multiple of `NUM_PER_BATCH`). |
| `--search-stages` | `0.1,0.2,…,0.9` | Insert ratios at which to run a search round. |
| `--concurrencies` | `5,10,20,60` | Search concurrency levels swept at each stage. |
| `--optimize-after-write` / `--skip-optimize-after-write` | on | Optimize the index and run a final search after all data is inserted. |
| `--read-dur-after-write` | `30` | Duration (s) of the final search run after all inserts complete. |
| `--dataset-with-size-type` | `Medium Cohere (768dim, 1M)` | Dataset + size. For 10M use `Large Cohere (768dim, 10M)`. |

Available datasets for `--dataset-with-size-type`: `Medium Cohere (768dim, 1M)`,
`Large Cohere (768dim, 10M)`, `Medium Bioasq (1024dim, 1M)`,
`Large Bioasq (1024dim, 10M)`, `Large OpenAI (1536dim, 5M)`,
`Medium OpenAI (1536dim, 500K)`.


### Garnet index/search options

| Flag | Default | Meaning |
| --- | --- | --- |
| `--max-degree` | *required* | Graph degree (`M`) used at build time. |
| `--l-build` | `128` | Build-time search-list size (`EF` on `VADD`). |
| `--l-search` | `15` | Query-time search-list size (`EF` on `VSIM`). Higher → better recall, lower QPS. |
| `--quantization` | `NOQUANT` | Vector quantization: `NOQUANT` (raw FP32), `Q8` (8-bit), `BIN` (1-bit). `Q8`/`BIN` keep the raw FP32 vector for reranking and cache the small quantized vector in memory for graph traversal. See [the quantized benchmark](#quantized-in-memory-vs-disk-tiered-benchmark-static-load). |
| `--filter-scale` | `16` | Adaptive filter scale factor (filtered search). |
| `--host` / `--port` | `127.0.0.1` / `6379` | GarnetServer address. |
| `--username` / `--password` | — | Optional auth. |

Load throughput is set by `--load-concurrency` (a common VectorDBBench flag, not
Garnet-specific): the number of parallel insert workers. Default is `min(cpu, 4)`;
raising it to `32` roughly 6× the load rate on a many-core host (concurrent `VADD`
to one set scales in memory — see the quantized benchmark's load step).

---

## Step 5 — Monitor progress

The 10M @ 1000/sec run inserts for ~2.8 hours (insertion time is inherent to the
rate). Watch the benchmark log, and confirm Garnet ingestion via `INFO STORE`
using any RESP client (Garnet is Redis-protocol compatible):

```bash
# Insertion progress: Log.TailAddress grows ~1000 vectors/sec
redis-cli -p 6379 INFO STORE | grep -E "Log.TailAddress|Log.HeadAddress|IndexTotalMemorySizeBytes"
```

If `redis-cli` is not installed, any Redis client works, e.g.:

```bash
uv run python -c "import redis; r=redis.Redis(port=6379, protocol=2).execute_command('INFO','STORE'); print('TailAddress', r['Log.TailAddress'], 'HeadAddress', r['Log.HeadAddress'])"
```

- `Log.TailAddress` increasing ≈ data being written.
- `Log.HeadAddress` staying at `64` confirms a **pure in-memory** run (nothing
  evicted to disk).

The runner logs each stage, e.g.:

```
Serial search - 50% done, recall=0.4106, p99=0.005, p95=0.0042
End search in concurrency 20: ... qps=..., p99=0.005s
```

---

## Step 6 — Results

A JSON result file is written to:

```
vectordb_bench/results/Garnet/result_<date>_<id>_garnet.json
```

It contains per-stage lists: `st_search_stage_list`, `st_max_qps_list_list`,
`st_recall_list`, `st_serial_latency_p99_list`, etc.

> **Recall note:** ground truth is the nearest neighbors over the *full* dataset,
> so per-stage recall ramps up as more data is inserted and converges to the true
> value at the 100% end-of-stream search. VectorDBBench's UI shows an
> "adjusted recall" = `raw_recall / fraction_inserted`.

---

## Quantized in-memory vs disk-tiered benchmark (static load)

This is the workflow for the **NOQUANT / Q8 / BIN** comparison and the
in-memory-vs-disk study: bulk-load the whole dataset once, then measure search
served **fully in memory** and **from the NVMe storage tier** (after the raw
vectors are evicted). Unlike the streaming case it uses the static
`Performance768D10M` case (bulk load, then a serial + concurrency search sweep).

### Quantization modes

`--quantization` selects how each vector is stored and used during traversal:

| Mode | Graph-traversal vector | Raw FP32 (for rerank) | In-graph memory (768-dim) |
| --- | --- | --- | --- |
| `NOQUANT` | full FP32 (3072 B) | — (traversal uses it) | 3072 B |
| `Q8` | 8-bit quantized (768 B) | kept | 768 B |
| `BIN` | 1-bit (96 B) | kept | 96 B |

Q8/BIN keep the compact quantized vector memory-resident for graph traversal and
keep the raw FP32 for final reranking. Under storage tiering, Garnet's read policy
**copies the quantized + adjacency "stub" records back into memory** (main-log
tail; no read cache required) when they are read from disk, while the raw FP32 is
served from disk. So a disk-tiered Q8/BIN graph traverses on memory-resident
quantized vectors and only reads raw FP32 from disk for reranking.

**What to expect** (10M Cohere, `M=16, l_build=300, l_search=192, k=100`, on the
hardware in BENCHMARK_RESULTS.md): in memory, Q8 roughly **doubles** peak QPS vs
NOQUANT at equal recall (cheaper 768 B distance computations). Served from disk,
recall is preserved and the quantized vectors are confirmed resident, but per-query
latency is currently dominated by the raw-FP32 disk reads — see the *reads/query*
step below.

### Step A — Start a disk-tiered server (native O_DIRECT)

The raw vectors must be able to spill to the NVMe device, so start with storage
tiering and the Linux **native O_DIRECT** device (bypasses the OS page cache).
`--enable-debug-command local` is required for `DEBUG FLUSHANDEVICT`. Use a
**separate `--logdir`/`--checkpointdir` per quantization mode** to keep them
isolated.

```bash
LOGDIR=/path/to/garnet_bench_Q8            # per-mode directory on the NVMe
dotnet ~/git/garnet/main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --bind 127.0.0.1 --enable-vector-set-preview \
  --index 2g --memory 64g --page 64m \
  --storage-tier --logdir "$LOGDIR" --checkpointdir "$LOGDIR" \
  --device-type Native --device-io-backend Libaio --device-completion-threads 16 \
  --enable-debug-command local
```

`--memory 64g` holds the whole 10M dataset (raw FP32 ~31 GiB, + Q8 quantized
~7.5 GiB, + graph) **in memory during load**, so nothing spills mid-load (a
concurrent bulk load into a spilling set is not safe). Eviction to disk happens
explicitly in Step C. `--index 2g` gives headroom for the extra quantized-vector
keys Q8/BIN add.

| Flag | Why |
| --- | --- |
| `--storage-tier` + `--logdir` | Enable the on-disk tier the raw vectors are evicted to. |
| `--device-type Native` | Linux O_DIRECT device — bypasses the OS page cache so disk numbers are real. |
| `--device-io-backend Libaio` | libaio backend (`Uring` also available). |
| `--device-completion-threads 16` | IO-completion threads for the device. |
| `--enable-debug-command local` | Enables `DEBUG FLUSHANDEVICT` (loopback only). |

### Step B — Load (32 insert workers)

Bulk-load with `--load-concurrency 32` (concurrent `VADD` to one set scales to
~13–37 cores in memory; 32 workers ≈ 6× the default-4 rate → ~20–35 min for 10M).
`--quantization` is set here at build time. Load only (skip search):

```bash
export DATASET_LOCAL_DIR=/path/to/vectordb_dataset      # pre-staged 10M (Step 3)
uv run vectordbbench garnet \
  --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 \
  --quantization Q8 --load-concurrency 32 \
  --skip-search-serial --skip-search-concurrent \
  --db-label Q8_load
```

Watch load progress with `INFO STORE` (`Log.TailAddress` growing; `HeadAddress`
stays `64` = in-memory). Run once per mode (`NOQUANT`, `Q8`, `BIN`) into its own
`--logdir`.

### Step C — Measure in memory

Search only (`--skip-drop-old --skip-load`), serial (recall + p99) plus a
concurrency sweep (QPS):

```bash
uv run vectordbbench garnet \
  --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
  --skip-drop-old --skip-load \
  --num-concurrency 5,10,20,40,60,80,120,140 --concurrency-duration 30 \
  --db-label Q8_inmem
```

### Step D — Evict to disk, and verify the quantized vectors come back to memory

Evict the log to the NVMe device, then confirm the quantized/adjacency stubs are
copied **back** into the main-log tail as queries touch them. The resident bytes
are `Log.TailAddress − Log.HeadAddress` (the raw FP32 stays below `HeadAddress` on
disk):

```bash
info() { uv run python -c "import redis; i=redis.Redis(port=6379,protocol=2,socket_timeout=120).execute_command('INFO','STORE'); h=int(i['Log.HeadAddress']); t=int(i['Log.TailAddress']); print('resident(Tail-Head)=%.2fGB Head=%.2fGB Tail=%.2fGB' % ((t-h)/1e9,h/1e9,t/1e9))"; }

# Evict everything to disk (Head advances to Tail)
uv run python -c "import redis; print(redis.Redis(port=6379,protocol=2,socket_timeout=120).execute_command('DEBUG','FLUSHANDEVICT'))"
info                       # right after evict: resident ≈ 0.00 GB
```

Then run a couple of **serial warm passes** and re-check `info` after each — the
resident set grows from ~0 and stabilizes (e.g. ~1.3 GB for the 10M query set)
as the quantized + adjacency stubs return to memory:

```bash
for p in 1 2 3; do
  uv run vectordbbench garnet --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
    --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
    --skip-drop-old --skip-load --skip-search-concurrent --db-label Q8_warm$p
  info                     # resident(Tail-Head) grows, then stabilizes
done
```

### Step E — Measure from disk

Once resident bytes have stabilized (cache warm), run the full serial +
concurrency sweep exactly as Step C (add `iostat` if you want the device view):

```bash
uv run vectordbbench garnet \
  --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
  --skip-drop-old --skip-load \
  --num-concurrency 5,10,20,40,60,80,120,140 --concurrency-duration 30 \
  --db-label Q8_disk
```

Recall matches the in-memory run (eviction changes *speed*, not *which* neighbors
are found).

### Step F — Disk reads per query (iostat)

To see how much raw-FP32 disk traffic each query drives, run `iostat` alongside a
single **serial** pass and integrate the read count over the pass:

```bash
iostat -x 2 100 /dev/nvme0n1 > /tmp/io.log 2>&1 &          # auto-stops after 200 s
uv run vectordbbench garnet --case-type Performance768D10M --host 127.0.0.1 --port 6379 \
  --max-degree 16 --l-build 300 --l-search 192 --k 100 --quantization Q8 \
  --skip-drop-old --skip-load --skip-search-concurrent --db-label Q8_io
# reads/query = sum(r/s * 2s) over the active window / 1000 queries:
python3 -c "print('reads/query ~ %.0f' % (sum(float(l.split()[1])*2 for l in open('/tmp/io.log') if 'nvme0n1' in l and float(l.split()[1])>50)/1000))"
```

`r/s` is column 2 of `iostat -x`; `rareq-sz` (column 7) is the average read size.
`reads/query` scales with `--l-search` (the candidate/visited set), which is the
main knob for the disk read volume.

### Step G — Checkpoint & recover (skip the reload)

Loading 10M takes ~20–35 min, so checkpoint a loaded/evicted graph and recover it
later (also validates recovery parity). Take a foreground checkpoint (`SAVE`
blocks until durable), then restart with `--recover` pointing at the same
directory:

```bash
uv run python -c "import redis; print(redis.Redis(port=6379,protocol=2,socket_timeout=3600).execute_command('SAVE'))"
# ... later, same flags as Step A PLUS --recover:
dotnet ~/git/garnet/main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --bind 127.0.0.1 --enable-vector-set-preview \
  --index 2g --memory 64g --page 64m \
  --storage-tier --logdir "$LOGDIR" --checkpointdir "$LOGDIR" \
  --device-type Native --device-io-backend Libaio --device-completion-threads 16 \
  --enable-debug-command local --recover
```

Recovery restores the graph + native index (no ~20–35 min rebuild). It is
CPU-busy for a few minutes finalizing the index; `INFO STORE` may block until it
settles. Recall after recover equals the pre-checkpoint value.

---

## Inspecting store state with INFO

Garnet exposes store internals over the RESP `INFO` command (Redis-protocol
compatible). Use any client; the examples use a Python one-liner.

### `INFO STORE` — log addresses

```bash
uv run python -c "import redis; i=redis.Redis(port=6379, protocol=2).execute_command('INFO','STORE'); print('\n'.join(f'{k}={i[k]}' for k in i if k.startswith(('Log.','ReadCache.'))))"
```

The hybrid-log addresses describe where data lives (addresses are byte offsets; an
empty store starts at `64`):

| Field | Meaning |
| --- | --- |
| `Log.BeginAddress` | Oldest valid address. Advances when the log head is truncated. |
| `Log.HeadAddress` | Below this, records are **on disk** (evicted). Equal to `BeginAddress` means nothing has been evicted. |
| `Log.SafeReadOnlyAddress` | Boundary between the mutable region (above) and the read-only/immutable region (below). |
| `Log.FlushedUntilAddress` | How far the log has been flushed to the disk tier. |
| `Log.TailAddress` | Current append point ≈ total in-log data size. |
| `ReadCache.*` | Same addresses for the read cache, or `N/A` when `--readcache` is off. |

**Expected addresses:**

- **Pure in-memory run:** `Log.HeadAddress == Log.BeginAddress == 64` (nothing
  evicted to disk) and `Log.TailAddress` ≈ the data size. This is the check used in
  the monitoring step above.
- **Tiered run (`--storage-tier`):** `Log.HeadAddress > Log.BeginAddress` once the
  log exceeds `--memory` and older records spill to disk.
- **Read cache active:** `ReadCache.TailAddress > 64` and grows as on-disk records
  are read and copied into the cache.

### `INFO STOREHASHTABLE` — hash-table distribution

This dumps the main-store hash-index distribution (the equivalent of Tsavorite's
`DumpDistribution`). It scans the whole index, so it is **expensive and not
returned by default** — request it explicitly:

```bash
uv run python -c "import redis; print(redis.Redis(port=6379, protocol=2).execute_command('INFO','STOREHASHTABLE'))"
```

Key fields for judging index health:

- `Number of hash buckets` / `Size of each bucket` — index capacity (set by `--index`).
- `Total distinct hash-table entry count` — number of occupied entries.
- `Average #entries per hash bucket` — load factor; well below 1 means the index is
  generously sized for the data.
- `Histogram of #entries per bucket` — the collision distribution. A healthy,
  well-sized index is dominated by buckets with `0` or `1` entries with a small tail;
  many buckets with high entry counts indicate collisions / an undersized `--index`.
- `Total entries in overflow buckets` — non-zero means buckets overflowed (raise
  `--index` if this grows large).

---

## Tuning notes

- **Recall vs. QPS** is governed by `--l-search` (no rebuild required): higher
  `l-search` raises recall and lowers QPS. As a rough guide on 10M Cohere with
  `--max-degree 16`: `l-search 128` → recall ≈ 0.77; `500` → ≈ 0.90; `800` → ≈ 0.92.
  A higher `--max-degree` shifts the whole frontier up but requires a rebuild.
- **Memory footprint** (10M, NOQUANT/FP32): ~31 GiB data log + the provisioned
  index ≈ ~39 GB RSS. **Quantizing** (`--quantization Q8` or `BIN`) keeps a small
  quantized vector (768 B for Q8, 96 B for BIN) memory-resident for traversal while
  the raw FP32 stays available for reranking, and roughly doubles in-memory search
  throughput at equal recall (see
  [the quantized benchmark](#quantized-in-memory-vs-disk-tiered-benchmark-static-load)).
- **Smaller smoke test:** swap `--dataset-with-size-type "Medium Cohere (768dim, 1M)"`,
  `--memory 16g --index 2g`, and lower `--search-stages` for a quick end-to-end
  validation.

---

## Revivification: reusing log space after dropping a set

Garnet's hybrid log is append-only, so `DEL <set>` (drop) tombstones the records
but does not reclaim their space in memory — a subsequent reload appends on top,
doubling `Log.TailAddress`. [Revivification](https://github.com/microsoft/garnet/blob/main/website/docs/dev/tsavorite/reviv.md)
reuses those tombstoned records' space, so an insert→drop→insert cycle keeps the
tail roughly flat instead of growing.

Vector records share the main store, and dropping a vector set background-deletes
every underlying record, so they feed the revivification free list. The default
`--reviv` bins (256 records each) are far too small for millions of vector
records; use **custom bins** sized to the records, with large counts:

```bash
dotnet main/GarnetServer/bin/Release/net10.0/GarnetServer.dll \
  --port 6379 --enable-vector-set-preview --index 8g --memory 128g --page 64m \
  --reviv-bin-record-sizes 64,128,256,512,1024,2048,4096,8192 \
  --reviv-bin-record-counts 12000000
```

- Bins span the vector record sizes: ~3 KB `FP32` full vectors land in the 4096
  bin; adjacency lists, id maps, and attributes use the smaller bins.
- The count must cover the records freed by a drop (~5 records/vector across
  types). Use ~12,000,000 for 10M vectors; scale down for smaller sets.
- Each free-list slot is 8 bytes, so 8 bins × 12M ≈ 768 MB of overhead.
- Drop cleanup is asynchronous; wait for it to finish before reloading.

Verify reuse via `Log.TailAddress`: load → drop → reload should grow it by a few
percent (records below `Log.SafeReadOnlyAddress` stay immutable and are not
revivified), versus ~2× without revivification.

| Test | Tail after load 1 | After drop | After load 2 | reuse |
| --- | --- | --- | --- | --- |
| 1M, reviv on | 3.06 GiB | 3.07 GiB | 3.18 GiB | ~96% |
| 1M, reviv off | 3.06 GiB | 3.07 GiB | 6.13 GiB | 0% (doubled) |

