+++
title = "simdxml for Python: a faster ElementTree you don't have to rewrite for"
date = 2026-03-28
description = "A read-only drop-in replacement for xml.etree.ElementTree backed by SIMD structural indexing."
[taxonomies]
tags = ["python", "xml", "simd", "rust"]
[extra]
math = false
charts = true
+++

> **tl;dr**: `pip install simdxml`, change `from xml.etree import ElementTree` to `from simdxml.etree import ElementTree`. Your existing code works unchanged. Parsing is 2-5x faster than lxml, `findall()` is up to 42x faster, text extraction is 10-23x faster. Read-only (no mutation), zero dependencies, pre-built wheels for all platforms.

I [wrote yesterday](/blog/simdxml/) about simdxml, a SIMD-accelerated XML parser that builds flat structural arrays instead of a DOM tree. That post covered the Rust internals: two-pass parsing, quote masking with carry-less multiply, parallel chunk parsing, the whole lot. If you haven't read it and you're interested in the systems-level details, start there.

This post is about the Python side. [simdxml](https://pypi.org/project/simdxml/) v0.3 ships a read-only drop-in replacement for `xml.etree.ElementTree`. You change one import line, your existing code works, and it's faster. That's the pitch. Let me show you what it actually looks like with real patent XML.

<!-- more -->

## The one-line change

Here's the standard way to parse XML in Python:

```python
from xml.etree import ElementTree as ET

tree = ET.parse("patent.xml")
root = tree.getroot()

for title in root.findall(".//invention-title"):
    print(title.text)
```

Here's simdxml:

```python
from simdxml.etree import ElementTree as ET

tree = ET.parse("patent.xml")
root = tree.getroot()

for title in root.findall(".//invention-title"):
    print(title.text)
```

One import. Everything else stays the same. `find()`, `findall()`, `iterfind()`, `findtext()`, `iter()`, `itertext()`, `attrib`, `text`, `tail`, indexing, iteration, `ET.tostring()`, `ET.fromstring()`, even `iterparse()` and `canonicalize()`.

The catch: it's read-only. If you try to mutate an element (set an attribute, append a child, assign to `.text`), you get a `TypeError` with a message pointing you back to stdlib for construction. This is by design. simdxml elements are views into a structural index, not mutable tree nodes. For read-heavy workloads (parsing, querying, extracting data), this is fine. For building XML, use stdlib.

## Real patent XML

At [Amplified](https://amplified.ai) we work with WIPO ST.36 patent documents. These are not small, polite XML files. They're dense, attribute-heavy, deeply nested, and they come by the million. Here's a taste of what a real US patent looks like (US-11543041-B2, a granted patent for a "Two-piece guide bushing"):

```xml
<patent-document ucid="US-11543041-B2" lang="EN" country="US"
    doc-number="11543041" kind="B2" date="20230103"
    family-id="61245163" status="corrected">
  <bibliographic-data>
    <publication-reference ucid="US-11543041-B2">
      <document-id>
        <country>US</country>
        <doc-number>11543041</doc-number>
        <kind>B2</kind>
        <date>20230103</date>
      </document-id>
    </publication-reference>
    <technical-data>
      <classifications-ipcr>
        <classification-ipcr>E21D 7/02 ...</classification-ipcr>
        <classification-ipcr>F16K 15/06 ...</classification-ipcr>
      </classifications-ipcr>
      <invention-title lang="EN">Two-piece guide bushing</invention-title>
      <citations>
        <patent-citations>
          <patcit ucid="JP-2015208792-A">...</patcit>
          <patcit ucid="JP-5708229-B2">...</patcit>
          <!-- dozens more -->
        </patent-citations>
      </citations>
    </technical-data>
  </bibliographic-data>
</patent-document>
```

I've simplified the nesting here. The real files have `mxw-id` attributes on nearly everything, multiple `document-id` variants per reference (one in EPO format, one in the original office format), classification codes with embedded whitespace, and priority chains that can go several levels deep. A single patent document is typically 50-100 KB of XML.

Let's work with two real patent fixtures (a US grant and a European application) and see what the practical differences look like.

## Extracting data from patents

The most common thing we do with patent XML is extract fields. Titles, abstracts, classification codes, citation lists. Let's start simple.

```python
from simdxml.etree import ElementTree as ET

tree = ET.parse("patent-us.xml")
root = tree.getroot()

# Patent metadata from attributes
print(root.get("ucid"))        # US-11543041-B2
print(root.get("country"))     # US
print(root.get("date"))        # 20230103

# Invention title
title = root.findtext(".//invention-title")
print(title)                   # Two-piece guide bushing
```

So far this is identical to stdlib. Nothing to learn. Now let's do something a bit more involved: extract all the IPC classification codes.

```python
for cls in root.findall(".//classification-ipcr"):
    print(cls.text.strip())
```

Or all cited patent UCIDs:

```python
citations = [
    pat.get("ucid")
    for pat in root.findall(".//patcit")
    if pat.get("ucid")
]
print(f"{len(citations)} citations")
```

Again, this is exactly what you'd write with stdlib or lxml. The simdxml `ElementTree` wrapper translates ET path expressions to XPath internally and evaluates them against the structural index.

## Where the speed shows up

For a single 88 KB patent, parsing is fast in any library. You won't notice a difference. The speed matters when you're processing thousands or millions of documents. But even on single documents, certain operations show a clear gap.

Here's a quick comparison on our two patent fixtures. This isn't a proper benchmark (for that, see the [PyPI page](https://pypi.org/project/simdxml/)), just a directional sense of what changes:

```python
import simdxml
from xml.etree import ElementTree as stdlib_ET
from simdxml.etree import ElementTree as simd_ET

xml = open("patent-us.xml", "rb").read()

# stdlib
%timeit stdlib_ET.fromstring(xml)                    # ~150 us
%timeit stdlib_ET.fromstring(xml).findall(".//patcit") # ~170 us

# simdxml
%timeit simd_ET.fromstring(xml)                      # ~40 us
%timeit simd_ET.fromstring(xml).findall(".//patcit")   # ~45 us
```

But the real gap opens up on larger workloads. The formal benchmarks on a 100K-element catalogue (5.6 MB) tell the story:

<figure>
<div id="chart-et-bench" class="vega-chart"></div>
<figcaption>ElementTree API benchmarks on 100K-element catalogue (5.6 MB). Apple Silicon, Python 3.14, GC disabled, median of 20 runs. <span style="color:#A0422A">&#9679;</span> simdxml &nbsp; <span style="color:#3A5068">&#9679;</span> lxml &nbsp; <span style="color:#A5A49C">&#9679;</span> stdlib. Lines connect simdxml to lxml.</figcaption>
</figure>

<script>
(function() {
  var data = [
    {"op": "parse()", "order": 1, "simdxml": 10, "lxml": 33, "stdlib": 55},
    {"op": "findall('item')", "order": 2, "simdxml": 0.23, "lxml": 4.8, "stdlib": 0.89},
    {"op": "findall('.//item')", "order": 3, "simdxml": 0.15, "lxml": 6.2, "stdlib": 3.0},
    {"op": "findall(predicate)", "order": 4, "simdxml": 1.5, "lxml": 12, "stdlib": 4.9},
    {"op": "xpath_text('//name')", "order": 5, "simdxml": 2.1, "lxml": 19, "stdlib": 4.4},
    {"op": "child_tags()", "order": 6, "simdxml": 0.40, "lxml": 6.2, "stdlib": 1.5},
    {"op": "itertext()", "order": 7, "simdxml": 2.6, "lxml": 33, "stdlib": 1.4},
    {"op": "canonicalize()", "order": 8, "simdxml": 1.8, "lxml": 4.7, "stdlib": 4.6}
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
      "symbolSize": 80, "orient": "bottom", "direction": "horizontal"
    }
  };
  vegaEmbed('#chart-et-bench', {
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "width": "container", "height": 320, "background": "transparent",
    "config": cfg,
    "data": {"values": data},
    "encoding": {
      "y": {"field": "op", "type": "nominal", "sort": {"field": "order"}, "title": null,
             "axis": {"labelLimit": 220}}
    },
    "layer": [
      {
        "mark": {"type": "rule", "color": "#E6E4DD", "strokeWidth": 1},
        "encoding": {
          "x": {"field": "simdxml", "type": "quantitative",
                 "scale": {"type": "log", "domain": [0.1, 60]}, "title": "Time (ms)"},
          "x2": {"field": "lxml"}
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 70},
        "encoding": {
          "x": {"field": "simdxml", "type": "quantitative"},
          "color": {"value": "#A0422A"},
          "tooltip": [{"field": "op"}, {"field": "simdxml", "title": "simdxml (ms)"}]
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 70},
        "encoding": {
          "x": {"field": "lxml", "type": "quantitative"},
          "color": {"value": "#3A5068"},
          "tooltip": [{"field": "op"}, {"field": "lxml", "title": "lxml (ms)"}]
        }
      },
      {
        "mark": {"type": "point", "filled": true, "size": 70},
        "encoding": {
          "x": {"field": "stdlib", "type": "quantitative"},
          "color": {"value": "#A5A49C"},
          "tooltip": [{"field": "op"}, {"field": "stdlib", "title": "stdlib (ms)"}]
        }
      }
    ]
  }, {actions: false, renderer: "svg"});
})();
</script>

simdxml is faster than lxml on every operation. It's faster than stdlib on 11 of 14 benchmarked operations. The three where stdlib wins (`iter()`, `itertext()`, `iter(tag)`) involve creating Python Element objects per node, where stdlib's C implementation has lower per-object overhead. simdxml provides batch alternatives for those workloads: `child_tags()` and `xpath_text()` skip Element creation entirely and return plain strings.

The `findall(".//item")` result is the most striking: 0.15 ms versus lxml's 6.2 ms. 42x. That's the inverted posting list doing its job. A descendant search for a known tag name is a scan over a sorted array, not a tree walk.

## Beyond ElementTree: the native API

The `ElementTree` compatibility layer is great when you have existing code. But if you're writing new code, the native API is more direct and often faster.

```python
import simdxml

doc = simdxml.parse(xml_bytes)

# XPath text extraction (no Element objects, just strings)
titles = doc.xpath_text("//invention-title")

# Compiled XPath for repeated use across documents
expr = simdxml.compile("//patcit/@ucid")
for xml in patent_xmls:
    doc = simdxml.parse(xml)
    ucids = expr.eval_text(doc)
```

The `xpath_text()` path is the fastest way to extract data. It skips Element creation entirely and returns strings directly from the structural index. On the PubMed benchmark corpus, it's 23x faster than lxml's `.xpath()` followed by `.text`.

### Compiled XPath

If you're running the same query across many documents, `simdxml.compile()` is like `re.compile()`. The XPath expression is parsed and optimised once, then reused:

```python
expr = simdxml.compile("//claim[@type='independent']")

expr.eval_text(doc)     # -> list[str]
expr.eval_count(doc)    # -> int
expr.eval_exists(doc)   # -> bool
expr.eval(doc)          # -> list[Element]
```

### Batch processing with Bloom filters

For workloads where you're scanning thousands of documents with the same query and many won't match, there's a batch API with a Bloom filter prescan:

```python
docs = [open(f, "rb").read() for f in xml_files]
expr = simdxml.compile("//claim[@type='independent']")

# Bloom filter skips non-matching docs at ~10 GiB/s
results = simdxml.batch_xpath_text(docs, expr)

# Or parallelise across cores
results = simdxml.batch_xpath_text_parallel(docs, expr, max_threads=4)
```

The Bloom filter checks tag names against a 128-bit filter before doing any parsing. Documents that can't possibly contain the target element are skipped entirely. On a batch of 1K small documents, the Bloom path is 3x faster than a serial Python loop.

## What's read-only and what's not

The full list of things that raise `TypeError`:

- Setting `.text`, `.tail`, or `.tag` on an Element
- `elem.set()`, `elem.append()`, `elem.remove()`, `elem.extend()`
- `ET.SubElement()`, `ET.Comment()`, `ET.ProcessingInstruction()`
- `ET.indent()`, `tree.write()`

Everything else works: all the query methods, iteration, attribute reading, serialisation via `ET.tostring()`, streaming via `iterparse()`, and even `canonicalize()` for C14N output.

If your workflow is "parse XML, extract data, do something with it", simdxml handles the whole thing. If your workflow involves modifying the XML and writing it back out, you still need stdlib or lxml for the mutation part. In practice, most XML processing I've seen in the wild is read-heavy. ETL pipelines, data extraction, config parsing, log analysis. The "parse and query" part is almost always the bottleneck, not the "construct and write" part.

## The trade-off

simdxml's parse step does more work upfront than stdlib. It eagerly builds CSR adjacency indices, name posting lists, parent maps, and pre/post-order numbers. This means `simdxml.parse()` is doing what stdlib spreads across parse + multiple queries.

If you parse a document and run a single `find()` that hits the first child, stdlib might win. If you parse once and run multiple queries, or if your query needs descendant search or predicates, simdxml pulls ahead quickly. The benchmarks reflect this: parsing is 3-5x faster, but queries are often 10-40x faster, because the index is already built.

## Installation

```bash
pip install simdxml
```

Pre-built wheels for Linux (x86\_64, aarch64), macOS (arm64, x86\_64), and Windows. No Rust toolchain needed. Zero runtime dependencies.

Source is on [GitHub](https://github.com/cigrainger/simdxml-python). The underlying Rust library is [here](https://github.com/cigrainger/simdxml).
