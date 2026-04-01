+++
title = "Introducing Dux"
date = 2026-03-30
description = "DuckDB-native dataframes for Elixir, with distributed execution on the BEAM."
[taxonomies]
tags = ["elixir", "dataframes", "duckdb", "dux"]
[extra]
math = false
charts = false
+++

> **Dux**: DuckDB-native dataframes for Elixir with distributed execution on the BEAM. [GitHub](https://github.com/elixir-dux/dux) | [Website](https://dux.now) | `{:dux, "~> 0.3.0"}`

Four years ago I [introduced Explorer](/blog/introducing-explorer/), a dataframe library for Elixir built on Polars. I wanted the elegance of dplyr, the speed of a proper columnar engine, and the joy of working in Elixir. Explorer delivered on that. It's been adopted widely in the community, it's integrated with Livebook, Nx, and Ecto, and I'm still proud of what we built.

But I've been working on something new. [Dux](https://dux.now) is a DuckDB-native dataframe library for Elixir ([GitHub](https://github.com/elixir-dux/dux), [website](https://dux.now)). Dux as in ducks, as in multiple ducks. Plus an 'x' because, you know, it's Elixir. It borrows Explorer's verb design from dplyr, but the architecture is fundamentally different. It's faster on single-node operations, it distributes natively across BEAM nodes, it has features Explorer never had (graph algorithms, ASOF joins, cross-source queries), and the whole thing compiles to SQL with no NIF layer. Let me explain why.

<!-- more -->

## Why not just keep building Explorer?

Explorer is great. I mean that sincerely. Philip, José, Billy, and the community have done incredible work, and it's a mature, well-tested library. If you're doing single-node data analysis in Elixir, Explorer is excellent.

But we ran into some things that were hard to fix within Explorer's architecture.

The first is that Polars maintenance became a true albatross. Explorer wraps Polars via Rust NIFs, which means every Polars release can break the bindings. Polars is an amazing project, but the development process is fast and a lot of it is focused on the Python library. We found ourselves in a constant catch-up cycle, eventually having to give up features because we couldn't keep pace with upstream changes. And the NIF layer itself is a source of friction: Rust compilation, precompiled binary distribution, debugging across the FFI boundary. It's a lot of machinery.

Dux sidesteps all of this. It talks to DuckDB via [ADBC](https://github.com/livebook-dev/adbc) (Arrow Database Connectivity), a pure Elixir driver with precompiled binaries that just downloads and works. Operations accumulate as an AST and compile to SQL, so the entire interaction with DuckDB is through a standard database protocol. No NIF, no Rust toolchain, no FFI. The development complexity difference is significant. And because DuckDB has a proper extension system, you can [build extensions](/blog/duckdb-hnsw-acorn/) that add capabilities to every Dux user without touching the Dux codebase itself.

We also always knew the right direction was lazy-by-default: accumulate operations and only execute when you need results. But this was very difficult with Polars's Series API and the eager/lazy split. You end up fighting the abstraction.

The second is distribution. We tried to build `Explorer.Remote` to run Explorer on remote BEAM nodes. It petered out. The fundamental issue is that Polars operations are Rust function calls, and you can't serialise a Rust function pointer and ship it to another node. You'd need to rebuild the entire operation on the receiving side, which means duplicating the expression compiler for the remote path. It's a lot of work for something that should be natural on the BEAM.

The third is that DuckDB happened. A few weeks ago, I [built a DuckDB backend for Explorer](https://github.com/cigrainger/explorer_duckdb_backend). And in doing so I saw that DuckDB would let us realise the lazy-by-default and distributed vision that we'd always wanted but couldn't make work with Polars. So I went for it.

DuckDB has quietly become extraordinary. It reads Parquet, CSV, NDJSON, and Excel natively. It pushes filters into S3 reads. It queries Postgres and SQLite via ATTACH with predicate pushdown. It has window functions, recursive CTEs, 500+ built-in functions, Iceberg and Delta support, and it runs everywhere. When DuckDB is your engine, a lot of features you'd otherwise have to build are just... there.

## What Dux is

Dux is DuckDB + Elixir, compiled to standard SQL under the hood. You write:

```elixir
require Dux

Dux.from_parquet("s3://data/events/*.parquet")
|> Dux.filter(amount > 100 and status == "active")
|> Dux.group_by(category)
|> Dux.summarise(total: sum(amount), n: count(id))
|> Dux.sort_by(desc: total)
|> Dux.compute()
```

If you've used Explorer, this looks familiar. The verb design is the same: `filter`, `mutate`, `select`, `group_by`, `summarise`, `sort_by`, `join`. Pipe through functions, each returns a new dataframe.

The differences from Explorer:

- **No Series API**. All operations are dataframe-level. Column expressions use macros where bare identifiers are column names and `^` interpolates Elixir values (SQL-injection safe parameter binding).
- **Lazy by default**. Operations accumulate as an AST until you call `compute/1`, `to_rows/1`, or `to_columns/1`. Data doesn't touch Elixir until you ask for it.
- **DuckDB is the only backend**. No pluggable backends, no abstraction tax. Every DuckDB function, extension, and optimisation is available.
- **`_with` variants** for raw SQL. Every macro verb has a `_with` suffix that accepts DuckDB SQL strings, so you can drop down to SQL for window functions or anything the macro system doesn't cover:

```elixir
Dux.mutate_with(df,
  rank: "ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)"
)
```

## Distribution

Performance and features are nice, but this is the reason Dux is its own thing rather than a thin DuckDB wrapper. And it's the part that I think is genuinely only possible because of the BEAM.

If you've used Spark, you know what distributed dataframes look like: a JVM cluster, a resource manager (YARN or Kubernetes), a distributed filesystem (HDFS or S3 with a metastore), job serialisation via Java/Python closures, and a scheduler that turns your query plan into stages. It works. It also means you're running a distributed systems platform just to process data. The operational overhead is significant, and it leaks into everything: your deployment, your debugging, your cold start times, your cost model.

The BEAM (Erlang's virtual machine, which Elixir runs on) was built for distributed systems. Nodes discover each other, processes communicate via message passing, supervision trees handle failures, and the runtime has been doing this reliably for 30 years in telecom and messaging infrastructure. None of this is novel. What's novel is using it as the distribution layer for a dataframe library.

A `%Dux{}` struct is plain data: a list of operations, a source reference, some metadata. It serialises naturally across BEAM nodes with zero special handling. There's no JVM closure serialisation, no pickle, no function pointer shipping. You build a pipeline on one node and send it to another, and the receiving node compiles it to SQL and runs it against its own local DuckDB instance.

```elixir
workers = Dux.Remote.Worker.list()

Dux.from_parquet("s3://data/**/*.parquet")
|> Dux.distribute(workers)
|> Dux.filter(amount > 100)
|> Dux.group_by(region)
|> Dux.summarise(total: sum(amount))
|> Dux.compute()
```

There are three layers, and each does what it's best at.

**The BEAM** handles the distributed systems primitives: node discovery (`:pg` process groups), message passing with backpressure, fault tolerance via supervision trees, process-level isolation. You don't write networking code, retry logic, or health checks. A worker dying mid-computation is just a process exit that the supervisor handles.

**DuckDB** is the SQL engine on every node. Each worker has its own embedded DuckDB instance. It reads Parquet from S3 directly, executes SQL, and returns results as Arrow IPC. The data never flows through the coordinator for reads. This is important: unlike Spark, where the driver coordinates data movement, Dux workers pull data directly from storage. The coordinator only sends the query plan and merges results.

**Dux** implements the distributed algorithms on top of both: pipeline splitting, data partitioning, result merging, and distributed joins. This is the part I spent the most time on, and it's what makes the distribution correct rather than a toy.

### What's actually distributed

It's not just "run the query on each partition and concatenate." Dux implements proper distributed algorithms for the operations that need them:

**Aggregate rewrites.** Distributed aggregation requires decomposing aggregates for correct merging. `SUM` is trivial (sum the partial sums), but `AVG` isn't (you can't average the partial averages), and `STDDEV` requires real care. Dux handles this with a lattice framework:

- `SUM(x)` → workers compute `SUM(x)`, coordinator sums the partial sums
- `AVG(x)` → workers compute `SUM(x)` and `COUNT(x)`, coordinator divides
- `STDDEV(x)` → Welford's pairwise parallel merge (numerically stable, no catastrophic cancellation)
- `COUNT(DISTINCT x)` → HyperLogLog approximate merge

Results merge incrementally via a streaming merger as workers complete, so there's no memory spike from buffering all results before reducing.

**Distributed joins.** Two strategies depending on data size:

- **Broadcast join**: for star-schema patterns where one side is small. The coordinator serialises the small table as Arrow IPC and broadcasts it to all workers. Each worker joins its partition locally. A Bloom filter pre-filtering step broadcasts distinct join keys first so workers can eliminate non-matching rows before the join materialises.
- **Shuffle join**: for large-to-large joins. Both sides are hash-partitioned on the join key (over-partitioned by 4x for skew absorption), exchanged so co-located buckets end up on the same worker, then joined locally. Skew detection kicks in for heavy buckets: the heavy side gets split and the other side gets replicated across sub-buckets.

**Distributed writes.** Workers write their partitions directly to storage (Parquet to S3, INSERT INTO attached databases). For Hive-partitioned output, each worker writes its own subdirectory. No data roundtrip through the coordinator.

**Distributed graph algorithms.** PageRank runs as a broadcast-iterate loop: each iteration serialises current ranks to IPC, broadcasts to workers, collects contributions, merges, repeats until convergence. The graph edges stay partitioned across workers; only the rank vector moves.

All of this is invisible to the user. You write `summarise(avg: mean(amount))` and the system decomposes it correctly. You write a `join` and the coordinator picks broadcast or shuffle based on table sizes.

### FLAME: elastic compute without the cluster

In Spark, if you want to scale up you're either pre-provisioning a cluster or waiting for YARN/Kubernetes to allocate containers. Cold start is measured in minutes. You pay for idle capacity or you wait.

[FLAME](https://github.com/phoenixframework/flame) is a Phoenix project that lets you spin up short-lived BEAM nodes on demand. Dux integrates with FLAME to give you elastic distributed compute from a Livebook notebook or a deployed application:

```elixir
# Spin up 5 workers on Fly.io
Dux.Flame.spin_up(5, pool: :dux_pool)

# They read from S3 directly, no data through coordinator
Dux.from_parquet("s3://speedtest/**/*.parquet")
|> Dux.distribute(pool: :dux_pool)
|> Dux.filter(year(test_date) == 2024)
|> Dux.group_by(country)
|> Dux.summarise(
  avg_download: mean(avg_d_kbps),
  tests: count(test_id)
)
|> Dux.compute()
```

Each FLAME runner boots a fresh machine with DuckDB (cold start is seconds, not minutes, because there's no JVM), reads S3 data directly, computes its partition, and ships back Arrow IPC results. When idle, the machines terminate. You pay for compute by the second.

I've been using Spark for over a decade. The idea that I can, from the Livebook I'm already working in, spin up an ad-hoc cluster, point it at Parquet on S3, run a distributed query, collect the results back into my notebook, and spin the cluster down... that's not something I could do before without a lot of infrastructure. With FLAME and Dux, the cluster is ephemeral. It exists for the duration of your query.

There's a [full guide](https://hexdocs.pm/dux/flame-clusters.html) that walks through this with the [Ookla Speedtest open dataset](https://registry.opendata.aws/speedtest-global-performance/) (~20 GB of global internet speed measurements as Parquet on S3). You explore one quarter locally, spin up 3 workers, query all years across the cluster, compare fixed vs mobile broadband, find the worst-latency regions by quadkey, and write aggregated results back to S3. Each worker reads its files directly from S3, computes locally, and sends small aggregated results back. The whole thing is a Livebook notebook.

And because these are BEAM nodes, the telemetry flows back in real time. Workers emit distributed telemetry events as they complete partitions. You can wire that into a LiveView or a Livebook chart that updates as results stream in. Progress, skew detection, per-worker timing. It's not a black box you submit a job to and wait. It's a live distributed computation you can watch and interact with.

## Beyond dataframes

Explorer never had graph analytics. Dux does, because DuckDB's recursive CTE support and relational engine make graph algorithms natural:

```elixir
Dux.Graph.pagerank(edges, source: :from, target: :to)
Dux.Graph.shortest_paths(edges, source: :from, target: :to, origin: 1)
Dux.Graph.connected_components(edges, source: :from, target: :to)
```

And these distribute too. PageRank partitions the edge list across workers and broadcasts the rank vector each iteration until convergence.

**ASOF joins** for time-series alignment:

```elixir
Dux.asof_join(trades, quotes, on: :ticker, by: {:timestamp, :>=})
```

**Cross-source joins.** Parquet on S3 joined with a Postgres table, with filter pushdown on both sides:

```elixir
Dux.from_attached(:postgres, "SELECT * FROM customers")
|> Dux.join(Dux.from_parquet("s3://orders/*.parquet"), on: :customer_id)
```

**Zero-copy Nx tensors.** Numeric columns go from DuckDB to Nx without copying:

```elixir
df |> Dux.select([:x, :y, :z]) |> Dux.to_tensor()
```

## Performance

On 10M rows (Apple M4 Max, 128 GB, Benchee, median of timed iterations):

| Operation | Dux | Explorer (Polars) | |
|---|---|---|---|
| Filter (lazy) | 24 ms | 59 ms | **Dux 2.5x faster** |
| Filter (eager) | 45 ms | 53 ms | **Dux 1.2x faster** |
| Mutate (eager) | 17 ms | 28 ms | **Dux 1.6x faster** |
| Group + Summarise (lazy) | 40 ms | 63 ms | **Dux 1.6x faster** |
| Group + Summarise (eager) | 81 ms | 88 ms | **Dux 1.1x faster** |

Dux is faster than Explorer on every operation at 10M rows. The lazy path is particularly fast because DuckDB's view-based compute means no data is copied until results are actually read. The eager path (which materialises to a table) is still faster, but the gap narrows because both engines are doing real work.

Memory per operation is similar: 11-15 KB for Dux versus 8-9 KB for Explorer. Explorer is slightly more compact here because Polars' NIF memory is outside the BEAM heap, while Dux's ADBC driver allocates small control structures on the Elixir side.

These are single-node numbers. With distribution, the picture changes further: Dux partitions across workers while Polars runs on one core. But the single-node numbers stand on their own.

## Relationship to Explorer

Explorer is a mature, well-maintained library with excellent documentation and a community around it. If you're happy with Explorer, there's no urgency to switch. It works, it's battle-tested, and it has good Livebook integration.

Dux offers a different set of trade-offs: faster single-node performance, native distribution, DuckDB's full feature set (SQL functions, external databases, lakehouse formats, graph algorithms), and a simpler development model (no NIF). The cost is that it's younger, less documented, and the API is still settling.

I learned an enormous amount building Explorer. The verb design, the tension between composability and expressiveness, the importance of zero-friction interop with the rest of the Elixir ecosystem. Dux wouldn't exist without those lessons.

## Getting started

```elixir
# mix.exs
{:dux, "~> 0.3.0"}
```

There's a [getting started guide](https://hexdocs.pm/dux/getting-started.html) as a Livebook notebook, guides for [distributed execution](https://hexdocs.pm/dux/distributed.html) and [FLAME clusters](https://hexdocs.pm/dux/flame-clusters.html), and [graph analytics](https://hexdocs.pm/dux/graph-analytics.html).

It's still early. The API will likely change. But the core is solid and I've been using it in production at [Amplified](https://amplified.ai) for our heavier data processing workloads. Feedback and contributions are very welcome.

The source is on [GitHub](https://github.com/elixir-dux/dux), and there's a website at [dux.now](https://dux.now).
