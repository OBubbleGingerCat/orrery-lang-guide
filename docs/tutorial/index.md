# Writing models well

The guide says what Orrery's pieces do. This tutorial says how to put them
together so that a model is:

- **fast**: it takes little time to run, on one thread and on several;
- **readable**: a reader sees one unit's behaviour in one place and has
  few things to keep in mind;
- **faster with more threads**: 8 threads take well under 1 thread's
  time.

Each chapter is one way of writing. It tells you when it applies, what to
write, and how to check that you wrote it right. It also says what the
way does to each of the three, with numbers from an example you can build
and run. Chapter 1 shows how to measure the three on your own model, and
every later chapter uses that method.

## The three measures

| Measure | What | How |
|---|---|---|
| Fast | the time to run, on 1, 2, 4 and 8 threads, building the simulation timed apart | your own driver, each thread count three times, the median ([chapter 1](./measure)) |
| Speedup | 1 thread's time ÷ *N* threads' time, next to the times themselves | the same runs, plus the execution units the load report gives for each thread count |
| Readable | (a) lines; (b) what a reader must keep in mind to follow one unit: its tasks, its gates, who wakes it, and what orders its tasks against other tasks; (c) whether one hardware unit's behaviour is in one place | counted by hand, before and after a change |

## When the three disagree

Compare a change against the model before it, thread count by thread
count. Differences within 10% are noise.

- **Better**: no thread count is slower, and at least one is faster
  beyond noise. Make the change. Say in one line what it does on 8
  threads: faster, the same, or slower.
  - Report the speedup too, but do not reject a change that is faster
    everywhere because its speedup fell. A change that makes 1 thread much
    faster can leave less work to share out. [Chapter 8](./arbitration)
    has such a case.
- **Mixed**: some thread counts are faster and some slower. There is no
  general answer. Choose by the thread count you will run on:
  - if you mostly run on 8 threads, take what is faster on 8;
  - if you run on both, put both times, and what the change does to
    reading, in one table and decide.
- **Easier to read, the same speed**: take it.
- **Easier to read, but slower beyond noise**: write down how much slower,
  and decide.

## How the chapters are written

Every chapter has the same parts:

- the rule: when it applies, what to write, and how to check it;
- an example, small enough to read whole, that builds and runs;
- what the rule did to the three measures on the example, and on the
  GPU model that comes with Orrery where there is a measured case;
- when not to use it;
- a checklist.

**The examples** are one Cargo package. Its library is the measuring
helper from chapter 1, and there is one binary per chapter. To run one:

```sh
cargo run --release --bin ch1_measure
```

Each example prints a line per thread count: the run time, the speedup,
the build time, the execution units, and a checksum. The checksum must
not change with the thread count, and it must not change between two
ways of writing the same thing.

**The numbers** in this tutorial are rough unless a chapter says
otherwise. Each is the median of three runs, on 8 cores that nothing else
was using. Your machine will give other times; compare the ratios.

## Chapters

1. [Measure first](./measure)
2. [Close the gate of an idle unit](./gates)
3. [Close until the next deadline](./due)
4. [Wake in time](./wakes)
5. [Lines and delays](./lines)
6. Split, then merge *(to come)*
7. How to lay out a unit's state *(to come)*
8. [Arbitration across modules](./arbitration)
9. Event logs *(to come)*
10. [Getting faster with more threads](./threads)

Then [the whole checklist](./checklist): every chapter's checks in one
list, to go through when a model is written.

Chapters 6, 7 and 9 wait for the next version of the GPU model, whose
SM they describe.

**Before you start**, read these pages of the guide:

- [Getting started](../guide/getting-started), and run the first model;
- [Modules](../guide/modules);
- [State and tasks](../guide/state-and-tasks);
- [Time](../guide/time);
- [Components](../guide/components);
- [External ports](../guide/external-ports), to feed a trace in and
  take results out;
- [Running](../guide/running).
