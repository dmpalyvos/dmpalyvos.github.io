---
layout: post
title: The problem definition is (half) the battle
---
What is the scariest thing you can hear at work on a Friday afternoon? 

For me it is the question: 

> *How did we get these numbers?* 

Especially when followed by some mumbling 
about an uncommitted shell script on someone's laptop
that produced a text file. You know, the text file that was 
copied into a spreadsheet to generate a plot that was then pasted 
into Word.

Oh, and by the way, the script is gone now. But don't worry, so are the input
files. And I might have tweaked the formulas before (or was it after?) generating the plot. 
I really wanted to try something out!

Sorry, what was the question again?

----

This is a story about my struggle with this question over the years.
It's also a story about defining complex problems and what that means
for solving them.

This story follows my journey towards benchmarking provenance, from 
scattered spreadsheets and tar files to automated, systematic,
and reproducible benchmarking.
It explains how today, at my current job at ZeroPoint, anyone can run
complex, reproducible benchmarks of our hardware at the press of a button.

Do you also hate stressful questions on Friday afternoons? If yes, read on!

## New product, new me

It all started with a new product. Maybe it always does. At ZeroPoint, we 
were developing a compression hardware IP for expanding the capacity of CXL memory
and we were soon going to evaluate it.
Since CXL memory is about exposing a new memory tier[^1],
we wanted to run real workloads in a full system and measure how their performance
was affected by our compression hardware. 
However, it quickly became clear that the
evaluation dimensions were almost endless. You have the configuration of the
hardware IP with tens of parameters like version, compression algorithm, tuning
parameters, etc. Then there's the firmware with its own set of parameters. 
And we haven't
even started considering the host system setup, the input files used by the
workloads, the build configurations and versions of the workloads, the kernel configuration,
the CPU frequency, etc. You get the idea.

Of course, one way to go through life is to ignore problems until they hit you in
the face. For this particular problem, ignoring it could mean just using
spreadsheets. At first glance, they don't sound so bad, right? 
Whip up a script that outputs measurements in CSVs, then plug those into
a spreadsheet and I can get you the first plot by lunch. And if your spreadsheets
are in the cloud, you can easily share the numbers with anyone. Not
a bad pace, right? Well WRONG. 
If you take away one thing from this article, let it be that
*spreadsheets are one of the biggest traps in benchmarking*.
The trap is simple: spreadsheets make it very easy to skip any kind of provenance. 

By **provenance**, I mean information about
how some specific measurement came to be. 
Such information allows you to:

1. reproduce the measurement and get the same result
2. if a measurement is wrong, understand why that might be the case.

Spreadsheets are bad for provenance because they are *out-of-band*. You run
your experiment in place A and then you copy the results into a spreadsheet in place B.
Even if you are diligent and you include some manual provenance, 
e.g., the file path of your input
data in the spreadsheet, there's no guarantee that someone won't edit
that file tomorrow. More importantly, nobody can use that spreadsheet alone
to rerun the experiment and get the same results.

With this benchmarking conundrum at hand, I was called in to find a better
alternative. The problem was that I was unsure about what the problem was, or
what a good solution would look like. Thankfully, I wasn't starting from scratch. 
I struggled with benchmarking provenance a lot during my PhD and I never found a 
fully satisfactory solution, so it was kind of a nemesis that came back to haunt me. 

The ghost of Christmas provenance.

## Me, myself, and my provenance

When you do a PhD, you have to do a lot of experiments. If you're
good and a bit lucky, you get some nice results that you can use
to write a publication. Then you submit this publication and you wait a
few months for peer review. When the reviews come, you usually need to go
back to your results and use them to answer reviewer questions. 
That involves the raw data you used to make the figures in the paper. 
In some (unfortunate!) cases, the reviewers might ask you for more experiments. 
If your experiments are not reproducible, this
process can quickly turn into a nightmare.

So I had a method. It was not great and it felt like a house of cards but it
served me well for five years without any hiccups, so I'll tell you all
about it. Or, even better, let's design it from scratch together. 
In the simplest possible terms, an experiment is some code, some binary artifacts, and some
measurements[^2], right?
- *What system can keep track of code?* Git.
- *What system can keep track of measurements?* Spreadsheets.
- *What system can keep track of binary artifacts?* Many. Let's say tar files.

Great! We have our system. Commit the code. Run the
experiment. Pack everything into a tar archive. Put the measurements, the
path to the tar file and the commit hash of the code into a spreadsheet. If you're
feeling fancy, you can also record the IDs of the relevant paper figures
in another column. For example:

|Commit|Date|Raw Data|Rate|Latency|Figures|
|---|---|---|---|---|---|
|14f3d87|2026-10-03|~/paperA/exp.tar|30 GB/s|990 ms|1, 3|

BOOM. We have provenance!
So why not use something like that now at ZeroPoint? It obviously worked for me before and
required very little effort to set up. What could go wrong?

## Social experiments

Spoiler alert: what could go wrong is *other people*. 
A whole team would now be involved in the benchmarking of the product.
How long do you think it takes before the spreadsheet starts falling apart? 
Before people forget to add data, or add some wrong data, or overwrite 
other people's data?

I am a practical person. I considered the option of moving to a remote island
and avoiding other people and the pesky problems they cause. But I persevered.
And I started realizing that I had just stumbled upon the most important 
aspect of the problem: *control*.

More precisely, the *lack of control* that people should have over the experiments.
People make mistakes. When those mistakes pollute the provenance itself, it
can be impossible to go back. So let's not allow them to.

1. How do you remove control during experiment runs? 
 *Automate* everything.
2. How do you prevent someone changing the data of the experiments later? 
 Make the experiments *immutable*. 
3. How do you prevent someone overlooking some dimensions of the experiment? 
 Make the process *systematic*.

That sounds promising. Spreadsheets did have one advantage, though. 
They were easy to share both with
technical and non-technical people. It would be very nice to keep that property
in the new solution.

To summarize, I wanted automated, immutable, reproducible experiments with
systematic tracking of their dimensions. And easily shareable too. Shouldn't
be hard to do, right?

## Version my data, please

And indeed, it wasn't. Having defined the problem clearly, the solution 
came almost organically. So organically that I don't really remember 
exactly how. With the requirements written down, it would not have been difficult 
to create the infrastructure myself. But I 
was lucky enough to have someone else do my work for me. In my book, that's the best kind
of solution!

More specifically, the solution I chose came in the form of a framework called 
[DVC](https://dvc.org/), which stands for *Data
Version Control*. Even though it is marketed for AI/ML and data science, it
allowed me to model the benchmarking problem and gave me a very elegant base for
a full provenance solution. DVC defines what an experiment[^3] is and models its
provenance in a systematic way. All that remained was to orchestrate
the automated experiment execution and result collection. 

In the end, DVC augmented with my own infrastructure made hardware benchmarking a 
single-command procedure.
Run your experiment and its data is stored immutably in an artifact store,
while its measurements are pushed to a database and visualized in a dashboard.
Nowhere in the process does a (non-malicious) user have the chance to leave out any
part of the experiment configuration or change the results. What you see is what you get.

I cannot overstate the peace of mind that such infrastructure gives to a team. 
Being able to *prove* which configuration was used to produce a specific measurement 
and what post-processing was applied afterward can dramatically increase the confidence
in the results. This, in turn, frees up time for things that matter:
more efficient execution, more experimentation, and more investigation of the 
performance of the product itself.

## Solving problems by defining problems

Reflecting on this journey, I come back to something that I have been
considering a lot over the years. The more experience I gain, the more it seems that
most of my job is about clearly defining the problem I am trying to solve. 

How I arrive at that definition varies, depending on the type of problem.
In this case, it helped to reflect on what I had tried before and why it couldn't work now. 
By the way, such reflection also works great at the team level. Ask *"why can't [the simplest
solution] work in our case?"* and then try (really try!) to answer this question honestly.
You might be surpised with the answers you end up with.

When I work on R&D projects, I tend to define the problem by
trying to solve it. Sounds counterintuitive, but the point is not to 
succeed. The first attempt usually fails, 
but the failure itself usually reveals aspects of the problem 
not considered before. I feel that this approach works better in
R&D projects because there are too many "unknown unknowns" for 
reflection to give any useful results.

Whatever technique you use, I think that the skill of clearly defining problems is
more important than ever. Nowadays, AI can solve a lot of engineering problems.
With the correct guidance, it can help you solve some really hard ones too. 
But in my experience, if you don't define your problems clearly, AI is not much help.

Without a good problem definition, AI can even make things worse, because it deprives 
you of the step-by-step problem-solving process that could have revealed 
a wrong problem definition.
When AI inevitably gives you an intricate solution for a different problem than the one you had,
you have wasted time, money, and energy that could be better spent elsewhere.

So what are you still doing here? Go out there and find some problems to define!

---

[^1]: Volatile CXL memory works like regular RAM as far as userspace programs are concerned (exposed as a CPU-less NUMA node) but it is usually slower. For this reason, it is frequently used together with OS-supported memory *tiering*, which moves less frequently accessed memory pages to CXL.

[^2]: We'll ignore the environment for now: Assume you're a PhD student working alone; you have your own environment that nobody touches so that's fine.

[^3]: The way it works is pretty smart. At a very high level, DVC hashes all declared inputs and outputs of the experiment and saves those hashes into a (text) lockfile. For each experiment, this lockfile is saved in a Git commit created by DVC. The binary data is stored in a content-addressable cache and can be uploaded to S3, SSH, and other types of remotes. Checking out an old experiment brings in the relevant lockfile and in turn the data referred by it.