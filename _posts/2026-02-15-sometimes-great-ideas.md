---
layout: post
title: Sometimes great ideas are just there
---

### The Nightmare

> It's 3:00 AM. You are woken up by alerts from your data pipeline about a neighborhood-wide blackout.
> You look out the window and you see the street bathed in the yellow glow of the street lights.
> The neighborhood clearly has power. You are starting to sweat profusely.
> Is it a bug in the query? A bug in the firmware? Faulty sensors?
> 
> You try to look at the raw data, but the previous inputs have already been dropped 
> from the limited storage of the smart energy meters.
> Your sweat is starting to run cold. What do you do?

The above might not just be a bad dream. 
In fact, such "nightmare" scenarios were the motivation for my research 
in explainable stream processing systems.
When you are processing millions of datapoints per second, can you go back 
and explain why a result is not what you expected? 
Is it even possible? And can you do it without destroying the performance of your system?
Let's find out!

### Data Streaming and Explainable Outputs

Data streaming queries extract low-latency insights by processing data items (*tuples*) in near real-time.
In contrast to relational databases where the data is "stationary" and users run queries on it, 
in data streaming there are long-running queries over streams of fast-moving data.
Those queries use transformations called *operators* (e.g., filter, join, aggregate).
In our nightmare scenario, smart meters in each building or city block could be running
a continuous query that processes per-second voltage and power measurements
and produces an out-of-power event if some conditions are met.

When something goes wrong in a streaming query, it can be critical to know why a specific output was produced. 
This question is answered by *fine-grained data provenance*,
which returns the set of the inputs that contributed to the production of an output event.
Provenance was not a big area in data streaming research in 2018,
when I was six months into my PhD.
At that time, I was taking slow steps on another topic, 
custom CPU scheduling for stream processing engines.
My supervisor, [Vincenzo Gulisano](https://vincenzogulisano.com/),
had come across a recent paper on data provenance
and thought we could come up with a better technique.
Looking back, I think he just wanted me to get started on paper writing and performance evaluations.
Maybe he was hoping that we would get a small publication out of it 
but without huge expectations for groundbreaking work.

That previous provenance [technique](https://dl.acm.org/doi/abs/10.1145/2488222.2488256) 
was a simple but functional idea:
- It annotated each tuple with a unique ID.
Every time a new tuple was produced, 
it carried the tuple IDs of all of its parents. 
- This way, input tuple IDs were propagated downstream to the outputs.
- Provenance was computed with a streaming join between
all the inputs and all the outputs: each input whose ID matched an annotation present 
in an output was part of that output's provenance.

However, we identified several issues:
1. The join could be **expensive**, considering that each output could depend on hundreds or more raw inputs.
2. The size of the ID annotations carried by each tuple could grow **arbitrarily large** (depending on query semantics). In practice, the memory usage of the stream processing engine could explode when recording provenance with this technique.
3. **All input data** had to be kept until it was joined with all possible results. This could be impossible in resource-constrained devices or high-throughput deployments.

The question was, could we do better?

### The Lightbulb Moment

Vincenzo's idea was to avoid the variable-length annotations and the join.
This could be done by constructing a Directed Acyclic Graph (DAG) for provenance. 
- The nodes of the **provenance graph** would be the tuple objects. 
The edges would be object references to "parent" (upstream) tuples.
Each output tuple could traverse the graph upstream to find its parents, 
eventually discovering its ancestral input tuples.
- **A join would no longer be needed**. The graph 
would continuously maintain the relation between inputs and outputs 
throughout the query. 
- As a bonus, it was possible to create the graph **without variable-length annotations** 
by taking advantage of the semantics of the query operators.
Each tuple required some small, constant-size metadata.

With the variable-length annotations and the join solved, 
we just needed to fix the last problem: 
not maintaining all query inputs.
However, when doing aggregations and joins, it is not trivial to decide
when an input has finished contributing to potential outputs somewhere in a streaming query. 
To reduce the problem space, we were brainstorming various solutions, 
such as traversing the graph for each tuple
to somehow mark tuples that were not part of any output's provenance.
It was then that I had my "aha moment". 

Having worked in large Java codebases before, 
I always had the garbage collector and object references in the back of my mind, 
mostly for performance.
As we were discussing reference-based graphs, 
something clicked.
Overwhelmed by anticipation, I told Vincenzo (with all the nervousness of a first-year PhD student),
*"Wait a minute... What if the system threw away the garbage tuples on its own? I think our problem is already solved!"*

Here's what I really meant:

> The query engine we were using ([Apache Flink](https://flink.apache.org/)) runs on the Java Virtual Machine (JVM).
> On a high level, the JVM *garbage collects* objects that are not reachable from anywhere in the program. 
> In our case, input tuples that did not contribute to any output would not be connected in the provenance graph and thus would eventually become unreachable.
> 
> Thus, our provenance graph would ensure that **only our provenance inputs would be kept alive in the memory of the process**. Other irrelevant inputs would be discarded automatically and transparently by the memory management.
> Since the provenance graph had no cycles, the same technique would work in languages that use plain reference counting instead of garbage collection, such as C++ (`shared_ptr`) and Rust (`Rc/Arc`).

This was great! We could take advantage 
of decades of optimizations in garbage collection to clean up our query inputs
and we did not need to dive deeper and analyze how long each input could live for[^1]. 
When we implemented and evaluated our provenance method, 
which we named [GeneaLog](https://dl.acm.org/doi/10.1145/3274808.3274826) (hinting at the tuple ancestry),
the results were impressive: we could compute live, eager data streaming provenance with overheads lower than 15% in most cases (where previous techniques would cause very significant slowdowns).

Taking inspiration from the previous streaming provenance work, I implemented the framework to *instrument* (wrap) existing query operators. 
This meant that there was no need for a custom version of the stream processing engine to compute provenance. 
It was enough to update your query to use our framework and you were set.
An implementation of GeneaLog is available on [GitHub](https://github.com/dmpalyvos/ananke). 
The repository contains an extended version of the technique that includes [Ananke](https://vldb.org/pvldb/vol14/p391-palyvos-giannas.pdf), my later work on forward streaming provenance. 

### What Does This All Mean?

As I reflect on that day, I am trying to explain the feeling of excitement
when a great idea just "clicks" into place.
That feeling is often disproportionately stronger than the idea's objective value.
Maybe it doesn't come from the idea itself, 
but from our ability to synthesize 
seemingly unrelated tidbits of knowledge acquired sometime *before*
and combine them with patterns we recognize in the *now* 
to **create something new**.

That feeling says that you do not need to plan everything. 
You do not need to constantly judge the things you read, watch, learn, 
based on some perceived *usefulness*.
You cannot really predict what could lead you to your next great idea. 
You're allowed to pursue that "pointless" project, to read that "useless" book,
even to sit on the couch and do nothing. 
Maybe if you give your mind enough space,
you will find more moments 
where great ideas are, indeed, just there.

As for me, that simple insight led to a conference paper (and later a journal extension), 
and eventually two more provenance papers at VLDB. 
More importantly, it changed how I build systems. 
Today, I've become a bit of a provenance evangelist in my current work,
constantly asking: *"What is the source of truth for this data and how do we keep it?"*. 

But that is a story for another day.

[^1]: This problem of figuring out when input tuples are no longer *live* becomes important the deeper you dive into streaming provenance. I did not manage to avoid this problem for long and it became a large part of my later PhD research.
