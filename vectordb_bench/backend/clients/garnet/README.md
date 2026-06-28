# Running the Garnet streaming benchmark

This document is a step-by-step guide to start a GarnetServer and run the
VectorDBBench **streaming** benchmark against it from the command line (no
browser/UI).

The streaming case (`StreamingPerformanceCase`) inserts vectors at a fixed rate
in the background while continuously measuring search **QPS**, **p99 latency**,
and **recall** at successive ingestion checkpoints. This reproduces the workload
behind the
[DiskANN + Garnet perf comparison](https://github.com/microsoft/DiskANN/wiki/Perf:-Garnet-Providers-vs-other-Vector-DBs-(Zilliz,-Pinecone,-etc.))
(Wikipedia-10M + Cohere 768-dim embeddings, 1000 inserts/sec).

VectorDBBench's Garnet client connects to an **already-running** GarnetServer —
it does not start one. So there are two processes:

1. **GarnetServer** — the database (from the [Garnet](https://github.com/microsoft/garnet) repo).
2. **vectordbbench** — the benchmark client (this repo), which loads the dataset
   and drives inserts + searches.

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
  --page 64m \
  --value-overflow-threshold 16k
```

Flag rationale (for the 10M Cohere 768-dim, NOQUANT/FP32 dataset ≈ 31 GiB of data):

| Flag | Value | Why |
| --- | --- | --- |
| `--enable-vector-set-preview` | — | Required to enable `VADD`/`VSIM` vector-set commands. |
| `--index` | `8g` | Hash-index size. Sized above the working set so it is not the limit. 8 GiB is generous for 10M; `4g` also works. |
| `--memory` | `128g` | Hybrid-log capacity. Must exceed the resident data (~31 GiB) so nothing spills. This is a **cap**, not an upfront allocation. |
| `--page` | `64m` | Log page size. Large pages reduce page count for a big in-memory log. |
| `--value-overflow-threshold` | `16k` | Keep the 3 KB FP32 vectors inline in the log record (no overflow allocations). |

Adjust `--memory` / `--index` to your dataset size and available RAM. For the 1M
dataset, `--index 2g --memory 16g` is plenty.

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
  --port 6379 --enable-vector-set-preview --index 8g --memory 128g --page 64m --value-overflow-threshold 16k
```

This caps only the **managed** heap; Garnet's native vector storage (log pages +
index) is unaffected. Leave comfortable headroom (≈24–32 GiB) for network buffers
and per-session state under high concurrency — too tight risks a managed OOM.

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
searching at every 10% of ingestion across concurrency levels 5/10/20.

```bash
cd /path/to/VectorDBBench
DATASET_LOCAL_DIR=/path/to/vectordb_dataset \
uv run vectordbbench garnet \
  --case-type StreamingPerformanceCase \
  --dataset-with-size-type "Large Cohere (768dim, 10M)" \
  --insert-rate 1000 \
  --search-stages 0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9 \
  --concurrencies 5,10,20 \
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
| `--concurrencies` | `5,10,20` | VectorDBBench streaming default |
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
| `--concurrencies` | `5,10,20` | Search concurrency levels swept at each stage. |
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
| `--filter-scale` | `16` | Adaptive filter scale factor (filtered search). |
| `--host` / `--port` | `127.0.0.1` / `6379` | GarnetServer address. |
| `--username` / `--password` | — | Optional auth. |

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

## Tuning notes

- **Recall vs. QPS** is governed by `--l-search` (no rebuild required): higher
  `l-search` raises recall and lowers QPS. As a rough guide on 10M Cohere with
  `--max-degree 16`: `l-search 128` → recall ≈ 0.77; `500` → ≈ 0.90; `800` → ≈ 0.92.
  A higher `--max-degree` shifts the whole frontier up but requires a rebuild.
- **Memory footprint** (10M, NOQUANT/FP32): ~31 GiB data log + the provisioned
  index ≈ ~39 GB RSS. Quantizing (the Garnet client currently uses `NOQUANT`)
  would substantially reduce the per-vector cost.
- **Smaller smoke test:** swap `--dataset-with-size-type "Medium Cohere (768dim, 1M)"`,
  `--memory 16g --index 2g`, and lower `--search-stages` for a quick end-to-end
  validation.
