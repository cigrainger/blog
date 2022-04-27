+++
title = "Introducing Explorer"
date = 2022-04-27

[taxonomies]
tags = ["elixir", "rust", "explorer"]
+++

Last night I published [`v0.1.0`](https://hexdocs.pm/explorer/Explorer.html) of [`Explorer`](https://github.com/elixir-nx/explorer), a library I've been working on to bring dataframes to Elixir. It's been available through GitHub for a while, but the Hex release is more than just a milestone: by publishing the package on Hex, we've enabled [Rustler Precompiled](https://dashbit.co/blog/rustler-precompiled) to do its thing. That means that now you can use Explorer without a Rust toolchain installed and without waiting for it to compile. Before you'd wait for up to five minutes to try it out, and now it's basically instant.

In this post, I'd like to lay out some of what led to Explorer and the philosophy behind it. I'll follow up in coming weeks with more 'how-to' style posts, but I think the unique nature of Explorer merits some discussion first.

## Background

When I first heard about the [`Nx`](https://dashbit.co/blog/nx-numerical-elixir-is-now-publicly-available) project, I was thrilled. At [Amplified](https://amplified.ai), we've been using Elixir in production since 2018. We switched over to [`LiveView`](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) in early 2020 and haven't looked back. And we do deep learning. A bunch of it. I'm endlessly grateful to all the folks who have made the Python machine learning ecosystem as robust as it is, but... it's painful. We don't have teams dedicated to different languages. With the machine learning, it's just me. I think there are objective reasons why Python is problematic for machine learning. I'm not the only one. But this isn't the post for that. At a minimum, Nx meant the prospect of deploying our machine learning models as part of our Elixir app instead of maintaining Python microservices alongside.

Immediately, however, I knew there was a potential challenge to adoption and an opportunity to make something I knew that I'd really love and thought others would too. Something I've commented on wistfully a bunch of times but never had an incentive to allocate the time or energy to championing. Dataframes in Elixir.

See, I cut my data science teeth in R. I love R. I love the `tidyverse`. I pray at the alter of Hadley Wickham. But I also know that R is a crazy language. Much of the effort of the `tidyverse` has been to emphasise functional paradigms in R. Because what is data science but piping data through functions? And R is great at that now. You can read this and -- even if you have no knowledge of R -- you know what's going on.

```R
df |>
  select(sepal_length, sepal_width, species) |>
  filter(sepal_length < 5.0) |>
  group_by(species) |>
  summarise(max_sepal_width = max(sepal_width),
            mean_sepal_length = mean(sepal_length))
```

Compare that to `pandas`.

```python
df[df.sepal_length < 5.0].filter(
    items=["sepal_length", "sepal_width", "species"]
).groupby(by=["species"]).agg({"sepal_width": "max", "sepal_length": "mean"})
```

... I think? And that's formatted with Black. Pandas is ['the most important tool in data science'](https://qz.com/1126615/the-story-of-the-most-important-tool-in-data-science/) but it's also one of the most loathed. Or not -- one thing I find is that people don't realise _there are alternatives_.

Look, I'm not here to throw grenades or start holy wars. Pandas is just not my cup of tea. It's not how my brain works when it comes to data science. Again, I'm not the only one. At the end of the day, we're writing code to investigate and manipulate data. There should be as little friction as possible between your brain and the computer. I think the `tidyverse` mostly gets that right. But the deep learning ecosystem in R is minimal and it's _not_ a general purpose programming language. Elixir is.

And this brings me to my point about the potential blocker to adoption. I [started making noise early] about the need to be able to do the data _manipulation_ tasks leading into and following on from the actual machine learning. They say data science is >90% data 'munging' and time and time again I've found this to be true in my work.

I was thinking of Nx like high speed rail: intercity trains are fantastic, but how do you get to the station and what happens once you arrive? Successful machine learning tooling must provide great "door to door" service. High speed rail without easy travel from home to station and from station to destination won't succeed. So before you start throwing data into a neural net, you need to understand your data, and you need to get it in the right format, which might be distant from how it's provided to you. You may even need to normalise, combine, or engineer variables.

## Enter Rust

Given that I'm the cofounder and CTO of a startup, there was no way I would have the time to write a dataframe library from scratch. But part of my job is keeping across trends and big movements in tech, and I noticed two things happening in my particular little world of interests. The first is that [Rustler](https://github.com/rusterlium/rustler) seemed to be more and more prominent. It's a way to easily write Rust NIFs (Native Implemented Functions) for Elixir. And the other thing was the emergence of [`Polars`](https://github.com/pola-rs/polars), a dataframe library written in Rust. Combine these with the need for dataframes in Elixir and all of a sudden there's a way to make it happen without creating something from the ground up!

It just so happened that someone else had this idea too. I found bindings to `polars` in [`ex_polars`](https://github.com/tyrchen/ex_polars), but the project seemed abandoned. Now's the time to mention I had little to no experience with Rust. But this seemed like a good way to learn! So I picked it up and started running.

I fell down, a lot.

I learned that Rust/Elixir interop isn't always the rosy picture that folks make it out to be. They're rightly very enthusiastic: Rust is an excellent language, `rustler` is a phenomenal project, and writing NIFs without worrying about bringing down the BEAM is an excellent premise. But there are sharp corners and _very_ little documentation. For a Rust novice, I cut myself on those sharp corners a whole bunch. And I'm still rectifying a lot of things in the Rust codebase.

There are some things that I just didn't even realise were possible. As an example, we needed to deserialise dates. I wrote a `NifMap` with a `__struct__` field with a value of `Elixir.Date`. `rust-analyzer` complained, but I kept on. Turns out, [you can just use a `NifStruct` for the core Elixir Date struct](https://github.com/elixir-nx/explorer/pull/143).

What became apparent early on, however, was that building bindings to `polars` provided the opportunity to design an API effectively from scratch on top of an engine with power to spare. Polars is crazy fast. But that speed comes with restrictions and some quirks. That's not to detract from Polars in the least -- it just meant that there was work to do to have my cake and eat it too.

But it was possible.

## The elegance of `dplyr`, the speed of `polars`, the joy of Elixir

I didn't just want "`dplyr` in Elixir". But the premise behind `dplyr`, that of making a 'grammar of data manipulation' -- that's what I'm after!

So before we go any further, let's have a look at the `iris` example above in Explorer:

```elixir
df
  |> DataFrame.select(["sepal_length", "sepal_width", "species"])
  |> DataFrame.filter(fn df -> Series.less(df["sepal_length"], 5.0) end)
  |> DataFrame.group_by(["species"])
  |> DataFrame.summarise(sepal_width: [:max], sepal_length: [:mean])
```

So what do we notice? First, this looks a lot like `dplyr` in terms of verbs. Second, there are callbacks! The second argument in `DataFrame.filter/2` takes a callback function against the dataframe. There's some cool stuff we can do with that, and I think callbacks against the dataframe are a natural way to work. This is especially true for `DataFrame.mutate/2` where the idea is that you're creating new columns by combining existing ones.

This (and the list passed to `DataFrame.select/3` above it and `DataFrame.group_by/2` below it) differ from `dplyr` in that there's no [tidy-select](https://dplyr.tidyverse.org/reference/dplyr_tidy_select.html). This makes things a little more visually cluttered, but more explicit. There are pros and cons, but this is something we'll be looking into in the future.

The next thing is that `DataFrame.summarise/2` looks a bit more like the `pandas` example. This is mainly a restriction of Elixir/Rust interop and the way `polars` is designed. A major hurdle in all of this is that we can't pass arbitrary Elixir functions to `polars`. Ways of working around this and keeping things performant are also in the works.

So why do I think this is a good solution?

### Verbs

Like `dplyr`, `Explorer` is centred around SQL-like 'verbs' of data manipulation. Mainly `select`, `mutate`, `filter`, `arrange`, and `summarise`. There are a few others, but using these verbs along with the ability to `group_by` and `pivot` gives you almost everything you need to massage data and get answers from it.

Reading the code above top to bottom, you know what's happening and in what order. The dataframe is being piped through a series of idempotent functions. Each returns a new dataframe (it would be very surprising to have mutated the underlying dataframe in Elixir, a language built around immutability), which is then piped to the next function. This makes data analysis pipelines eminently composable.

Dataframes are the nouns, and the functions are the verbs. Simple.

### Multiple backends

And that leads us to the thing I'm most excited above. The API above is not tied to Polars. Polars is, instead, the _first backend_ for Explorer. What does that mean?

The Explorer DataFrame (and Series) API is [a behaviour](https://github.com/elixir-nx/explorer/blob/v0.1.0/lib/explorer/backend/data_frame.ex#L1). A behaviour in Elixir is a contract, a way to describe an API in a public way that allows you to fulfill the contract with any module with functions matching the typespecs.

This means that it will be possible to provide a SQL backend for Explorer. Or a [Ballista](https://github.com/apache/arrow-datafusion) backend for distributed compute. Or whatever cool new dataframe library comes out. Or hell, `pandas` if you really want.

And the API will remain the same. This is one of the coolest things about `dplyr` and I think it can be taken even further with Elixir and `Explorer`.

As an example, you could write Explorer analyses that you can apply to tables in a remote database that are computed _on the remote database_ and then you could pull down the results. Or you could even _join_ against a remote table by transferring your local table to the remote database or vice versa.

The same applies to distributed dataframes on Ballista. Or on disk with an OLAP engine. The possibilities are extensive and I'm really excited to see what we can do.

## Downsides

I've alluded to a few problems above. One is in the nature of the Elixir and Rust interop. Dataframes are stored 'Rust-side' as [resource objects](https://www.erlang.org/doc/man/erl_nif.html). That means that they're just pointers to native Rust data structures in Elixir. Serialising and deserialising the dataframes on every function call would be way too much overhead, so instead we pass the pointer around and operate against that.

This is a very cool thing. Especially because those pointers are atomic reference counted, and once there are no more references to them, the memory is dropped. But the Erlang garbage collector may not run fast enough to keep up with the operations you may want to do on dataframes. And in order to keep things immutable in practice Elixir-side, every dataframe operation copies the entire dataframe. As far as I know, nobody has been bitten yet, but this is _not_ ideal for memory useage.

The obvious answer to this for us was to explore the `polars` [lazy API](https://pola-rs.github.io/polars/polars/docs/lazy/index.html). I tried to use the lazy API in an opaque way, hidden from the end user, so that we'd minimise copying and maybe get performance benefits. This ended up being a dead end due to the differences in the way the API is structured. We intend on making lazy opt-in and explicit. There will still be eager-mode footguns, but this should alleviate the issue if you run into memory problems.

The other major challenge is the inability to execute arbitrary Elixir functions 'Rust-side'. That means if you want to apply a function to a `Series`, it has to be serialised into an Elixir list and then deserialised back into a Rust `Series`. Needless to say this is not ideal. I don't have a good answer for this right now. It's no big deal on smaller datasets but for long ones it can be a big performance penalty. If you have ideas I'd love to hear them!

## Next steps

Okay, so you can use Explorer in your workflows _today_. It works and it's _fast_. It's not as mature as `dplyr` or `pandas`, but if you're already working in Elixir you should be able to get work done with `Explorer`. And Elixir is rapidly developing a really phenomenal ecosystem of ML tooling. [Livebook](https://livebook.dev), for example, is the absolute bees knees. I gave a talk at Elixir Melbourne a few weeks back and folks's faces when I clicked the `Run in Livebook` badge and could run documentation locally were priceless.

The immediate next step is to get the lazy Polars backend going. Then I think SQL. Then Ballista. In the meantime, we're [improving](https://github.com/elixir-nx/explorer/issues/169) [the API](https://github.com/elixir-nx/explorer/issues/167), [improving performance through better interop](https://github.com/elixir-nx/explorer/pull/138), and [writing more documentation](https://github.com/elixir-nx/explorer/pull/170).

With the rest of the ecosystem improving by leaps and bounds, Elixir is poised to be a better and better choice for data science all the time. Many of the tools are already there. Come try it out!
