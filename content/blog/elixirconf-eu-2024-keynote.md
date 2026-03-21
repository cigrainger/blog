+++
title = "What I mean when I say that machine learning in Elixir is production-ready"
date = 2024-05-09
description = "Reflections on my ElixirConf EU 2024 keynote and why the BEAM is a secret weapon for ML."
[taxonomies]
tags = ["elixir", "machine-learning", "nx", "talks"]
+++

My ElixirConf EU 2024 keynote ["Ship it! A roadmap for putting Nx into production"](https://youtu.be/5FlZHkc4Mq4?si=agofn18zjoKHZw6S) is up on youtube. I'm hoping it will help folks gain the confidence to go out and... ship it!

I often say 'machine learning in Elixir is ready for production'. On further reflection, I realised that this statement dramatically undersells the true potential and capabilities of machine learning within the Elixir ecosystem. (I already posted these reflections [on Twitter](https://twitter.com/cigrainger/status/1788147429021237592)).

<!-- more -->

When I say "production-ready," it's not just about the ability to deploy machine learning models in a live environment. In the context of Elixir, being production-ready means deep integration with the weird and wonderful thing called the BEAM (Bogdan/Björn's Erlang Abstract Machine). It means seamless compatibility with OTP (Open Telecom Platform) primitives, which are the building blocks of Elixir applications.

This deep integration is what makes machine learning in Elixir so powerful. Just as Elixir has proven to be a cheat code for building web applications and distributed systems, it now offers the same advantage for machine learning tasks. The Elixir ecosystem provides a secret weapon that empowers developers to build and deploy machine learning solutions with unparalleled ease and efficiency.

For a growing set of areas where Elixir's machine learning ecosystem is applicable (which is already pretty extensive), I firmly believe that you're better off shipping inference with Elixir than basically anything else. In many cases, it's even worthwhile to conduct experiments and model training in Elixir, despite the relative youth of that part of the ecosystem.

So, what makes machine learning in Elixir so powerful? Mainly: OTP. Also: good choices. [Nx](https://github.com/elixir-nx/nx) was built from the ground up with inspiration from JAX, a popular machine learning framework. By leveraging metaprogramming and embracing pluggable backends and compilers from day one, Nx benefits from a significant second-mover advantage. This architectural decision, which I covered in more detail during my ElixirConf EU talk, unlocks huge potential that we see play out in the libraries built on top of it like [Axon](https://github.com/elixir-nx/axon), [Bumblebee](https://github.com/elixir-nx/bumblebee), and [Scholar](https://github.com/elixir-nx/scholar).

Thanks to these foundational choices, the entire Elixir machine learning ecosystem can reap the benefits. With `defn`, a macro provided by Nx, machine learning capabilities are practically built into the language itself. Take a moment to reflect on the significance of this: by simply writing Elixir code, you gain access to multi-stage compilation that targets various hardware platforms, including GPUs and TPUs.

Moreover, [`Nx.Serving`](https://hexdocs.pm/nx/Nx.Serving.html), a module in `Nx` for serving machine learning models, is just wild. You get out-of-the-box distributed, clustered, hardware-agnostic automatic batching without the caller needing to concern itself with any of that? Just wild. It's the right separation of concerns.

The actor model of concurrency, which is at the heart of Elixir, proves to be the perfect abstraction for serving machine learning workloads and, just as importantly, integrating them into broader systems. By encapsulating models within processes, you can leverage all the built-in features that make the BEAM so powerful. `Nx.Serving` is just another application in your supervision tree, making it just another component of your robust and fault-tolerant system.

So integrating machine learning into a Phoenix application becomes a breeze. You can take advantage of libraries like Oban for durable and robust job processing, Broadway for concurrent data processing pipelines with backpressure, and FLAME for lambda-like execution on the BEAM. You can send pubsub messages about progress, results, etc to LiveViews (which are, of course, just processes). And of course those updates can trigger changes in views like I describe in [this thread](https://x.com/cigrainger/status/1788490429341532351).

The beauty of the Elixir ecosystem lies in its ability to create brilliant features for end users with minimal effort. We routinely handle thousands and millions of messages per day over pubsub and between processes, showcasing the scalability and efficiency of the system.

You can add machine learning to Saša Jurić's famous chart. So if Rails is considered the "one person framework," then Phoenix can be seen as the "half person framework," from basic CRUD operations to the cutting edge of LLMs.

Machine learning in Elixir is production-ready. In this ecosystem, that statement means a lot.

{{ youtube(id="5FlZHkc4Mq4") }}
