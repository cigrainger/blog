+++
title = "Explorer 0.8: The dtype release"
date = 2024-01-22
description = "Lists, structs, precision types, and Series macros — Explorer grows up."
[taxonomies]
tags = ["dataframes", "elixir", "explorer", "rust"]
+++

Well... it's been a while. Almost two years in fact! I would opine that time flies but it terrifies me so let's leave it there (everybody loves a bit of existential dread with their code blogs). I [last posted](https://cigrainger.com/introducing-explorer/) having published `v0.1.0`  of [`Explorer`](https://hexdocs.pm/explorer/Explorer.html), a dataframe library for Elixir. Since then, this little library has grown up -- a lot. I moved to the other side of the world and between that and some big work stuff that took all my time and meant we stopped using dataframes in our ETL, I drifted away from the project a bit. [Philip Sampaio](https://github.com/philss) and [José Valim](https://github.com/josevalim) took over maintenance of the project in my absence and really drove it forward. And in November, we added another member to the core team: [Billy Lanchantin](https://github.com/billylanchantin)!

<!-- more -->

There have been a ton of milestones that have made Explorer a unique contender among dataframe libraries:
- [`Explorer.Query`](https://hexdocs.pm/explorer/Explorer.Query.html) brought macros to the main `DataFrame` verbs, enabling the use of regular Elixir code like infix operators, `if`, `unless`, `cond`, and for-comprehensions to compile to efficient dataframe operations. This makes writing `Explorer` much more concise and readable, and stands alone when compared to `pandas` and `dplyr` even in terms of reducing impedance mismatch between the dataframe library and the rest of the language. I could (and should) write a blog post about this one alone, but go check out [the docs](https://hexdocs.pm/explorer/Explorer.Query.html) in the meantime.
- José and [Cocoa](https://github.com/cocoa-xu) (and others in the community) put together [ADBC](https://github.com/elixir-explorer/adbc) bindings for Elixir (which José [livestreamed](https://www.twitch.tv/collections/W_3_kJ4ncRcYhw)). This permits us to get data from databases over [ADBC](https://arrow.apache.org/docs/format/ADBC.html) and use it in Explorer without copying or marshalling. Explorer now has [`Explorer.DataFrame.from_query/4`](https://hexdocs.pm/explorer/Explorer.DataFrame.html#from_query/4), so you can get a dataframe directly from an SQL query.
- Phils and others put together [FSS](https://github.com/elixir-explorer/fss) so we can work with CSVs, IPC (Arrow), and Parquet files directly from S3 or HTTP. So you can pass in a URI instead of a filepath to load a dataset.
- [Christine Guadelupe](https://github.com/cristineguadelupe) spearheaded the [`KinoExplorer`](https://github.com/livebook-dev/kino_explorer) project, which integrates `Explorer` with [Livebook](https://github.com/livebook-dev/livebook). You can use the Data Transform Smart Cell (as highlighted in Livebook's launch week [here](https://news.livebook.dev/data-wrangling-in-elixir-with-explorer-the-power-of-rust-the-elegance-of-r---launch-week-1---day-5-1xqwCI)) to build Explorer queries with a graphical interface. As with other smart cells, this is generating code behind the scenes and you can convert it to raw code and manipulate it from there!
- Innumerable improvements to the code from so many different contributors. It's been an absolute blast to see people improving the Rust, suggesting and implementing functions, and generally just taking the bare bones of a library that I put up and making it into something great. Thank you all.

`0.8.0` brings a number of other exciting developments closer to completion, so I'm going to lump them all in together. Let's get into it.

## `dtypes` galore

### Precision

When I first put together `Explorer`, I wanted to match the simplicity of working with data types in `R`. It can be overwhelming when you're doing data analysis to have to think about precision, casting, and all that fun stuff. Most of the time, you want an `integer` to be an `integer` and a `float` to be a `float` and not have to think more about it. Until you don't. Because `Explorer` supports _zero copy_ conversion [to Nx tensors](https://hexdocs.pm/explorer/Explorer.Series.html#to_tensor/2), you can go straight from ADBC -> `Nx.Tensor` or IPC -> `Nx.Tensor`, or whatever you want. If you're doing machine learning, you care about precision and you care about speed of dataloading. So `Explorer` now supports `{:f, size}` for 64- or 32-bit floats, `{:s, size}` for 8-, 16-, 32-, or 64-bit signed integers, and `{:u, size}` for the same for unsigned integers. This follows on from improvements made earlier to add precision to `:datetime` and `:duration` dtypes.

To avoid complicating this for everyday data analysis, however, we retained `:float` and `:integer` as aliases for `{:f, 64}` and `{:s, 64}` respectively. These match Elixir's floats and integers. So if you don't want to worry about precision, _you don't have to_. And a note on the notation (which also can be written as `:f64`, `:s64`, etc.): this matches [`Nx`'s notation](https://github.com/elixir-nx/nx/tree/main/nx#readme) -- just one more way we try to keep Explorer aligned with core Elixir and major libraries.

### Lists and Structs

The `:list` dtype was introduced in `v0.7.2`. I'm discussing it here because I'm catching up on blogging, and because we've added additional functionality to really flesh it out. List is really `{:list, any()}` where `any()` is any valid dtype, including lists. It's a recursive dtype, and we can do some powerful stuff with it. Let's have a look at some code.

```elixir
alias Explorer.Series

s =
  1..100
  |> Stream.map(fn _ -> :random.uniform(100) end)
  |> Enum.chunk_every(10)
  |> Series.from_list()
```

```elixir
#Explorer.Series<
  Polars[10]
  list[s64] [
    [46, 55, 76, 68, 37, 51, 7, 62, 4, 37],
    [20, 40, 81, 35, 74, 95, 10, 94, 46, 5],
    [77, 79, 28, 56, 10, 18, 65, 8, 35, 4],
    [27, 95, 88, 46, 55, 76, 79, 35, 5, 20],
    [89, 21, 94, 1, 51, 13, 46, 72, 96, 32],
    [12, 39, 14, 88, 39, 43, 79, 55, 99, 3],
    [9, 76, 47, 67, 78, 77, 82, 3, 87, 72],
    [72, 94, 17, 2, 56, 25, 64, 5, 57, 77],
    [68, 61, 74, 69, 21, 81, 17, 89, 84, 45],
    [67, 28, 90, 71, 45, 96, 2, 50, 23, 29]
  ]
>
```

Okay, so we can see we've got a `list` of `s64` here. Unsurprisingly, sometimes you have a column of lists. Maybe names, maybe (like I've got) an array of sentences. In this case, we've got something that allows us to work with 2d-tensor-like data without needing a separate column for each element. The API for lists is currently very limited. The goal initially was just basic support to avoid blocking IO with data from Parquet, IPC, etc. There are a number of useful functions now, like [`Series.join/2`](https://hexdocs.pm/explorer/Explorer.Series.html#join/2), which allows us to join lists into strings like [`Enum.join/2`](https://hexdocs.pm/elixir/1.16.0/Enum.html#join/2):

```elixir
s =
  Series.from_list([
    ["José", "Valim"],
    ["Philip", "Sampaio"],
    ["Billy", "Lanchantin"],
    ["Christopher", "Grainger"]
  ])
```

```elixir
#Explorer.Series<
  Polars[4]
  list[string] [
    ["José", "Valim"],
    ["Philip", "Sampaio"],
    ["Billy", "Lanchantin"],
    ["Christopher", "Grainger"]
  ]
>
```

```elixir
Series.join(s, " ")
```

```elixir
#Explorer.Series<
  Polars[4]
  string ["José Valim", "Philip Sampaio", "Billy Lanchantin", "Christopher Grainger"]
>
```

Now for structs.

Thanks to [Raphael Costa](https://github.com/costaraphael) (who has provided a number of excellent contributions), we now have [initial support for the struct dtype](https://github.com/elixir-explorer/explorer/pull/756)!

I'm particularly excited about this one, as it more fully unlocks common patterns with Parquet and NDJSON. It also makes Explorer more useful in the hands of folks who don't have a background in data science. Let's say you're working with Ecto and you're using associations. A typical schema will have assocs, which can be a struct or a list of structs. Let's flesh that out a bit.

I work with a lot of patent data, and our data warehouse is a bunch of XML in postgres in a star-like schema. It all revolves around a `PatentDocument`.

```elixir
import Ecto.Query

data =
  Warehouse.PatentDocument
  |> select([p], %{publication_id: p.publication_id, published: p.published, ucid: p.ucid})
  |> limit(2)
  |> Connie.Warehouse.Repo.all()
```

```elixir
[
  %{publication_id: 89474253, published: ~D[2008-06-04], ucid: "JP-4096750-B2"},
  %{publication_id: 114561913, published: ~D[2011-04-13], ucid: "EP-2309729-A1"}
]
```

Which I can turn into a dataframe easily, because a list of maps implements the [`Table` protocol](https://hexdocs.pm/table/Table.html):

```elixir
df = DataFrame.new(data)
```

```elixir
#Explorer.DataFrame<
  Polars[2 x 3]
  publication_id integer [89474253, 114561913]
  published date [2008-06-04, 2011-04-13]
  ucid string ["JP-4096750-B2", "EP-2309729-A1"]
>
```

Now, I could have just used ADBC and in practice I do. But I also appreciate how Ecto handles joins and unmarshalling. But it can be tough to deal with in a dataframe!

```elixir
data =
  Warehouse.PatentDocument
  |> join(:inner, [p], a in assoc(p, :abstracts))
  |> select([p, a], %{
    publication_id: p.publication_id,
    published: p.published,
    ucid: p.ucid,
    abstract: a
  })
  |> limit(2)
  |> Connie.Warehouse.Repo.all()
  |> Enum.map(
    &Map.update!(&1, :abstract, fn a ->
      a |> Map.from_struct() |> Map.drop([:publication_id, :__meta__, :patent_document])
    end)
  )
```

```elixir
[
  %{
    abstract: %{
      status: "v",
      modified_load_id: 445358,
      content: "<abstract mxw-id=\"PA2805192\" lang=\"EN\" source=\"translation\" load-source=\"docdb\"><p>A process for the production of a nanoparticle-containing medium, especially paint, in which the nanoparticles are produced by hydrolysis and condensation of metal alkoxide in the medium itself. An Independent claim is also included for a similar process using nanoparticles obtained by hydrolysis and condensation of silanes, also in the medium.</p></abstract>",
      abstract_id: 657383904
    },
    publication_id: 16788813,
    ucid: "DE-19924644-A1",
    published: ~D[2000-11-30]
  },
  %{
    abstract: %{
      status: "v",
      modified_load_id: 445358,
      content: "<abstract mxw-id=\"PA2805191\" lang=\"DE\" source=\"national office\" load-source=\"docdb\"><p>Ein Verfahren zum Herstellen eines Nanopartikel enthaltenden Lackes sieht vor, daß die Nanopartikel durch Hydrolyse von Metallalkoxid und/oder Silanen im Lack selbst erzeugt werden.</p></abstract>",
      abstract_id: 657383905
    },
    publication_id: 16788813,
    ucid: "DE-19924644-A1",
    published: ~D[2000-11-30]
  }
]
```

Let's see what we get!

```elixir
df = DataFrame.new(data)
```

```elixir
#Explorer.DataFrame<
  Polars[2 x 4]
  abstract struct[4] [
    %{"abstract_id" => 657383904,
     "content" => "<abstract mxw-id=\"PA2805192\" lang=\"EN\" source=\"translation\" load-source=\"docdb\"><p>A process for the production of a nanoparticle-containing medium, especially paint, in which the nanoparticles are produced by hydrolysis and condensation of metal alkoxide in the medium itself. An Independent claim is also included for a similar process using nanoparticles obtained by hydrolysis and condensation of silanes, also in the medium.</p></abstract>",
     "modified_load_id" => 445358, "status" => "v"},
    %{"abstract_id" => 657383905,
     "content" => "<abstract mxw-id=\"PA2805191\" lang=\"DE\" source=\"national office\" load-source=\"docdb\"><p>Ein Verfahren zum Herstellen eines Nanopartikel enthaltenden Lackes sieht vor, daß die Nanopartikel durch Hydrolyse von Metallalkoxid und/oder Silanen im Lack selbst erzeugt werden.</p></abstract>",
     "modified_load_id" => 445358, ...}
  ]
  publication_id s64 [16788813, 16788813]
  published date [2000-11-30, 2000-11-30]
  ucid string ["DE-19924644-A1", "DE-19924644-A1"]
>
```

That's pretty exciting. With `FSS` you could take this and send it directly to `S3` as is. But you can do more:

```elixir
DataFrame.unnest(df)
```

```elixir
#Explorer.DataFrame<
  Polars[2 x 7]
  abstract_id s64 [657383904, 657383905]
  content string ["<abstract mxw-id=\"PA2805192\" lang=\"EN\" source=\"translation\" load-source=\"docdb\"><p>A process for the production of a nanoparticle-containing medium, especially paint, in which the nanoparticles are produced by hydrolysis and condensation of metal alkoxide in the medium itself. An Independent claim is also included for a similar process using nanoparticles obtained by hydrolysis and condensation of silanes, also in the medium.</p></abstract>",
   "<abstract mxw-id=\"PA2805191\" lang=\"DE\" source=\"national office\" load-source=\"docdb\"><p>Ein Verfahren zum Herstellen eines Nanopartikel enthaltenden Lackes sieht vor, daß die Nanopartikel durch Hydrolyse von Metallalkoxid und/oder Silanen im Lack selbst erzeugt werden.</p></abstract>"]
  modified_load_id s64 [445358, 445358]
  status string ["v", "v"]
  publication_id s64 [16788813, 16788813]
  published date [2000-11-30, 2000-11-30]
  ucid string ["DE-19924644-A1", "DE-19924644-A1"]
>
```

Unnesting can be really powerful. For now, these features mostly give us IO parity and the ability to get at the data. Later, we can add more functionality to expand on our ability to do data analysis directly on lists and structs. I'm looking at you, [nested tibbles](https://tidyr.tidyverse.org/articles/nest.html).

## Series macros

Back in November, Billy raised [a PR](https://github.com/elixir-explorer/explorer/pull/728) in which he argued we should use macros with `Series`. At first we were unsure, but I think we came to a pretty elegant solution. Now before, you needed to pass a callback, like so:

```elixir
series = S.from_list([1, 2, 3])

S.filter(series, &(&1 |> S.remainder(2) |> S.equal(1)))
```

Now I don't this this is too bad, but it's definitely verbose. But now, we can use macros and it's much nicer:

```elixir
require Explorer.Series, as: Series

S.filter(series, remainder(_, 2) == 1)
```

We decided to use `_`, which is a bit unorthodox in Elixir and was a tough choice when we focus so much on avoiding impedance mismatch. But I do think it's really nice, and it makes sense because of the arbitrariness of names here. You can't pattern match within queries, so no risk of confusion there.

For `v0.8.0` Billy has added macro support for `sort` (which is now [`sort_by`](https://hexdocs.pm/explorer/Explorer.Series.html#sort_by/3)) and [`map`](https://hexdocs.pm/explorer/Explorer.Series.html#map/2). I think these are just really nice. From the docs:

```elixir
s = Explorer.Series.from_list([1, 2, 3])
Explorer.Series.map(s, _ * 2)
#Explorer.Series<
  Polars[3]
  s64 [2, 4, 6]
>
```

A note on `sort_by`: we brought `DataFrame` into line with `Series` here -- now instead of `arrange` and `sort`, they both use `sort_by`. Nice.

## What's next?

I think Explorer is more than functional today. For _most_ data science workflows, I think you could switch from R or Python and get your work done in Elixir, with all the benefits that implies.

First up is more complete support for `List` and `Struct` dtypes. I've raised that issue [here](https://github.com/elixir-explorer/explorer/issues/835) if you'd like to track progress.

After that, we want to make `lazy` mode more complete. In particular, this is useful for OLAP-style work, but it will make regular day to day work faster and more intuitive. What do I mean by that? Today, [lazy doesn't work with groups](https://github.com/elixir-explorer/explorer/issues/494). Making it work requires a reorganisation of where we accumulate lazy functions, introducing a stack to lazy dataframes.

Getting this right will be key for the next big step, which is to finally introduce our first non-Polars backend. The original design of pluggable backends was a conscious decision from someone who lived and loved `dplyr` and `dbplyr`. I think it's still the thing that folks who don't work in that world struggle to understand the most. `dbplyr` allows you to write dataframe functions in `dplyr` and then apply them against remote database tables. But I don't mean 'execute a query to bring data into memory' (though you can), I mean 'execute this query and keep it in a tmp table on the server until I ask'. I mean 'join this local table to a remote table'. I mean 'my data analysis pipeline is identical whether I'm running it in-memory, in Postgres, or on DuckDB'. I'm really excited to bring this to Elixir and we're nearly ready to do it.

We also have plans in the works to leverage Elixir's most powerful weapon: distribution. Keep an eye on this space!

### Acknowledgments

A huge thank you to all our contributors for this release and for all the work between `v0.1.0` and this one. An especially big thanks goes to those who are joining us for the first time. Your efforts continue to make Explorer something special.

Explore the full [changelog](https://hexdocs.pm/explorer/changelog.html#v0-8-0-2024-01-20) and come join us in shaping the future of data science in Elixir. For me, it's been your feedback and contributions that make this project so special. I really believe Elixir has a unique opportunity to combine the best of many (all?) worlds for data science.
