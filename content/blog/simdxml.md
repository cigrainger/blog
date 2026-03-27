+++
title = "simdxml: structural indexing for XML"
date = 2026-03-27
description = "Applying simdjson's ideas to XML parsing. It's complicated, but it works."
[taxonomies]
tags = ["rust", "simd", "xml", "duckdb"]
[extra]
math = true
charts = true
+++

At [Amplified](https://amplified.ai), we work with patent data. A lot of it. Patent documents worldwide are published in WIPO's [ST.36](https://www.wipo.int/standards/en/st36/) standard, which means XML. Deeply nested, attribute-heavy, namespace-laden XML. Our data warehouse is fundamentally a collection of these documents, and we're constantly querying into them to extract claims, titles, classifications, and citations.

We've been experimenting with [DuckDB](https://duckdb.org) recently as a replacement for some of our heavier infrastructure, and I've been spending time with Arrow and SIMD techniques in Rust. At some point the obvious question came up: [simdjson](https://simdjson.org) showed that SIMD can radically speed up JSON parsing by building a structural index in a single vectorised pass. Could the same ideas work for XML?

The short answer: it's complicated, but yes. The result is [simdxml](https://github.com/cigrainger/simdxml), a Rust library that applies SIMD structural indexing to XML and integrates XPath evaluation directly on the resulting flat arrays. It looks like it's faster than quick-xml for parsing (despite also building a full XPath-ready index), and it matches or beats pugixml on end-to-end XPath queries. Its biggest wins are on large files and attribute-dense documents. It falls behind on some query patterns too, and I'll get into where and why.

This post walks through the architecture, the chain of optimisations we found along the way, and the benchmarks. I'll get into some of the maths where it's interesting.

<!-- more -->

## Why XML is harder than JSON

simdjson's core insight is that you can classify structural characters (`{`, `}`, `[`, `]`, `:`, `,`) in bulk using SIMD, producing a flat tape of positions that a second pass interprets. JSON's structural characters are single bytes with relatively simple quoting rules.

XML is messier. Tag names must be parsed and matched, not just delimited. Attributes create structure _within_ tags, not just between them. Self-closing tags need detection. And quoted attribute values can contain angle brackets, meaning you can't just scan for `<` and `>` without understanding quoting context.

The quoting problem is the most interesting one. In JSON, you only have double quotes, and simdjson handles them with a clever prefix-XOR trick. XML has both single and double quotes in attribute values, and they can be nested. There's also the matter of CDATA sections, comments, and processing instructions, none of which exist in JSON.

So the direct port isn't straightforward. But the core idea (SIMD classification of structural characters into bitmasks, followed by a sequential index-building pass) does transfer. It just takes more work.

## The two-pass architecture

simdxml uses a two-pass approach:

**Pass 1** processes the input in 64-byte chunks using SIMD instructions. On each chunk, it classifies every byte, producing bitmasks for `<`, `>`, `/`, `=`, `"`, and `'` positions. On AArch64 this uses NEON `VCEQ` comparisons across four 16-byte vectors per iteration; on x86\_64 it uses AVX2 `PCMPEQB` on two 32-byte vectors, or SSE4.2 as a fallback.

**Pass 2** is a sequential scan over those bitmasks. It walks the structural positions and builds flat arrays encoding the document tree: tag byte-offsets, interned tag names, depths, parent pointers, and text ranges. The output is a struct-of-arrays layout using roughly 16 bytes per tag, compared to pugixml's approximately 35 bytes per DOM node.

That second number matters when your XML files are 5 GB.

## Quote masking with carry-less multiply

The trickiest part of Pass 1 is figuring out which structural characters are inside quoted attribute values and should therefore be ignored.

simdjson solves this for JSON using a prefix-XOR on the double-quote bitmask. The idea: if you XOR a bitmask of quote positions with an all-ones constant using carry-less multiplication, the result has 1-bits in every position between matching quotes. AND the complement of that mask with your angle-bracket bitmask and you've masked out all quoted regions in one shot.

<div>

Concretely, let \\(q\\) be the bitmask of double-quote positions within a 64-byte chunk. The carry-less product

$$q \otimes \texttt{0xFFFF\ldots} = \operatorname{prefix\text{-}XOR}(q)$$

produces a mask where bits between matching quotes are set. This maps to a single instruction: `PCLMULQDQ` on x86 or `PMULL` on ARM NEON.

</div>

For XML, there's a complication: both `"` and `'` are valid attribute delimiters, and they can appear inside each other's values. The common case (only `"` appears, which holds for the vast majority of 64-byte chunks in real XML) is handled identically to simdjson. When both `"` and `'` appear in the same chunk, simdxml falls back to sequential state tracking for that chunk. In practice this fallback fires on less than 1% of chunks across all evaluated datasets. The cost is negligible.

## When SIMD isn't worth it

Here's something that surprised me. Not all XML benefits from the SIMD classifier.

The two-stage pipeline (SIMD classification followed by bitmask walking) pays off when there are lots of structural characters per byte of input. Attribute-heavy XML qualifies: every attribute means more `"`, `=`, and whitespace to classify in bulk. But text-heavy XML, where you have a handful of tags wrapping large stretches of prose, doesn't benefit as much. The structural characters are sparse, and the overhead of classifying _every_ byte is wasted work when most bytes are just text content.

The [memchr](https://docs.rs/memchr) crate uses SIMD internally for single-character scanning. For text-heavy XML, just scanning for `<` positions with `memchr` and parsing sequentially from each one turns out to be competitive with the full SIMD classifier. Sometimes faster.

simdxml handles this with a heuristic: at document open, it samples the first 4 KB and computes a _quote ratio_ (the fraction of 64-byte chunks containing at least one quote character). If the ratio is below 0.05, it skips the SIMD pipeline entirely and uses the `memchr`-based path.

The threshold is not sensitive. Varying it from 0.01 to 0.20 changes end-to-end query time by less than 3% across all evaluated datasets. The dispatch decision is made once and applies for the remainder of the parse.

This was a useful lesson. Sometimes the best SIMD optimisation is knowing when not to use SIMD.

## Flat arrays, not trees

Before talking about XPath, it's worth understanding what simdxml actually builds. A DOM parser like pugixml allocates a tree of linked nodes, each with pointers to parent, first-child, and next-sibling. This is flexible, but it means pointer chasing during traversal, per-node allocation overhead, and roughly 35 bytes per node.

simdxml builds a set of flat, densely packed arrays:

- **Tag table**: arrays of `(start, end, name_id, depth, pre, post, parent, flags)` tuples. Each entry is a 32-byte record, but since self-closing tags, text nodes, and attributes share entries, the amortised cost comes out around 16 bytes per tag.
- **Parent vector**: `parent[i]` gives the index of tag `i`'s parent. Upward traversal without pointer chasing.
- **Name table**: an interned string table mapping each unique tag/attribute name to an integer identifier. A reverse index maps each name ID to its sorted list of tag indices (an inverted posting list).
- **Text segments**: byte ranges for text content between tags, stored as `(tag_index, start, end)` triples.
- **CSR adjacency**: parent-child relationships in [Compressed Sparse Row](https://en.wikipedia.org/wiki/Sparse_matrix#Compressed_sparse_row_(CSR,_CRS_or_Yale_format)) format. An offsets array of length `n+1` and a children array. The children of tag `i` occupy `children[offsets[i]..offsets[i+1]]`. No per-node allocation, cache-friendly sequential access.

This struct-of-arrays layout is what makes everything else possible. It's cache-friendly, it fits in memory for very large documents, and it supports efficient XPath evaluation without ever building a tree.

## XPath on flat arrays

Building a structural index is only half the problem. The index needs to support actual XPath queries.

In a DOM, evaluating XPath means traversing pointer-linked tree nodes. In simdxml, the "tree" is a set of flat arrays, and XPath axes map to array operations.

### Pre/post-order numbering

<div>

Each tag receives a pre-order number (its position in document order) and a post-order number (assigned when its closing tag is encountered). This gives you constant-time ancestor and descendant checks: node \\(u\\) is an ancestor of node \\(v\\) if and only if

$$\operatorname{pre}(u) < \operatorname{pre}(v) \quad \text{and} \quad \operatorname{post}(u) > \operatorname{post}(v)$$

No tree walking required. This is a well-known technique from relational XML processing (Grust, 2002), adapted here to in-memory flat arrays.

</div>

### Inverted posting lists

Each interned tag name maps to a sorted array of tag indices. A name test like `//article` becomes a scan over the `article` posting list rather than a traversal of the entire document.

### Fused descendant scan

A common XPath pattern is a multi-step descendant path like `//article/title`. A naive evaluator would first collect all `article` elements (potentially millions), then for each one enumerate children named `title`.

<div>

simdxml recognises this at compile time and rewrites it as a single scan: iterate over the `title` posting list and, for each candidate, check whether its nearest ancestor named `article` exists using the parent vector. This avoids materialising the intermediate `article` node set and reduces work from \\(O(|\texttt{article}| \cdot k)\\) to \\(O(|\texttt{title}|)\\), where \\(k\\) is the average child count per article.

</div>

On DBLP (5.1 GB, 4.2 million articles), this optimisation is the difference between the query being feasible and it timing out.

### Fused attribute predicates

A similar optimisation applies to patterns like `//DescriptorName[@MajorTopicYN='Y']`. Rather than materialising all 330K `DescriptorName` elements and then filtering by attribute, the attribute check is performed inline during the descendant scan. Each candidate is tested as it's found, keeping the working set small. On PubMed this brings the query from 271 ms down to 132 ms.

## Parallel chunk parsing

This is the part I found most surprising. XML parsing has traditionally been considered inherently sequential. You need to track quoting state, nesting depth, and parent-child relationships, all of which depend on everything that came before.

simdjson doesn't parallelise its structural indexing stage either. For JSON, the sequential pass is fast enough. But for multi-gigabyte XML, the parse dominates wall-clock time, and I wanted to see if cores could help.

The key insight is that the SIMD first pass identifies all tag boundary positions. If you can find safe split points between complete elements (positions where a `>` ends one element and a `<` begins another), you can parse each chunk independently.

The algorithm:

1. Find `K-1` candidate split points by scanning for `>` characters at roughly equal intervals.
2. For each candidate, verify it's actually between elements (not inside a comment, CDATA section, or tag). This is a short backward scan.
3. Spawn `K` threads, each parsing its chunk independently. Each produces local tag tables with local offsets.
4. A sequential merge step concatenates the results and computes global depths, parent pointers, and pre/post numbers.

The expensive work (SIMD classification and bitmask scanning) runs in parallel. The sequential merge (depth and parent computation) is `O(n)` and fast.

<div>

On the 195 MB PubMed dataset (M4 Max), an Amdahl's law decomposition gives a serial fraction \\(s = 77\\) ms (57% of sequential time) and a parallel fraction \\(p = 59\\) ms (43%). Fitting:

$$T(n) = s + \frac{p}{n} = 77 + \frac{59}{n} \;\text{ms}$$

The theoretical maximum speedup is \\(1/0.57 = 1.75\times\\), consistent with the observed \\(1.48\times\\) at 4 threads. The serial portion comprises the result merge (adjusting parent indices, computing global pre/post numbers, and building the name table).

</div>

This approach trades generality for efficiency: it requires well-formed, record-oriented XML (a root element containing many sibling records). That is exactly what ST.36 patent documents, DBLP, PubMed, and most large real-world XML corpora look like.

## Chaining optimisations

One thing I enjoyed about this project was how the optimisations chain together. Each one opens the door to the next.

### Bloom filter prescan

For batch workloads where you're querying thousands of documents with the same XPath expression, many documents might not contain the target element at all. simdxml optionally constructs a 128-bit Bloom filter over tag names during the first pass. Before building the full structural index, the query's name tests are checked against the filter. Documents that definitely don't contain any relevant tag name are skipped entirely.

The Bloom filter construction adds negligible overhead since tag name positions are already being extracted. With two FNV-1a hash functions on different seeds, the false positive rate is around 2% for 20 unique tag names. Prescan speed is roughly 10 GB/s, near `memchr` throughput.

### Lazy index construction

Not every XPath query needs every index. A simple path like `/root/child` needs the CSR adjacency and name table, but not pre/post numbers. simdxml's query compiler analyses the XPath AST to determine which structural indices are required and constructs only those. For a query touching a single name at a known depth, pre/post computation can be skipped entirely, saving a full document traversal.

### Persistent index files

For corpora queried repeatedly, the structural index can be serialised to an `.sxi` file. On subsequent queries, the index is memory-mapped directly without reparsing. For DBLP, this turns a 3.2-second parse into near-zero amortised cost.

## Benchmarks

All experiments run on an Apple M4 Max (NEON) unless noted. simdxml is compiled with `rustc` in release mode. Baselines are pugixml 1.14 (compiled with `-O2`), quick-xml (Rust pull parser), and `xmllint` from libxml2. x86\_64 benchmarks use an AMD Ryzen 9 3950X (AVX2).

### Datasets

- **DBLP** (5.1 GB): The DBLP computer science bibliography. ~10 million bibliographic records in a single file. Large, deep, tag-heavy.
- **PubMed** (195 MB): PubMed abstracts. Mixed tags and text content, representative of biomedical data extraction workloads.
- **Attr-heavy** (10 MB): Synthetic dataset with a high ratio of attributes to elements, designed to stress quote masking.

### End-to-end XPath query performance

| Dataset | sxq | pugixml | vs. pugixml | xmllint | vs. xmllint |
|---|---|---|---|---|---|
| DBLP (5.1 GB) | 3.2 s | 8.3 s | **2.6x** | — | — |
| PubMed (195 MB) | 130 ms | 174 ms | **1.3x** | 831 ms | **6.4x** |
| Attr-heavy (10 MB) | 7.8 ms | 13.1 ms | **1.7x** | 138 ms | **17.7x** |

DBLP is too large for `xmllint` to process in reasonable time.

On DBLP, simdxml completes `//article` (selecting all 4.2 million article records) in 3.2 seconds versus pugixml's 8.3. The speedup comes from three sources: faster SIMD-based parsing, lower memory overhead (eliminating allocation pressure), and the fused descendant scan.

The attribute-heavy result (17.7x over `xmllint`) shows the SIMD quote-masking path doing its job. The 1.7x over pugixml is consistent across datasets with dense attribute content.

### Per-query breakdown on PubMed

This is where things get interesting and honest.

<figure>
<div id="chart-pubmed" class="vega-chart"></div>
<figcaption>PubMed per-query workload (195 MB, median of 5 runs). Lower is better. simdxml wins on descendant queries (Q1, Q3) and ties or loses on aggregation (Q4) and text predicates (Q5, Q6).</figcaption>
</figure>

<script>
vegaEmbed('#chart-pubmed', {
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": "container",
  "height": 280,
  "background": "transparent",
  "config": {
    "font": "Schibsted Grotesk, Helvetica Neue, sans-serif",
    "view": {"stroke": null},
    "axis": {
      "labelFont": "DM Mono, SF Mono, monospace",
      "labelFontSize": 10,
      "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk, Helvetica Neue, sans-serif",
      "titleFontSize": 11,
      "titleColor": "#33322E",
      "gridColor": "#E6E4DD",
      "domainColor": "#CBC9C0",
      "tickColor": "#CBC9C0"
    },
    "legend": {
      "labelFont": "DM Mono, SF Mono, monospace",
      "labelFontSize": 10,
      "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk",
      "titleFontSize": 11,
      "symbolSize": 80
    }
  },
  "data": {
    "values": [
      {"query": "//MeshHeading", "tool": "simdxml", "time": 130, "order": 1},
      {"query": "//MeshHeading", "tool": "pugixml", "time": 174, "order": 1},
      {"query": "//Article/ArticleTitle", "tool": "simdxml", "time": 150, "order": 2},
      {"query": "//Article/ArticleTitle", "tool": "pugixml", "time": 148, "order": 2},
      {"query": "//...[@MajorTopicYN='Y']", "tool": "simdxml", "time": 132, "order": 3},
      {"query": "//...[@MajorTopicYN='Y']", "tool": "pugixml", "time": 159, "order": 3},
      {"query": "count(//PubmedArticle)", "tool": "simdxml", "time": 133, "order": 4},
      {"query": "count(//PubmedArticle)", "tool": "pugixml", "time": 117, "order": 4},
      {"query": ".../Author (5-level)", "tool": "simdxml", "time": 186, "order": 5},
      {"query": ".../Author (5-level)", "tool": "pugixml", "time": 175, "order": 5},
      {"query": "contains(.,'cancer')", "tool": "simdxml", "time": 162, "order": 6},
      {"query": "contains(.,'cancer')", "tool": "pugixml", "time": 148, "order": 6}
    ]
  },
  "mark": {"type": "bar", "cornerRadiusEnd": 2},
  "encoding": {
    "y": {"field": "query", "type": "nominal", "sort": {"field": "order"}, "title": null,
           "axis": {"labelLimit": 200}},
    "x": {"field": "time", "type": "quantitative", "title": "Time (ms)",
           "scale": {"domain": [0, 200]}},
    "color": {
      "field": "tool", "type": "nominal", "title": null,
      "scale": {"domain": ["simdxml", "pugixml"], "range": ["#A0422A", "#3A5068"]}
    },
    "yOffset": {"field": "tool"},
    "tooltip": [
      {"field": "query", "title": "Query"},
      {"field": "tool", "title": "Parser"},
      {"field": "time", "title": "Time (ms)"}
    ]
  }
}, {actions: false, renderer: "svg"});
</script>

simdxml wins on high-selectivity descendant queries (Q1: `//MeshHeading`, 1.34x) and attribute predicates (Q3: `//DescriptorName[@MajorTopicYN='Y']`, 1.20x), where the posting list and fused predicate evaluation beat DOM traversal.

pugixml wins on scalar aggregation (Q4: `count()`, 0.88x) and text predicates (Q6: `contains()`, 0.91x). For `count`, pugixml's single-pass DOM construction skips the structural merge step, making it leaner. For `contains`, pugixml evaluates predicates inline during its DOM walk without a separate text-extraction pass, which the flat-array architecture can't match.

The deep path (Q5: `//PubmedArticle/.../Author`, 5 levels) is essentially a tie. CSR child enumeration matches DOM pointer chasing.

I find the per-query breakdown more illuminating than the headline numbers. The architectural trade-offs are real. Flat arrays win on selective traversal and lose on workloads where the DOM's single-pass construction avoids redundant work.

### Parse throughput

<figure>
<div id="chart-throughput" class="vega-chart"></div>
<figcaption>Parse-only throughput on PubMed (195 MB), M4 Max. Criterion microbenchmark. simdxml builds a full structural index; quick-xml yields pull-parser events only.</figcaption>
</figure>

<script>
vegaEmbed('#chart-throughput', {
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": "container",
  "height": 200,
  "background": "transparent",
  "config": {
    "font": "Schibsted Grotesk, Helvetica Neue, sans-serif",
    "view": {"stroke": null},
    "axis": {
      "labelFont": "DM Mono, SF Mono, monospace",
      "labelFontSize": 10,
      "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk, Helvetica Neue, sans-serif",
      "titleFontSize": 11,
      "titleColor": "#33322E",
      "gridColor": "#E6E4DD",
      "domainColor": "#CBC9C0",
      "tickColor": "#CBC9C0"
    }
  },
  "data": {
    "values": [
      {"config": "simdxml (4 threads)", "throughput": 2.12, "order": 1, "kind": "simdxml"},
      {"config": "simdxml (sequential)", "throughput": 1.43, "order": 2, "kind": "simdxml"},
      {"config": "quick-xml (sequential)", "throughput": 1.27, "order": 3, "kind": "baseline"}
    ]
  },
  "mark": {"type": "bar", "cornerRadiusEnd": 2},
  "encoding": {
    "y": {"field": "config", "type": "nominal", "sort": {"field": "order"}, "title": null,
           "axis": {"labelLimit": 200}},
    "x": {"field": "throughput", "type": "quantitative", "title": "Throughput (GB/s)",
           "scale": {"domain": [0, 2.4]}},
    "color": {
      "field": "kind", "type": "nominal", "legend": null,
      "scale": {"domain": ["simdxml", "baseline"], "range": ["#A0422A", "#A5A49C"]}
    },
    "tooltip": [
      {"field": "config", "title": "Configuration"},
      {"field": "throughput", "title": "GB/s", "format": ".2f"}
    ]
  }
}, {actions: false, renderer: "svg"});
</script>

Sequential simdxml achieves 1.43 GB/s on the M4 Max, comparable to quick-xml at 1.27 GB/s. This is the important bit: simdxml's sequential throughput is in the same ballpark as a widely-used Rust pull parser _despite building a full structural index_ (tag names, depths, parents, text ranges, CSR adjacency). quick-xml builds nothing; it just yields parse events. The index construction is essentially "free" relative to the raw parsing cost.

With 4 threads, throughput rises to 2.12 GB/s. Sub-linear scaling, as predicted by the Amdahl decomposition above.

### Memory

<figure>
<div id="chart-memory" class="vega-chart"></div>
<figcaption>Peak memory overhead during XPath evaluation (RSS). simdxml's flat arrays use roughly half the memory of pugixml's DOM.</figcaption>
</figure>

<script>
vegaEmbed('#chart-memory', {
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": "container",
  "height": 220,
  "background": "transparent",
  "config": {
    "font": "Schibsted Grotesk, Helvetica Neue, sans-serif",
    "view": {"stroke": null},
    "axis": {
      "labelFont": "DM Mono, SF Mono, monospace",
      "labelFontSize": 10,
      "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk, Helvetica Neue, sans-serif",
      "titleFontSize": 11,
      "titleColor": "#33322E",
      "gridColor": "#E6E4DD",
      "domainColor": "#CBC9C0",
      "tickColor": "#CBC9C0"
    }
  },
  "data": {
    "values": [
      {"dataset": "PubMed (195 MB)", "parser": "simdxml", "value": 16, "unit": "bytes/tag", "order": 1},
      {"dataset": "PubMed (195 MB)", "parser": "pugixml", "value": 35, "unit": "bytes/tag", "order": 1},
      {"dataset": "DBLP (5.1 GB)", "parser": "simdxml", "value": 3.6, "unit": "GB total", "order": 2},
      {"dataset": "DBLP (5.1 GB)", "parser": "pugixml", "value": 7.9, "unit": "GB total", "order": 2}
    ]
  },
  "facet": {
    "column": {"field": "dataset", "type": "nominal", "title": null,
               "header": {"labelFont": "Schibsted Grotesk", "labelFontSize": 11, "labelColor": "#33322E"}}
  },
  "spec": {
    "width": 220,
    "height": 160,
    "mark": {"type": "bar", "cornerRadiusEnd": 2, "width": 32},
    "encoding": {
      "x": {"field": "parser", "type": "nominal", "title": null},
      "y": {"field": "value", "type": "quantitative", "title": null},
      "color": {
        "field": "parser", "type": "nominal", "legend": null,
        "scale": {"domain": ["simdxml", "pugixml"], "range": ["#A0422A", "#3A5068"]}
      },
      "tooltip": [
        {"field": "parser"},
        {"field": "value"},
        {"field": "unit"}
      ]
    }
  }
}, {actions: false, renderer: "svg"});
</script>

On PubMed, simdxml uses roughly 16 bytes per tag versus pugixml's ~35 bytes per node. The savings come from eliminating per-node pointers (replaced by integer indices), interning tag names, and using CSR format (one integer per edge vs. pugixml's first-child/next-sibling pointer pairs).

On DBLP with 227 million tags, simdxml's index fits comfortably at ~3.6 GB. pugixml's DOM needs ~7.9 GB, which on a 16 GB machine doesn't leave much headroom for the query engine.

## The DuckDB extension

Part of the motivation for simdxml was DuckDB. We've been building DuckDB-native replacements for parts of our search infrastructure at Amplified, and one piece we needed was fast XPath evaluation over XML stored in Parquet columns.

[duckdb-xpath](https://github.com/cigrainger/duckdb-xpath) is a DuckDB extension wrapping simdxml. It provides `xpath_text`, `xpath_list`, `xpath_count`, `xpath_exists`, and `xpath_eval` as scalar functions, plus a `read_xml` table function.

The extension uses a three-tier evaluation strategy per row:

1. **Grep mode**: for simple `//tagname` queries, a SIMD-backed `QuickScanner` does direct string search on raw bytes. No XML parsing at all.
2. **Bloom pre-filter**: for complex queries, a Bloom filter prescan rejects documents that can't possibly match.
3. **Full parse**: only when the first two tiers don't resolve. A reusable `ParseScratch` buffer eliminates per-row heap allocation after the first document in each DuckDB chunk.

XPath compilation happens once per chunk, not per row. The Bloom filter targets are computed once and reused. This layering means that for a 10K-document patent corpus, the gap is dramatic:

<figure>
<div id="chart-duckdb" class="vega-chart"></div>
<figcaption>duckdb-xpath (<span style="color:#A0422A">&#9679;</span>) vs. libxml2-backed baseline (<span style="color:#A5A49C">&#9679;</span>) on 10K patent documents. Log scale. Speedup labels show the gap.</figcaption>
</figure>

<script>
(function() {
  var data = [
    {"query": "//invention-title (text)", "order": 1, "duckdb": 0.005, "libxml2": 3.42, "speedup": "685x"},
    {"query": "//claim-text (exists)",    "order": 2, "duckdb": 0.025, "libxml2": 3.71, "speedup": "148x"},
    {"query": "//claim-text (count)",     "order": 3, "duckdb": 0.035, "libxml2": 3.64, "speedup": "104x"},
    {"query": "//claim-text (text)",      "order": 4, "duckdb": 0.655, "libxml2": 3.75, "speedup": "5.7x"}
  ];
  var cfg = {
    "font": "Schibsted Grotesk, Helvetica Neue, sans-serif",
    "view": {"stroke": null},
    "axis": {
      "labelFont": "DM Mono, SF Mono, monospace", "labelFontSize": 10, "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk, Helvetica Neue, sans-serif", "titleFontSize": 11, "titleColor": "#33322E",
      "gridColor": "#E6E4DD", "domainColor": "#CBC9C0", "tickColor": "#CBC9C0"
    },
    "legend": {
      "labelFont": "DM Mono, SF Mono, monospace", "labelFontSize": 10, "labelColor": "#7A7972",
      "symbolSize": 80
    }
  };
  vegaEmbed('#chart-duckdb', {
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "width": "container", "height": 200, "background": "transparent",
    "config": cfg,
    "data": {"values": data},
    "encoding": {
      "y": {"field": "query", "type": "nominal", "sort": {"field": "order"}, "title": null,
             "axis": {"labelLimit": 220}}
    },
    "layer": [
      {
        "mark": {"type": "rule", "color": "#E6E4DD", "strokeWidth": 1.5},
        "encoding": {
          "x": {"field": "duckdb", "type": "quantitative", "scale": {"type": "log", "domain": [0.003, 5]}, "title": "Time (seconds)"},
          "x2": {"field": "libxml2"}
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 90},
        "encoding": {
          "x": {"field": "duckdb", "type": "quantitative", "scale": {"type": "log"}},
          "color": {"value": "#A0422A"},
          "tooltip": [{"field": "query"}, {"field": "duckdb", "title": "duckdb-xpath (s)", "format": ".3f"}, {"field": "speedup"}]
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 90},
        "encoding": {
          "x": {"field": "libxml2", "type": "quantitative", "scale": {"type": "log"}},
          "color": {"value": "#A5A49C"},
          "tooltip": [{"field": "query"}, {"field": "libxml2", "title": "libxml2 (s)", "format": ".3f"}, {"field": "speedup"}]
        }
      },
      {
        "mark": {"type": "text", "align": "center", "dy": -12, "fontSize": 10,
                 "font": "DM Mono, SF Mono, monospace", "color": "#7A7972"},
        "encoding": {
          "x": {"field": "duckdb", "type": "quantitative", "scale": {"type": "log"}},
          "text": {"field": "speedup", "type": "nominal"}
        }
      }
    ]
  }, {actions: false, renderer: "svg"});
})();
</script>

The `//invention-title` query (100% hit rate, simple tag match) runs in 5 ms versus 3.42 seconds. That's the grep-mode fast path: SIMD string search on raw bytes, no parsing at all. As queries get more complex and the full parse path is needed, the gap narrows but remains substantial.

## What it can't do (yet)

Some honesty about limitations:

- **DTDs and schemas**: not processed. The parser assumes well-formed XML.
- **Text predicates at scale**: as the PubMed Q6 benchmark shows, `contains()` over many text nodes is not a strength. pugixml evaluates predicates during DOM construction; simdxml's flat-array layout requires a separate text-extraction pass.
- **Small files**: parallel parsing only pays off above ~64 KB. For small documents, the sequential path is used automatically, but the thread pool setup cost is avoided rather than amortised.
- **`count()` and aggregation**: pugixml's single-pass DOM build has an advantage for queries that do minimal XPath work. The structural merge step in simdxml adds overhead that isn't recovered when the query itself is trivial.

## Python bindings

As of today, simdxml is also available as a [Python package](https://pypi.org/project/simdxml/). Pre-built wheels for Linux (x86\_64, aarch64), macOS (arm64, x86\_64), and Windows. `pip install simdxml`.

The Python API has two faces. The native one exposes `parse`, `xpath_text`, compiled queries, and the like:

```python
import simdxml

doc = simdxml.parse(xml_bytes)
titles = doc.xpath_text("//title")

# Compiled queries for repeated use
expr = simdxml.compile("//book[@lang='en']/title")
for doc in documents:
    results = expr.eval_text(doc)
```

There's also a read-only `ElementTree`-compatible layer for drop-in use with existing code:

```python
from simdxml.etree import ElementTree as ET

tree = ET.parse("books.xml")
root = tree.getroot()
for title in root.findall(".//title"):
    print(title.text)

# Full XPath 1.0 also available
root.xpath("//book[contains(title, 'XML')]")
```

Elements are immutable (it's a structural index, not a mutable tree), so you get a `TypeError` if you try to modify them. For read-heavy workloads, though, the numbers against lxml and stdlib are solid.

Benchmarks are on Apple Silicon, Python 3.14, lxml 6.0. GC disabled during timing, 3 warmup + 20 timed iterations, median reported. Three corpus types: data-oriented (product catalogue), document-oriented (PubMed abstracts), config-oriented (Maven POM).

### Parsing

simdxml eagerly builds structural indices (CSR, name posting lists) at parse time. lxml builds a DOM tree without precomputed query indices. simdxml front-loads more work so queries are faster. Both numbers are real; the trade-off depends on your workload.

| Corpus | Size | simdxml | lxml | vs lxml |
|---|---|---|---|---|
| Catalogue (data) | 17 MB | 32 ms | 82 ms | **2.6x** |
| PubMed (document) | 17 MB | 27 ms | 61 ms | **2.2x** |
| POM (config) | 2.1 MB | 2.7 ms | 8.3 ms | **3.1x** |

### XPath queries (returning Elements)

This is the apples-to-apples comparison, both libraries returning Element objects:

<figure>
<div id="chart-python" class="vega-chart"></div>
<figcaption>XPath query benchmarks returning Elements, simdxml vs lxml. Log scale.</figcaption>
</figure>

<script>
(function() {
  var data = [
    {"query": "//item (catalogue 17 MB)", "order": 1, "simdxml": 3.4, "lxml": 21, "speedup": "6x"},
    {"query": "//item[@cat='cat5'] (17 MB)", "order": 2, "simdxml": 1.6, "lxml": 69, "speedup": "42x"},
    {"query": "//PubmedArticle (17 MB)", "order": 3, "simdxml": 0.35, "lxml": 9.8, "speedup": "28x"},
    {"query": "//Author[LastName=...] (17 MB)", "order": 4, "simdxml": 13, "lxml": 29, "speedup": "2.2x"},
    {"query": "//dependency (POM 2.1 MB)", "order": 5, "simdxml": 0.34, "lxml": 1.1, "speedup": "3.3x"},
    {"query": "//dep[scope='test'] (2.1 MB)", "order": 6, "simdxml": 2.4, "lxml": 3.6, "speedup": "1.5x"}
  ];
  var cfg = {
    "font": "Schibsted Grotesk, Helvetica Neue, sans-serif",
    "view": {"stroke": null},
    "axis": {
      "labelFont": "DM Mono, SF Mono, monospace", "labelFontSize": 10, "labelColor": "#7A7972",
      "titleFont": "Schibsted Grotesk, Helvetica Neue, sans-serif", "titleFontSize": 11, "titleColor": "#33322E",
      "gridColor": "#E6E4DD", "domainColor": "#CBC9C0", "tickColor": "#CBC9C0"
    },
    "legend": {
      "labelFont": "DM Mono, SF Mono, monospace", "labelFontSize": 10, "labelColor": "#7A7972",
      "symbolSize": 80
    }
  };
  vegaEmbed('#chart-python', {
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "width": "container", "height": 260, "background": "transparent",
    "config": cfg,
    "data": {"values": data},
    "encoding": {
      "y": {"field": "query", "type": "nominal", "sort": {"field": "order"}, "title": null,
             "axis": {"labelLimit": 260}}
    },
    "layer": [
      {
        "mark": {"type": "rule", "color": "#E6E4DD", "strokeWidth": 1.5},
        "encoding": {
          "x": {"field": "simdxml", "type": "quantitative", "scale": {"type": "log", "domain": [0.2, 100]}, "title": "Time (ms)"},
          "x2": {"field": "lxml"}
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 90},
        "encoding": {
          "x": {"field": "simdxml", "type": "quantitative"},
          "color": {"value": "#A0422A"},
          "tooltip": [{"field": "query"}, {"field": "simdxml", "title": "simdxml (ms)"}, {"field": "speedup"}]
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 90},
        "encoding": {
          "x": {"field": "lxml", "type": "quantitative"},
          "color": {"value": "#3A5068"},
          "tooltip": [{"field": "query"}, {"field": "lxml", "title": "lxml (ms)"}, {"field": "speedup"}]
        }
      },
      {
        "mark": {"type": "text", "align": "center", "dy": -12, "fontSize": 10,
                 "font": "DM Mono, SF Mono, monospace", "color": "#7A7972"},
        "encoding": {
          "x": {"field": "simdxml", "type": "quantitative"},
          "text": {"field": "speedup", "type": "nominal"}
        }
      }
    ]
  }, {actions: false, renderer: "svg"});
})();
</script>

The predicate query (`//item[@category="cat5"]`) is the standout: 1.6 ms versus lxml's 69 ms. 42x. The fused attribute predicate evaluation pays off even across the Python FFI boundary. Simple descendant queries are 6-28x faster, and even on small config files it's 1.5-3.3x.

### Text extraction

For ETL workloads where you just want strings, `xpath_text()` skips Element object creation entirely:

| Query | Corpus | simdxml | lxml | vs lxml |
|---|---|---|---|---|
| `//name` | Catalogue 17 MB | 1.8 ms | 37 ms | **20x** |
| `//AbstractText` | PubMed 17 MB | 0.31 ms | 7.1 ms | **23x** |
| `//artifactId` | POM 2.1 MB | 0.21 ms | 2.0 ms | **10x** |

This is the path that matters most for our patent data extraction workloads at Amplified. Skip the DOM, skip the Element objects, just give me the text.

## Wrapping up

simdxml is about 9,000 lines of Rust with zero runtime dependencies. The compiled CLI binary is 684 KB. It passes all 327 libxml2 XPath tests and the pugixml XPath test suite (the handful of discrepancies are down to ambiguities in the XPath spec, not correctness bugs).

It's available as a [Rust crate](https://crates.io/crates/simdxml), a [Python package](https://pypi.org/project/simdxml/), a [DuckDB extension](https://github.com/cigrainger/duckdb-xpath), and a CLI (`cargo install simdxml-cli`). Source is on [GitHub](https://github.com/cigrainger/simdxml).

XPath 2.0 support is in progress (currently at 80% of the QT3 conformance suite).
