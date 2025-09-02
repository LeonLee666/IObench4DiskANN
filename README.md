## I/O Efficiency in Graph-Based Disk-Resident Approximate Nearest Neighbor Search — Benchmark and Reproduction Guide

This repository provides a consolidated, research-oriented benchmark to reproduce and extend the empirical study of I/O efficiency in disk-resident graph-based approximate nearest neighbor (ANN) search, as explored in the paper “**I/O Efficiency in Graph-Based Disk-Resident Approximate Nearest Neighbor Search: A Design Space Exploration.**” The benchmark organizes dataset layouts, evaluation metrics, and links to implementations spanning multiple codebases.

The primary goals are: (i) to facilitate fair, repeatable comparisons across representative systems; (ii) to expose I/O-level trade-offs alongside algorithmic parameters (e.g., graph degree, beam width, product quantization settings); and (iii) to document a reproducible workflow consistent with academic standards.

### External Implementations (Evaluated Systems)

- DiskANN (baseline/fork): [github.com/LeonLee666/DiskANN](https://github.com/LeonLee666/DiskANN)
- Starling: [github.com/LeonLee666/starling](https://github.com/LeonLee666/starling)
- PipeANN: [github.com/LeonLee666/PipeANN](https://github.com/LeonLee666/PipeANN)
- AISaQ-DiskANN: [github.com/KioxiaAmerica/aisaq-diskann](https://github.com/KioxiaAmerica/aisaq-diskann)

Please consult each repository’s README for build and runtime specifics. This benchmark serves as the methodological and organizational layer, unifying evaluation scenarios across the systems above.

## Obtaining Implementations (Suggested Layout)

Clone this meta-repository and the four systems into a common workspace. A suggested layout is:

```
IObench4DiskANN/
  external/
    DiskANN/
    starling/
    PipeANN/
    aisaq-diskann/
  datasets/
    sift
    deep
    spacev
```

Example cloning (adapt as needed):

```bash
cd external
git clone --recursive https://github.com/LeonLee666/DiskANN.git DiskANN
git clone --recursive https://github.com/LeonLee666/starling.git starling
git clone --recursive https://github.com/LeonLee666/PipeANN.git PipeANN
git clone --recursive https://github.com/KioxiaAmerica/aisaq-diskann.git aisaq-diskann
```

Follow each repository’s build and installation instructions. Prefer out-of-source builds and record compiler flags and link options.

## Datasets

This benchmark supports commonly used ANN datasets. Recommended sources include:

- SIFT1M and SIFT100M
- DEEP100M
- SPACEV100M

Please download the datasets and organize them according to the directory structure above. Download links:
- SIFT1M and SIFT100M: http://corpus-texmex.irisa.fr/
- DEEP100M: https://research.yandex.com/blog/benchmarks-for-billion-scale-similarity-search
- SPACEV100M: https://github.com/microsoft/SPTAG/tree/main/datasets/SPACEV1B

NOTE that: SIFT1M has been included in this repo and can be pulled by: `git lfs pull`

Dataset statistics and canonical index build parameters (B and M are memory budgets in GB):

| Dataset    | Dim | Type  | #Vec | #Q  | R  | L   | α   | B (GB) | M (GB) |
|------------|-----|-------|------|-----|----|-----|-----|--------|--------|
| SIFT1M     | 128 | float | 1M   | 10K | 64 | 125 | 1.2 | 0.02   | 50     |
| SIFT100M   | 128 | uint8 | 100M | 10K | 64 | 125 | 1.2 | 4.0    | 50     |
| DEEP100M   | 96  | float | 100M | 10K | 64 | 125 | 1.2 | 4.0    | 50     |
| SPACEV100M | 100 | uint8 | 100M | 29K | 64 | 125 | 1.2 | 4.0    | 50     |


### Metrics

- Recall@10, QPS/throughput, mean latency.
- IOPS, read bandwidth, and device utilization from OS/driver counters.
- CPU utilization, resident memory.

### Measurement Guidance

- Warm-up runs to stabilize caches and device state.
- Multiple independent trials; report mean and standard deviation.
- Pin processes/threads (NUMA-aware placement where relevant).
- Flush/prime caches consistently, document whether OS page cache is used or disabled.


## Optimization Primitives (Single-Factor I/O Methods)

This benchmark evaluates individual I/O‑centric optimizations in isolation to quantify their standalone effects, following the taxonomy in the accompanying paper. The primitives and their primary intent are:

- Memory layout:
  - PQ: compress vectors to move early filtering into memory and reduce disk reads.
  - Cache: retain hot vertices near entry points; accelerate early hops.
  - MemGraph: an in‑memory navigator providing high‑quality entry points to shorten paths.
- Disk layout:
  - PageShuffle (PS): co‑locate graph neighbors to raise page‑level utility (overlap ratio).
  - All-in-Storage (AiS/AiSAQ): place PQ on SSD alongside full vectors to minimize memory.
- Search algorithm:
  - DynamicWidth (DW): adapt beam width across approach and convergence phases.
  - Pipeline: overlap I/O and compute; continuous pipelining to raise bandwidth utilization.
  - PageSearch (PSe): score all records in a fetched page to exploit expensive I/Os.

When reported alone, each method is toggled against an identical baseline configuration at a matched Recall@k target. For interpretability, we recommend reporting per‑primitive deltas on: I/O per query, QPS, mean latency, and device IOPS/BW.

## Compositional Strategies (Multi-Factor Combinations)

We benchmark complementary combinations that target orthogonal drivers—raising per‑page utility and shortening search paths—reflecting the paper’s findings:

- C1 = PS + PSe (on PQ baseline): improve inter‑page locality and intra‑page utilization.
- C2 = Pipeline + DW: mitigate speculative I/O of pipeline with adaptive width control.
- C3 = MemGraph + PS + PSe: add high‑quality entries to C1 to cut path length.
- C4 = MemGraph + Pipeline + DW: navigator plus bandwidth‑oriented execution.
- C5 = MemGraph + PS + PSe + DW (OctopusANN): combined path‑shortening and page‑utility.

Report both accuracy–throughput and latency–recall curves at matched Recall@k. Highlight resource overhead (disk, memory) and device utilization to expose potential read amplification or contention.

## Experimental Design for Single vs. Combined Methods

- Baseline: PQ with fixed budgets; identical datasets and recall targets across all runs.
- Controls: identical threading, async I/O engine, page size, and logging cadence.
- Isolation: change one factor at a time when ablating; add minimal deltas when composing.
- Metrics: I/Os per query, QPS, mean/p95/p99, IOPS, bandwidth, memory/disk footprint.
- Stability: 3–5 independent trials; warm‑up; pin affinity; consistent cache policy.

**Practical notes from the paper:**
- **Single‑factor: MemGraph and DynamicWidth show strong standalone gains; PageShuffle and PageSearch are weak alone but synergize; Pipeline can regress under concurrency; AiS trades memory for 10×+ disk footprint.**
- **Compositions: PS+PSe are complementary; adding MemGraph amplifies benefits; DW complements Pipeline but tends to dominate under concurrency. The best overall under our settings is C5 (OctopusANN).**

## Dependencies

This benchmark has been tested on Ubuntu 22.04. Run the following commands before running experiments.

```bash
sudo apt install build-essential make cmake g++ clang-format
sudo apt install libaio-dev liburing-dev
sudo apt install libgoogle-perftools-dev libboost-all-dev
sudo apt install libmkl-full-dev

# increase AIO limit (if needed)
echo 'fs.aio-max-nr=1048576' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

## How to Run the Benchmarks

This section demonstrates how to reproduce single‑factor optimizations and their compositions on SIFT1M, aligned with the paper’s taxonomy. Each subsection points to the corresponding repository and branch. Commands are provided verbatim, with minimal branch checkout steps where helpful.



### NoOPT (baseline)
DiskANN (baseline/fork): [github.com/LeonLee666/DiskANN](https://github.com/LeonLee666/DiskANN)

Use the no‑pq branch to build a DiskANN variant without PQ/Cache, serving as the NoOPT baseline.

```bash
git checkout no-pq
cd build && cmake -DCMAKE_BUILD_TYPE=Release .. && make -j 
```

Build the index and run NoOPT on SIFT1M:
```bash
bash ./sift1m_build.sh
bash ./sift1m_no_opt.sh
```

### PQ/Cache/MemGraph

Starling: [github.com/LeonLee666/starling](https://github.com/LeonLee666/starling) (main branch) supports PQ, Cache, and MemGraph.

```bash
git checkout main
# compile
./run_benchmark.sh release
# build disk index
./run_benchmark.sh release build
# build MemGraph index
./run_benchmark.sh release build_mem
# search (configure CACHE and MEM_L in config_params.sh to toggle Cache/MemGraph)
./run_benchmark.sh release search knn
```

### PageShuffle and PageSearch (Single‑Factor)
Starling: [github.com/LeonLee666/starling](https://github.com/LeonLee666/starling) (individual-opt branch).

```bash
git checkout individual-opt
```

To evaluate PageSearch:
```bash
# enable USE_PAGE_SEARCH=1
# disable Cache and MemGraph
vi config_params.sh
# run search
./run_benchmark.sh release search knn
```

To evaluate PageShuffle:
```bash
# graph partition and shuffling page
./run_benchmark.sh release gp
# ensure USE_PAGE_SEARCH=0
# disable Cache and MemGraph
vi config_params.sh
# run search
./run_benchmark.sh release search knn
```


#### Compositional: PageShuffle + PageSearch (± MemGraph)

Switch to Starling main to evaluate the compositions PS+PSe and PS+PSe+MemGraph.
```bash
git checkout main
# enable USE_PAGE_SEARCH=1
# with Cache and MemGraph disabled
vi config_params.sh
./run_benchmark.sh release search knn
# then enable MemGraph
vi config_params.sh
./run_benchmark.sh release search knn
```

#### OctopusANN
Starling: [github.com/LeonLee666/starling](https://github.com/LeonLee666/starling) (dyn-page branch).

```bash
git checkout dyn-page
# enable USE_PAGE_SEARCH=1
# enable MemGraph
vi config_params.sh
./run_benchmark.sh release search knn
```


### All‑in‑Storage (AiSAQ)

AISaQ-DiskANN: [github.com/KioxiaAmerica/aisaq-diskann](https://github.com/KioxiaAmerica/aisaq-diskann)

Use this project to evaluate the All‑in‑Storage (AiSAQ) variant.

```bash
mkdir build && cd build && cmake .. && make -j

./apps/build_disk_index --data_type float --dist_fn l2 --data_path ../../../datasets/sift/sift_base.fbin --index_path_prefix sift1m_idx -R 64 -L 125 -B 0.02 -M 50 --use_aisaq --inline_pq 64

sudo ./apps/search_disk_index --data_type float --dist_fn l2 --index_path_prefix sift1m_idx --query_file ../../../datasets/sift/sift_query.fbin --gt_file ../../../datasets/sift/sift_query_base_gt100 -K 10 -L 10 20 30 40 50 60 70 80 90 100 --result_path res --num_nodes_to_cache 0 -W 8 --use_aisaq --pq_read_io_engine aio
```

### Pipeline ± DynamicWidth ± MemGraph

PipeANN: [github.com/LeonLee666/PipeANN](https://github.com/LeonLee666/PipeANN) (main branch)
Pipeline evaluation:
```bash
# ensure you are on main branch
git checkout main
# ensure -DDYN_PIPE_WIDTH and -DSTATIC_POLICY are commented out in CMakeLists.txt
cd build; cmake ..; make -j
cd ..

# build disk index
build/tests/build_disk_index float ../../datasets/sift/sift_base.fbin ${PWD}/sift-index 64 125 0.02 50 48 l2 0

# build in‑memory navigator
build/tests/utils/gen_random_slice float ../../datasets/sift/sift_base.fbin ${PWD}/sift-index_sample_rate_0.01 0.01

build/tests/build_memory_index float ${PWD}/sift-index_sample_rate_0.01_data.bin  ${PWD}/sift-index_sample_rate_0.01_ids.bin ${PWD}/sift-index_mem.index 0 0 48 128 1.2 24 l2

# run pipeline search
build/tests/search_disk_index float ${PWD}/sift-index 48 8 ../../datasets/sift/sift_query.fbin  ../../datasets/sift/sift_query_base_gt100 10 l2 2 0 10 20 30 40 50 60 70 80 90 100

```

Pipeline + DynamicWidth:
```bash
# ensure -DDYN_PIPE_WIDTH and -DSTATIC_POLICY are defined (not commented) in CMakeLists.txt
vi CMakeLists.txt
cd build; cmake ..; make -j ; cd ..
build/tests/search_disk_index float ${PWD}/sift-index 48 8 ../../datasets/sift/sift_query.fbin  ../../datasets/sift/sift_query_base_gt100 10 l2 2 0 10 20 30 40 50 60 70 80 90 100
# optional: also enable MemGraph
build/tests/search_disk_index float ${PWD}/sift-index 48 8 ../../datasets/sift/sift_query.fbin  ../../datasets/sift/sift_query_base_gt100 10 l2 2 1 10 20 30 40 50 60 70 80 90 100
```

### DynamicWidth (dyn‑beam branch)
DynamicWidth evaluation:

```bash
git checkout dyn-beam
# ensure -DDYN_PIPE_WIDTH and -DSTATIC_POLICY are commented; -DDYN_BEAM_WIDTH is enabled
vi CMakeLists.txt
cd build; cmake ..; make -j; cd ..

build/tests/search_disk_index float ${PWD}/sift-index 48 32 ../../datasets/sift/sift_query.fbin  ../../datasets/sift/sift_query_base_gt100 10 l2 0 0 10 20 30 40 50 60 70 80 90 100
```
## Citing

If you use this benchmark, please cite the original paper and this repository. Example BibTeX entries (fill in as appropriate):

```bibtex
@inproceedings{ioeff-ann-graph,
  title     = {I/O Efficiency in Graph-Based Disk-Resident Approximate Nearest Neighbor Search: A Design Space Exploration},
  author    = {Liang Li and Shufeng Gong and Yanan Yang and Yiduo Wang and Jie Wu},
  booktitle = {Proceedings of VLDB},
  year      = {<Year>},
  doi       = {<DOI>}
}

@misc{iobench4diskann,
  title  = {IObench4DiskANN: A Reproducible Benchmark for Disk-Resident Graph ANN I/O Efficiency},
  author = {Liang Li},
  year   = {2025},
  howpublished = {GitHub repository},
  note   = {\url{https://github.com/LeonLee666/IObench4DiskANN}}
}
```

## License

This meta-repository is intended for research use. External systems retain their respective original licenses. Please review and comply with the license terms in each linked repository before use.

## Acknowledgments

We thank the maintainers of DiskANN, Starling, PipeANN, and AISaQ-DiskANN for open-sourcing their implementations, and the community for datasets and tooling that enable reproducible ANN research.

