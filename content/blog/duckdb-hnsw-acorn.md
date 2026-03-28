+++
title = "Making HNSW actually work with WHERE clauses"
date = 2026-03-28
description = "Transparent filtered vector search in DuckDB, or: why post-filtering is broken and what to do about it."
[taxonomies]
tags = ["duckdb", "vector-search", "rust", "sql"]
[extra]
math = true
charts = false
+++

> **tl;dr**: DuckDB's built-in vector search extension applies WHERE clauses _after_ the index returns results, which means filtered queries silently return fewer results than requested (often zero). [hnsw\_acorn](https://github.com/cigrainger/duckdb-hnsw-acorn) fixes this by pushing predicates into the graph traversal using ACORN-1, adds RaBitQ quantisation for 21-30x memory reduction, and handles metadata joins and grouped top-K through optimiser rewrites. Standard SQL, no special syntax. `INSTALL hnsw_acorn FROM community`.

<!-- more -->

## The problem

DuckDB ships a vector search extension called `duckdb-vss`. It builds an HNSW index and you write queries like:

```sql
SELECT * FROM movies
WHERE language = 'Korean'
ORDER BY array_distance(embedding, query_vec)
LIMIT 10;
```

This looks like it should find the 10 nearest Korean-language movies. It doesn't. The HNSW index finds the 10 globally nearest movies, then the WHERE clause filters out non-Korean ones. If none of the global top 10 happen to be Korean, you get 0 rows. If 3 are Korean, you get 3 rows. The behaviour depends on the data distribution, and it fails silently.

This is not a minor issue. In practice, filtered vector search _is_ vector search. You almost never want "nearest to anything in the entire table." You want nearest within some category, from some user, in some date range. Post-filtering breaks all of these.

On a 228K movie dataset with 768-dim embeddings:

| Language filter | Selectivity | duckdb-vss | hnsw\_acorn |
|---|---|---|---|
| English | ~60% | 10/10 | 10/10 |
| Japanese | ~3% | 0-1/10 | **10/10** |
| Korean | ~1% | 0/10 | **10/10** |

At 60% selectivity, post-filtering works fine (most global nearest neighbours match anyway). At 3%, you're lucky to get one result. At 1%, you get nothing.

## ACORN-1: filter during traversal, not after

The fix is conceptually simple: instead of finding neighbours then filtering, only visit nodes that pass the predicate during the graph walk. [ACORN-1](https://arxiv.org/abs/2403.04871) does this by pushing a filter bitset into the HNSW traversal.

The interesting part is what happens when the filter is selective. An HNSW graph is built for global connectivity. When you filter out 97% of nodes, the graph has holes. Your direct neighbours might all fail the predicate, so the traversal has no edges to follow and terminates early with poor recall. This is why naive filtered search (even with the predicate pushed into the traversal) doesn't work well.

ACORN-1's fix is two-hop expansion. When a first-hop neighbour fails the predicate, instead of giving up on it, you expand through that failed neighbour's neighbours. If your direct neighbour isn't Korean, maybe your neighbour's neighbour is, and they're geometrically close enough to be a good candidate.

The important thing: this is a traversal strategy, not a new index structure. The existing HNSW graph is completely untouched. All the changes happen during search.

### Selectivity-based strategy switching

ACORN-1 is the right tool for moderate selectivity (1-60%), but it's not free. The two-hop expansion adds overhead, and at the extremes you can do better:

- **Above 60% selectivity**: standard HNSW + post-filter works fine. The graph is well-connected among matching nodes and two-hop expansion is pure overhead.
- **1-60% selectivity**: ACORN-1 with two-hop expansion. The sweet spot where post-filtering fails and brute force is too expensive.
- **Below 1% selectivity**: brute-force sequential scan. The graph is so sparse under the predicate that walking it is slower than just scanning all matching rows.

There's a per-node refinement too, borrowed from [Lucene's implementation](https://github.com/apache/lucene/pull/14160): if 90% of a node's direct neighbours already pass the filter, skip the two-hop expansion for that node. No point expanding when connectivity is fine.

## RaBitQ: making the index fit

HNSW indexes store full F32 vectors at every graph node. At 768 dimensions (a standard embedding size), that's 3 KB per vector. A million vectors is 3 GB just for the index.

<div>

[RaBitQ](https://arxiv.org/abs/2405.12497) quantises to 1 bit per dimension. The approach: subtract a centroid, normalise the residual, then store just the sign bits. For each quantised vector, two scalar corrections (the residual norm and a dot-product factor \\(\text{vdot} = \sum|u_d| / \sqrt{D}\\)) are stored alongside the binary representation.

</div>

The approximate distance between two quantised vectors uses Hamming distance plus these corrections:

<div>

$$\hat{d}^2 = \|r_a\|^2 + \|r_b\|^2 - 2\|r_a\|\|r_b\| \cdot \text{vdot}_a \cdot \text{vdot}_b \cdot \frac{D - 2H}{D}$$

where \\(H\\) is the Hamming distance between the sign-bit vectors and \\(D\\) is the dimensionality.

</div>

This gives roughly 21x compression at 128 dimensions and 30x at 768. The trick for maintaining accuracy: search the quantised index for candidates, then rescore the top results against the original F32 vectors stored in the table. You get the memory savings of binary search with the accuracy of exact distance computation.

```sql
CREATE INDEX idx ON embeddings USING HNSW (vec)
WITH (quantization = 'rabitq');
```

## The real challenge: making it transparent

Getting ACORN-1 and RaBitQ working was the easy part, in a sense. The hard part was making it all invisible. No special syntax, no `vss_match()` macros, no query hints. Just write SQL and the optimiser figures out the rest.

### Metadata joins

In practice, embeddings live in one table and metadata lives in another. The natural query is a JOIN:

```sql
SELECT m.title, m.genre
FROM embeddings e
JOIN metadata m ON e.id = m.id
WHERE m.genre = 'sci-fi'
ORDER BY array_distance(e.vec, query_vec)
LIMIT 10;
```

Making this use the HNSW index required a new optimiser rule that:

1. Recognises the JOIN + WHERE + ORDER BY + LIMIT pattern
2. Pre-scans the metadata table to find matching join keys
3. Maps those keys to row IDs in the embeddings table (using DuckDB's zone map pruning to skip irrelevant storage segments)
4. Builds a filter bitset and runs ACORN-1 filtered search
5. Keeps the JOIN in the plan so metadata columns can be reattached to the results

The optimiser has to match this pattern defensively. Wrong matches produce wrong results. It validates join key types, table relationships, distance function matching against the index, and preserves TOP\_N above the JOIN for non-1:1 key relationships.

### Grouped nearest neighbours

"Give me the 5 nearest items per category" is a natural question:

```sql
SELECT category,
       min_by(id, array_distance(vec, query_vec), 5)
FROM items
GROUP BY category;
```

My first attempt used oversampling: search for k\*10 results globally, then let the aggregate pick per-group top-K. This is a heuristic, and it fails badly with skewed data. If 90% of your nearest neighbours are in one category, the other categories get nothing.

The fix: per-group ACORN-1. For each distinct group value, build a filter bitset of rows belonging to that group, run a separate filtered HNSW search, collect all per-group results. No oversampling, no heuristic. Exact per-group recall regardless of data distribution.

The optimiser matches `AGGREGATE(min_by(col, distance, k))` with a GROUP BY, resolves the group column through DuckDB's compression projections, and replaces the plan with a grouped HNSW scan.

## The bugs that tests found

936 assertions across 33 test cases. Writing them wasn't just a box-ticking exercise. They caught two bugs that would have been invisible in basic testing, and both would have caused wrong results in production.

### Stale compression statistics

DuckDB's optimiser inserts integer compression projections based on column statistics. When tables are created and dropped in the same session, the statistics from a previous table can leak into the compression for the current table. Our scan replacement changes the data flowing through these projections, and the stale statistics cause the perfect hash aggregate to overflow.

The symptom: "aggregate group 256 exceeded total groups 4." It only appeared in the test runner (where multiple tests share a session), never in isolation. I spent the better part of a day tracing it back to a dropped table's cached min value.

The fix: refresh the compression constants from current table statistics after scan replacement. Then clear the group stats so `PERFECT_HASH_GROUP_BY` recalculates its bounds.

### WHERE filter pushdown ignored

When `WHERE category = 1` on an outer query gets pushed down into the scan by DuckDB's optimiser, the grouped HNSW scan was ignoring it. It would return all groups instead of just the filtered one.

The fix: scan filter columns alongside the group column and apply pushed-down predicates during bitset construction. A small code change. Without the tests, this would have shipped.

## The four optimiser rules

The extension installs four optimiser rules that work together:

1. **Expression optimiser**: rewrites `1 - array_cosine_similarity(a, b)` to `array_cosine_distance(a, b)` so the distance function matches the index metric.

2. **Scan optimiser**: matches `WHERE ... ORDER BY distance LIMIT k` and replaces the sequential scan with an HNSW index scan. Handles both plain filters (build bitset from WHERE) and metadata joins (build bitset from pre-scanned join table).

3. **TopK optimiser**: matches `GROUP BY category, min_by(id, distance, k)` and replaces with a grouped HNSW scan that runs per-group filtered searches.

4. **Join optimiser**: matches lateral join patterns (the SQL engine's internal representation of correlated subqueries) and replaces them with a physical HNSW index join operator.

All four converge on the same mechanism: build a bitset, pass it to ACORN-1, get results. Whether the bitset comes from a WHERE clause, a metadata table pre-scan, or a per-group partition, the search itself is the same.

## Benchmarks

100K rows, 128 dimensions, L2 distance. All latencies are single-query wall clock.

### Filtered search

This is the headline. 100% recall at every selectivity level, where upstream returns nothing at 1%.

| Selectivity | Recall@10 | Results | Latency |
|---|---|---|---|
| 100% (no filter) | 100% | 10/10 | 2 ms |
| 10% | 100% | 10/10 | 5 ms |
| 1% | 100% | 10/10 | 30 ms |
| 0.1% | 100% | 10/10 | 29 ms |

The latency increase at low selectivity is the cost of two-hop expansion. At 0.1% the strategy switches to brute force, which is actually slightly faster than 1% ACORN-1 (fewer candidates to evaluate).

### RaBitQ compression

| Method | Recall@10 | Index memory | Compression |
|---|---|---|---|
| HNSW (F32) | 100% | 50,000 KB | - |
| RaBitQ 3x | 90% | 2,344 KB | **21.3x** |
| RaBitQ 10x | 100% | 2,344 KB | **21.3x** |
| RaBitQ 30x | 100% | 2,344 KB | **21.3x** |

RaBitQ 3x means the candidate list is 3x the final k before rescoring. At 10x, recall reaches 100% because the rescore phase reranks with exact F32 distances. The memory savings are the same regardless of the oversampling factor (the quantised vectors are always 1 bit per dimension).

### Metadata joins

| Variant | Latency | Results |
|---|---|---|
| 10% selectivity, with index | 6 ms | 10/10 |
| 10% selectivity, brute force | 3 ms | - |
| 1% selectivity, with index | 31 ms | 10/10 |
| 1% selectivity, brute force | 3 ms | - |

The index is slower at 100K rows because bitset construction requires scanning the join column. The value here is correctness (guaranteed 10/10 filtered results), not raw latency. At 1M rows, 10% selectivity reaches parity (22 ms vs 23 ms brute force).

### Grouped nearest neighbours

| Variant | Latency |
|---|---|
| 10 groups, k=5, with index | 13 ms |
| 10 groups, k=5, brute force | 5 ms |
| 100 groups, k=5, with index | 2,974 ms |
| 100 groups, k=5, brute force | 6 ms |

Per-group HNSW search overhead dominates at high group counts (each group runs a separate filtered traversal). At 1M rows with 10 groups, the index wins: 34 ms vs 53 ms brute force. The crossover is around 500K rows for 10 groups. With 100 groups, brute force wins at any practical scale.

I want to be honest about the numbers. The primary value here is correctness (10/10 results with filters where upstream gives 0/10) and transparency (standard SQL, no special API). Speed wins come from filtered search and RaBitQ compression. Metadata joins and grouped search deliver correct results and scale well, but at small-to-medium sizes the overhead of bitset construction means brute force can be faster. That's fine. Correctness first.

## What you get

```bash
INSTALL hnsw_acorn FROM community;
LOAD hnsw_acorn;
```

Standard SQL, transparent optimisation. Users write normal queries and the optimiser figures out whether to use filtered HNSW search, metadata-derived bitsets, or per-group filtered search.

```sql
-- Filtered kNN
SELECT * FROM items
WHERE category = 'sci-fi'
ORDER BY array_distance(vec, query_vec) LIMIT 10;

-- Metadata join
SELECT m.title FROM embeddings e
JOIN metadata m ON e.id = m.id
WHERE m.year > 2020
ORDER BY array_distance(e.vec, query_vec) LIMIT 10;

-- Grouped top-K
SELECT category,
       min_by(title, array_distance(vec, query_vec), 5)
FROM items GROUP BY category;

-- RaBitQ quantisation
CREATE INDEX idx ON items USING HNSW (vec)
WITH (quantization = 'rabitq');

-- Prepared statements
PREPARE search AS
SELECT * FROM items
WHERE category = $2
ORDER BY array_distance(vec, $1::FLOAT[3]) LIMIT 10;
EXECUTE search([1,2,3], 'sci-fi');
```

All distance metrics (L2, cosine, inner product). All of it composes. You can do a cosine metadata join with RaBitQ quantisation and it just works.

Source is on [GitHub](https://github.com/cigrainger/duckdb-hnsw-acorn).
