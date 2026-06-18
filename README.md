# RangeGraph
### Augmented Dynamic Graph Maintenance via Lazy Segment Trees and Binary Indexed Trees


> **MSc Research Project** · Department of Computer Science  
> Extending: D-Tree (VLDB 2022) · DS-Tree (SIGMOD 2024)  
> Contribution: Lazy Segment Tree + BIT layer on fully dynamic graph adjacency

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Concrete Example](#2-concrete-example--dhaka-city-road-network)
3. [Research Questions](#3-research-questions)
4. [Literature Review](#4-literature-review)
5. [Proposed Approach](#5-proposed-approach)
6. [Datasets](#6-datasets)
7. [Project Plan](#7-10-week-project-plan)
8. [References](#8-references)

---

## 1. Motivation

Modern networked systems — road networks, social graphs, communication networks, and biological pathways — share a defining characteristic: **they change constantly**. Edges appear and disappear. Weights shift. Components split and merge. The ability to answer meaningful queries over these evolving structures in real time is one of the central challenges of modern algorithmics.

Existing work on dynamic graphs has made impressive progress on one specific problem: *connectivity*. Given two nodes u and v, is there a path between them after the latest edge insertion or deletion? The D-Tree (VLDB 2022) and DS-Tree (SIGMOD 2024) are state-of-the-art solutions to this problem, and they answer connectivity queries remarkably fast on real-world graphs.

However, practical applications demand far more than connectivity alone. Consider the following questions that arise naturally in any dynamic network:

- *"What is the minimum-weight edge connecting any pair of nodes in the range [u₁, u₂]?"*
- *"Increase the congestion weight of all road segments in zone 3 by 5 units — in one operation."*
- *"Find all nodes reachable within exactly k hops from node v?"*
- *"Which edges, sorted in lexicographic order of their endpoint pairs, fall within the range [(100, 200), (300, 400)]?"*

None of these questions can be answered efficiently by D-Tree or DS-Tree. Both systems operate at the level of individual edges — one insertion, one deletion, one query at a time. They have no mechanism for **bulk range updates**, **range edge queries**, or **lexicographically ordered edge scanning**.

This is the gap that **RangeGraph** addresses. By augmenting the dynamic graph maintenance layer with a **lazy segment tree** and a **Binary Indexed Tree (BIT)**, we introduce range query primitives directly into the adjacency structure of a fully dynamic graph. The result is a system that retains the fast connectivity guarantees of existing work while unlocking an entirely new class of operations — all within O(log n) per query or update.

---

## 2. Concrete Example — Dhaka City Road Network

To ground the motivation concretely, consider the road network of **Dhaka, Bangladesh** — one of the densest urban road networks in the world.

### Graph structure

```
Nodes  : ~5,000  (road intersections)
Edges  : ~12,000 (road segments, each with a congestion weight)
Updates: Continuous — floods, construction, accidents open and close roads
Queries: Continuous — routing apps, emergency services, traffic control
```

### Scenario: Monsoon flood event

It is June. A monsoon flood has submerged **zone 3** of Dhaka — specifically, all road segments connecting intersections numbered **800 through 1,200** in the graph. The following things need to happen simultaneously:

**1. Bulk weight update — "all roads in zone 3 are now impassable"**

A routing system needs to mark all ~200 edges in the range [800, 1200] as having congestion weight = ∞ (impassable). With existing systems:

```
D-Tree / DS-Tree approach:
  for each edge e in affected_edges:       ← loop over ~200 edges
      update(e, weight=infinity)           ← one operation per edge
  Total cost: O(200 × log n) operations
```

With RangeGraph's lazy segment tree:

```
RangeGraph approach:
  range_update(node_range=[800, 1200], delta=+infinity)   ← ONE operation
  Total cost: O(log n)   ← regardless of how many edges are in that range
```

This is the lazy propagation principle: the update is **deferred** and stored at the segment tree node covering [800, 1200]. It is only pushed to child nodes when those nodes are individually queried — which may never happen for most of them.

**2. Range query — "what is the cheapest detour around zone 3?"**

An emergency services dispatcher needs to find the **minimum-weight edge** connecting any node outside zone 3 to any node just outside the flood boundary (say, nodes 750–799 or 1201–1250).

```
D-Tree / DS-Tree:
  No range query support.
  Manual workaround: scan all edges incident to nodes 750–1250
  Cost: O(degree × n) in the worst case

RangeGraph:
  range_min_query(node_range=[750, 1250])
  Cost: O(log n)
```

**3. Lexicographic edge scan — "list all affected road pairs in order"**

A traffic authority needs to generate a report of all road closures, listed in order of their intersection IDs (i.e., sorted lexicographically by the pair (u, v)).

```
D-Tree / DS-Tree:
  No ordered edge structure. Requires sorting after retrieval.
  Cost: O(m log m) where m = number of affected edges

RangeGraph:
  Edges stored as (u, v) key pairs in lexicographic order within the segment tree.
  Scan from (800, 0) to (1200, ∞) in sorted order directly.
  Cost: O(k log n) where k = number of edges returned
```

**4. k-hop neighbourhood — "which hospitals are within 3 road segments of the flood zone?"**

Emergency services need all nodes reachable within **k = 3 hops** from any flooded node — to position ambulances at the nearest reachable intersections.

```
D-Tree / DS-Tree:
  Requires k rounds of BFS over the full adjacency structure.
  Cost: O(k × (n + m))

RangeGraph:
  k rounds of range queries on the segment tree layer.
  Cost: O(k × log n)
```

### Summary table for this example

| Operation | D-Tree / DS-Tree | RangeGraph | Speedup |
|---|---|---|---|
| Bulk update 200 edges in zone 3 | O(200 log n) | **O(log n)** | ~200× |
| Min-cost edge in node range | Not supported | **O(log n)** | ∞ |
| Lexicographic edge scan | O(m log m) post-sort | **O(k log n)** | Significant |
| k-hop neighbourhood (k=3) | O(k(n+m)) | **O(k log n)** | ~n/log n |
| Connectivity query (u, v) | O(1) [DS-Tree] | O(1) preserved | Same |

The key insight is that **RangeGraph does not sacrifice connectivity performance** — it preserves the O(1) connectivity query of DS-Tree while adding the full suite of range operations on top. This is the core technical claim of the project.

---

## 3. Research Questions

This project is guided by five specific research questions set by the supervising professor:

| # | Research Question | Technique in RangeGraph |
|---|---|---|
| RQ1 | Can edge weights be updated by an increment Δ > 1 in a single operation across a range? | Lazy propagation on segment tree |
| RQ2 | Can edge insertions and deletions be performed in O(log n) without rebuilding the structure? | Dynamic segment tree with BITS-Tree-style insertion |
| RQ3 | Can lexicographically ordered (u, v) edge pairs be queried and scanned efficiently? | Edges indexed as (u, v) keys in lex order within segment tree |
| RQ4 | Can both adjacency list and adjacency matrix representations be supported, alongside N(k) queries? | Segment tree for list, 2D BIT for matrix; k rounds of range queries for N(k) |
| RQ5 | Can full dynamic graph maintenance (connectivity + range) be sustained simultaneously? | Augmented D-Tree with synchronized segment tree and BIT layers |

---

## 4. Literature Review

| Paper | Year | Venue | Tools / Structures | What It Solved | Shortcomings |
|---|---|---|---|---|---|
| **D-Tree** (Chen, Lachish, Helmer, Böhlen) | 2022 | VLDB | Spanning trees, union-find, adjacency lists | Fast average-case dynamic connectivity on large real-world graphs | No range queries; no bulk updates; no lexicographic edge ordering; no k-hop support |
| **DS-Tree** (Xu, Zhang, Wen, Qin, Zhang, Lin) | 2024 | SIGMOD | Dual spanning tree + disjoint-set (union-find) | O(1) connectivity queries; best-known query time for dynamic undirected graphs | Higher update overhead; still no range query layer; single-edge operations only |
| **Dynamic Segment Tree for Ranges and Prefixes** | 2003 | IEEE Trans. Computers | Balanced BST augmented with range sets, endpoint scheme | Dynamic range insert/delete/query without needing all keys in advance | Not applied to graphs; no lazy propagation; no edge-pair key ordering |
| **BITS-Tree** (Easwarakumar, Hema) | 2015 | arXiv | Modified segment tree with extended interval nodes | Faster dynamic segment insertion/deletion; faster stabbing and range queries than standard segment trees | Not graph-aware; no lazy bulk update; no integration with connectivity structures |
| **Concurrent Dynamic Connectivity** (Fedorov, Koval, Alistarh) | 2021 | SPAA | Euler Tour Trees, concurrent spanning forest | First truly concurrent dynamic connectivity with sequential-equivalent time complexity | Complex implementation; no range query primitives; not benchmarked on standard datasets |

### Common pattern in the literature

Every paper in the table above — including the two state-of-the-art works — shares the same structural blind spot: **none integrates a range query data structure at the adjacency layer of the graph**. This is not an oversight; it reflects the historical focus of the dynamic graph community on connectivity as the primary query type. RangeGraph is the first work to take the position that range query support is not an optional add-on but a first-class primitive that should be embedded in the core graph maintenance structure.

---

## 5. Proposed Approach

RangeGraph is organized as a **three-layer architecture**:

```
┌─────────────────────────────────────────────────┐
│              Graph Input Layer                  │
│   Dynamic edge inserts / deletes / updates      │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│           Augmented D-Tree Core                 │
│  Spanning tree per component                    │
│  + Union-Find for O(1) connectivity             │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│       Lazy Segment Tree + BIT Layer             │
│  Lexicographic (u,v) edge indexing              │
│  Bulk Δ-updates via lazy propagation            │
│  Range min/max/sum queries in O(log n)          │
│  k-hop N(k) via k rounds of range queries       │
└─────────────────────────────────────────────────┘
```

### Complexity summary

| Operation | Before (D-Tree/DS-Tree) | After (RangeGraph) |
|---|---|---|
| Connectivity query | O(1) | **O(1)** (preserved) |
| Single edge insert/delete | O(log² n) amortized | O(log n) |
| Point edge weight update | O(log n) | O(log n) |
| Bulk range update (Δ on range) | Not supported | **O(log n)** |
| Range min/max/sum query | Not supported | **O(log n)** |
| Lexicographic edge scan | Not supported | **O(k log n)** |
| k-hop N(k) query | O(k(n+m)) | **O(k log n)** |

---

## 6. Datasets

| Dataset | Source | Scale | Type | Purpose |
|---|---|---|---|---|
| **SNAP Road Networks** | Stanford SNAP | ~1M nodes, 2.7M edges | Real-world | Simulate dynamic edge changes via timestamped road events; primary benchmark |
| **DIMACS Challenge Graphs** | DIMACS | 10K – 10M nodes | Standard benchmark | Direct comparison against D-Tree and DS-Tree published results |
| **DBLP Co-authorship** | Stanford SNAP | ~317K nodes, 1M edges | Real-world | Dynamic edge insertions as new papers are published; tests sparse update patterns |
| **LiveJournal Social Graph** | Stanford SNAP | ~4M nodes, 35M edges | Stress test | Tests scalability and bulk update performance at social-network scale |
| **Synthetic (Erdős–Rényi)** | Generated | Variable | Controlled | Isolates specific parameters: graph density, update rate, range width |

All SNAP datasets are freely available at [snap.stanford.edu](https://snap.stanford.edu/data/).  
DIMACS graphs are available at [dimacs.rutgers.edu](http://dimacs.rutgers.edu/programs/challenge/).  
D-Tree source code: [github.com/qingchen3/D-tree](https://github.com/qingchen3/D-tree)

---

## 7. 10-Week Project Plan

Biweekly professor check-ins at the end of weeks **2 · 4 · 6 · 8 · 10**.

| Weeks | Phase | Tasks | Checkpoint Deliverable |
|---|---|---|---|
| **1–2** | Baseline Reproduction | Read D-Tree & DS-Tree papers (use summaries above) · Clone D-Tree GitHub code · Run on SNAP Road Network dataset · Document observed performance and structural weaknesses | Baseline system running + written weakness analysis |
| **3–4** | Range Query Layer | Implement lazy segment tree from scratch · Unit-test on standalone arrays · Design (u,v) lexicographic key scheme for edges · Verify O(log n) bulk update correctness | Standalone segment tree with lazy propagation — tested and documented |
| **5–6** | Integration | Augment D-Tree with segment tree layer · Synchronize both structures on every insert/delete · Implement range edge queries · Implement lexicographic edge scanning | Integrated RangeGraph prototype — D-Tree core + range layer working together |
| **7–8** | Full Feature Set | Add lazy bulk Δ-updates (RQ1) · Implement k-hop N(k) via range query rounds (RQ4) · Add 2D BIT for adjacency matrix support · All 5 research questions answered in working code | All RQs implemented and tested · Performance measurements begin |
| **9–10** | Benchmarking + Paper | Run experiments comparing RangeGraph vs D-Tree vs DS-Tree on SNAP + DIMACS · Plot and analyse results · Write paper in IEEE two-column format · Finalize slides for submission | Submitted paper + benchmark results + presentation slides |

---

## 8. References

1. Chen, Q., Lachish, O., Helmer, S., & Böhlen, M. H. (2022). **D-Tree: Dynamic Spanning Trees for Connectivity Queries on Fully-Dynamic Undirected Graphs**. *Proceedings of the VLDB Endowment*. [arXiv:2207.06887](https://arxiv.org/abs/2207.06887)

2. Xu, J., Zhang, Y., Wen, D., Qin, L., Zhang, W., & Lin, X. (2024). **Constant-time Connectivity and 2-Edge Connectivity Querying in Dynamic Graphs**. *Proceedings of ACM SIGMOD 2024*. [arXiv:2601.17285](https://arxiv.org/abs/2601.17285)

3. **Dynamic Segment Trees for Ranges and Prefixes**. *IEEE Transactions on Computers*, 2003. [IEEE:4167788](https://ieeexplore.ieee.org/document/4167788)

4. Easwarakumar, K. S., & Hema, A. (2015). **BITS-Tree: An Efficient Data Structure for Segment Storage and Query Processing**. *arXiv*. [arXiv:1501.03435](https://arxiv.org/abs/1501.03435)

5. Fedorov, V., Koval, A., & Alistarh, D. (2021). **A Scalable Concurrent Algorithm for Dynamic Connectivity**. *ACM SPAA 2021*. [arXiv:2105.08098](https://arxiv.org/abs/2105.08098)

6. Fenwick, P. M. (1994). **A New Data Structure for Cumulative Frequency Tables**. *Software: Practice and Experience*, 24(3), 327–336.

---

> *This repository documents the research proposal and ongoing implementation of the RangeGraph project as part of an MSc Computer Science programme. The project is supervised and reviewed biweekly.*
