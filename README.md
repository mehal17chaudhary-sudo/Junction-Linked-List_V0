# Junction Linked List (JLI) — AMJ List (V0)

**Stability. Structural Intelligence. Transparent Cost.**

> **Status: early version (V0), kept for the record.**
> This repository holds the first design of JLI: a segmented list with junctions and shortcuts, searched in O(√N). The design has since been rebuilt as a three-level external index (junctions → blocks → block-level skip list) with a much broader benchmark study against a skip list.
> **For the current design and results, see [Junction-Linked-List_V1](https://github.com/mehal17chaudhary-sudo/Junction-Linked-List_V1).**

---

## What Is JLI?

Junction Linked List (JLI), also known as the **Anna Mani Junction List (AMJ List)**, is an in-memory, structurally driven ordered index built around one guiding principle: **the structure should carry the information it needs to stay efficient, rather than depending on the workload.**

JLI does not tune itself to access patterns. It does not rely on probabilistic balancing. Instead, it maintains explicit structural invariants that govern traversal, bound maintenance costs, and give you direct visibility into what the structure is doing and why.

---

## About the Name

JLI is also referred to as the **Anna Mani Junction List (AMJ List)**, named in tribute to **Anna Mani** (1918–2001), a pioneering Indian physicist and meteorologist.

In a newly independent India that depended on foreign meteorological instruments, Anna Mani took a harder path: she built reliable scientific instruments domestically, capable of operating under India's demanding and diverse conditions. Through her work at the India Meteorological Department, she standardized weather instruments for the entire country, later making foundational contributions to solar radiation measurement before renewable energy was a global concern.

Her legacy is one of **structural self-reliance over dependence on external conditions** — the same design ethos that guides this data structure.

---

## Why JLI Exists

Many fast in-memory structures hide their true costs. Probabilistic structures offer strong averages but no structural guarantee on tail behavior or maintenance timing. Structures that rely on workload-specific tuning can degrade when their assumptions stop holding.

These problems rarely show up in microbenchmarks. They show up at scale, over time, under mixed or shifting workloads. JLI was built to make that behavior explicit and measurable.

---

## Structural Design

JLI is built from three components:

**Segments** divide the list into fixed-size regions, bounding traversal and maintenance cost to local areas. Optimal segment size scales with `sqrt(N)`.

**Junctions** sit at segment midpoints. Each junction stores segment metadata and participates in traversal as a normal node, providing O(N/S) coarse navigation without a separate index layer.

**Shortcuts** are regular nodes within a segment that accelerate fine-grained traversal. They are placed at computed offsets, not randomly, and reduce the linear scan within a segment without adding a new structural tier.

---

## Maintenance Model

JLI uses three levels of explicit, rate-limited maintenance:

| Level | Scope | Trigger | Purpose |
|---|---|---|---|
| Local | Single segment | Junction drifts > `t_j` from center | Corrects drift in individual segments |
| Suboptimal | Flagged segments | Segment deviation exceeds `soft_pct` / `hard_pct` | Targets structurally degraded regions |
| Global | Full list | Last resort: escalation gate or emergency ratio | Emergency structural recovery |

None of these run on every operation. They are triggered by structural conditions rather than probabilistic thresholds, and their costs are exposed through the `get_metrics()` API. If a global rebuild fires, you will see it in the counter.

---

## What Was Observed

These are empirical observations from the V0 benchmarks, not proofs. They hold for the tested sizes, workloads, and machine, and should be read that way.

### 1. Structural invariants are maintained

In the tested runs, segment size bounds, junction positioning, and shortcut validity were kept within their limits by the maintenance system. Maintenance triggers on measurable drift, not on probability.

### 2. Maintenance behavior was stable across tested workloads

The *decision* to maintain is driven by structural state, not by the operation mix. Across the tested workloads, from pure-delete to extreme insert floods, the maintenance scan rate stayed within a narrow band.

This does not mean **latency** is workload-independent. It is not: search traversal and insert pressure have different raw costs. The observation is about maintenance behavior, not raw speed.

### 3. Long-run p99 drift was bounded in the tested runs

Over a 300k-operation continuous mixed workload, p99 latency drifted by a mean of **1.28×** from the first window to the last (range **1.16×–1.52×** across 4 seeds), with exactly 1 global rebuild per seed. No runaway growth was observed, but the drift is real and is not reported as zero.

### 4. p99 latency scaled flatly with N over a limited range

Across N = 5k–200k, the log-log scaling exponent of p99 latency was **−0.073**, so p99 did not grow with N in this range. This is an observation over a narrow range of sizes. It is **not** an asymptotic claim, and it should not be read as the structure getting cheaper as it grows: the cost model below is O(√N), and constant overheads dominate at the smaller sizes tested.

### 5. Tested size range and its limits

The structure kept its invariants across the tested range of in-memory dataset sizes. Parameters scale with `sqrt(N)`. Below roughly N = 5,000, constant overhead dominates and JLI is not a good fit.

---

## Cost Model

**Core operations:**

| Operation | Best Case | Average / Worst Case |
|---|---|---|
| Search | O(1) | O(N/S + S) |
| Insert | O(1) | O(N/S + S) |
| Delete | O(1) | O(N/S + S) |

`S` = segment size. At `S ≈ sqrt(N)`, this reduces to O(√N). Average and worst-case bounds are identical, because traversal follows structural logic rather than distributional assumptions.

**Comparison with a skip list:** a skip list's expected search cost is O(log N), which is asymptotically better than V0's O(√N). V0's contribution is its explicit structure, observable maintenance, and the stable behavior reported above, not a better asymptotic search bound. V1 addresses this with a multi-level external index; see the V1 repository.

**Maintenance:**

| Level | Cost |
|---|---|
| Local Rebuild | O(r · S) per pass, r = rebuilt segments |
| Suboptimal Rebuild | O(r · S) per pass, r = affected regions |
| Global Rebuild | O(N), last resort only |

All maintenance costs are measurable via `get_metrics()`.

---

## Observability

JLI exposes a set of read-only counters via `get_metrics()`:

```python
m = jli.get_metrics()
# {
#   "local_scan":   int,   # local drift scan events
#   "sub_scan":     int,   # suboptimal fragmentation scan events
#   "local_event":  int,   # local rebuild events
#   "sub_event":    int,   # suboptimal rebuild events
#   "global_event": int,   # global rebuild events
#   "local_cost":   int,   # nodes processed in local rebuilds
#   "sub_cost":     int,   # segments rebuilt in suboptimal passes
#   "global_cost":  int,   # nodes processed in global rebuilds
# }
```

These are direct counts of what the structure has done, not estimates.

---

## Configuration Notes

The empirically tuned parameter set outperformed the alternative configurations tried. Two hard limits are worth knowing: a `sub_interval` set excessively high lets deferred maintenance build up and eventually fire as a large latency spike, and a `local_interval` below ~500 causes thrashing. Both are documented in the API reference.

---

## Latency vs. Throughput

JLI carries an explicit maintenance overhead. This is by design and fully measurable. If your primary metric is peak raw throughput on a single operation type, a simpler structure will outperform it. If your primary metric is **stable, observable behavior over long runtimes under mixed workloads**, JLI is built for that.

---

## When to Use JLI

**A reasonable fit when:**
- Workloads are mixed, unknown, or shift over time
- Systems run continuously at high operation counts
- Maintenance cost must be measurable and bounded, not hidden
- Long-term latency stability matters
- Dataset size changes over the lifetime of the system

**Not designed for:**
- Very small datasets (N < ~5,000), where constant overhead dominates
- Random-access semantics: JLI is an ordered index, not a hash map or array
- Single-operation benchmarks optimized for one workload type
- Workloads that are overwhelmingly search-dominant

---

## API Quick Reference

```python
from JLI import JLI

# Initialize
jli = JLI(segment_size=175, shortcuts_per_junction=6)

# Load (most efficient path for initial data)
jli.build_from_values(sorted_keys)
jli.build_from_values(objects, key=lambda o: o["id"])

# Core operations
jli.insert(key)
jli.insert(key, payload=obj)
jli.delete(key)
node = jli.search(key)      # returns Node or None
jli.insert_many(keys)
jli.delete_many(keys)
jli.size()

# Observability
m = jli.get_metrics()       # full maintenance counter dict
```

Full API documentation: `API_README.md`

---

## Who Built This

**Mehal Chaudhary** — computer science engineering student and independent researcher, 18. Interested in systems design where structure, performance, and long-term behavior intersect.

JLI reflects a focus on structural invariants, maintenance-aware systems, and predictable performance over time rather than short-lived optimizations.

Open to internships, research collaborations, and engineering roles where systems thinking and scalability reasoning are valued.

---

*If the ideas behind JLI interest you, start with the [V1 repository](https://github.com/mehal17chaudhary-sudo/Junction-Linked-List_V1) and its benchmark results, then try it on your own workload.*
