# Running a simulation

## From plan to simulation

```rust
use orrery_lang as orr;

let plan = orr::SimPlan::<Top>::new(Some("params.toml".into()), None)?;
let mut sim = orr::Sim::new(plan, orr::SimConfig::default())?;
```

- `SimPlan::new(params, snapshot)` elaborates the model and checks its
  connections. Both arguments are optional paths: without a parameter
  file every parameter keeps its default; with a snapshot the simulation
  starts from it ([Snapshots](./snapshots)).
- `Sim::new(plan, config)` loads the simulation. It works out how the
  model's tasks are ordered, and rejects a model whose order would be
  ambiguous ([Errors](./errors#when-the-simulation-loads)).
  `Sim::with(plan, config, event_log, |top, ports| ..)` takes two more
  arguments: a sink for the [event log](#event-log), or `None`, and a
  closure that binds the [external ports](./external-ports) of a model
  that has them.

## Configuration

The fields of `orr::SimConfig` a model's user sets:

| Field | Default | Meaning |
|---|---|---|
| `workers_per_group` | 1 | the threads that run the simulation: 1 runs it on the thread that calls `run_until`; *N* ≥ 2 runs one thread that hands out the work and *N* − 1 workers that run the tasks, so 2 is not faster than 1 ([Threads](#threads)); 0 fails the load |
| `groups` | 1 | the groups of threads; for now 1 is the only number that loads |
| `checks` | `false` | record the order in which tasks access state, for `sim.check_trace()`; for tests: what it records grows with the run's length, in memory and in time |
| `delta_limit` | 30 | the most delta cycles at one physical time, at most 62; more is an error ([Delta cycles](./time#delta-cycles)) |
| `snapshots` | `None` | when and where to take [snapshots](./snapshots) |
| `event_capacity` | 2²⁰ | the most events in flight from one event source of one instance: sent, and not yet arrived; more is an error |
| `fail_on_drop` | `false` | an event that reaches a trigger while its gate is closed fails the run, rather than being dropped and counted ([Clock gates](./time#clock-gates)) |
| `reports` | `orr::Reports::Warnings` | what the simulation prints to stderr: `Warnings`, the [warnings](./errors#warnings) of the load and, as it is dropped, the count of events dropped at closed gates; `Off`, nothing; `All`, also a report of the load and of each stop |
| `cut` | `orr::CutPolicy::Runs` | how a run on several threads divides the instances of one kind of module among its execution units: `Runs` in runs of neighbours, `Dealt` dealt out one by one ([When to deal the instances out](#dealt)) |
| `cpus` | `None` | the CPUs a run on several threads may take its cores from; `None` for any the process may run on ([Threads](#threads)) |
| `host_spin` | 1 ms | how long the thread that calls `run_until` spins, waiting for a run on several threads to finish, before it sleeps; `0` sleeps at once. Set it and `idle_spin` to `0` when the process may use only as many CPUs as the run has threads ([Threads](#threads)) |
| `idle_spin` | 1 ms | how long a run's own threads (the one that hands out the work, the workers, the event log's) spin between calls before they sleep; `0` sleeps at once. Set it to `0` with `host_spin` ([Threads](#threads)) |
| `slots` | 16 | keep the default |

None of these changes results, as long as a run stays within
`delta_limit` and `event_capacity`, except `fail_on_drop`: with it, a run
that would drop an event fails there instead. Those three change results
the same way on any machine and on any number of threads. `SimConfig` has
two more fields, `timings` and `pause`; leave them at their defaults.

Fields will be added. A configuration written with
`..orr::SimConfig::default()` keeps compiling:

```rust
let config = orr::SimConfig {
    workers_per_group: 4,
    delta_limit: 50,
    ..orr::SimConfig::default()
};
```

## Threads {#threads}

`workers_per_group: N` runs the simulation on *N* threads. With *N* = 1,
the thread that calls `run_until` runs every task. With *N* ≥ 2, one
thread hands out the work and *N* − 1 workers run the tasks, while the
thread that calls `run_until` starts the run and waits for it. Two
threads therefore run the tasks on one worker, as one thread does, and
pay for running on several: tasks run in parallel from three threads on.
A run on several threads with an [event log](#event-log) has one more
thread, which writes the log. The number of threads never changes
results.

**Cores.** A run on *N* ≥ 2 threads keeps each of them on a physical core
of its own, all in one cache domain (the cores that share one L3 cache),
and its workers wait for work by spinning, not sleeping. Give it cores of
its own: another program on the same cores makes it markedly slower and
its times unsteady. `cpus: Some(vec![..])` limits the CPUs it may take
cores from; the two hardware threads of one core count as one core. A run
on one thread is not pinned and does not read `cpus`. When no cache domain
has *N* free physical cores among those allowed, building the simulation
fails and says which.

**The thread that calls `run_until`** is not one of the *N*. While a run
on several threads goes on, it waits: it spins for `host_spin` (1 ms by
default), then sleeps, and spinning takes a CPU. Between calls it is the
other way round: the run's own threads spin for `idle_spin` (also 1 ms),
then sleep, while that thread does what the program does between calls.
When the process may use only as many CPUs as there are threads (8
threads, say, where the process may run on 8 CPUs, one of each core; or
2 threads on 2, 4 on 4), that thread has no CPU of its own: whichever
side spins takes a CPU the other needs, and the run is slower. With
`run_until` called every thousand cycles it took up to four times as
long; in a model whose work comes in short bursts, about twenty times as
long, even when the whole run was one call. Set both to zero then,
however long each call runs:

```rust
let config = orr::SimConfig {
    workers_per_group: 8,
    host_spin: std::time::Duration::ZERO,
    idle_spin: std::time::Duration::ZERO,
    ..orr::SimConfig::default()
};
```

That brought the runs back to within a few percent of their time with
CPUs to spare, a tenth for the bursty model; calls every few cycles
still took up to a fifth longer. With `host_spin` alone at 0 those took
up to 1.4 times as long, and setting both was never slower beyond the
scatter between repeated runs. Count the CPUs the process may use: a
core's second hardware thread that is idle but not among them does not
help. Limiting `cpus` alone, while the process may still use other CPUs,
needs no change. With CPUs to spare, leave both as they are: there,
sleeping at once gains nothing, and a thread that sleeps must be woken.

**A simulation holds its cores until it is dropped**, not only while it
runs. To run one simulation after another in the same process, each on
several threads, let the first go out of scope (or `drop(sim)`) before
building the next; otherwise its cores are still taken, and building the
next fails as above. Binding the next simulation to the same name does not
drop the first. Each run in a block of its own:

```rust
let config = orr::SimConfig { workers_per_group: 4, ..orr::SimConfig::default() };
{
    let plan = orr::SimPlan::<Top>::new(Some("small.toml".into()), None)?;
    let mut sim = orr::Sim::new(plan, config.clone())?;
    sim.run_until(orr::Time::us(10))?;
} // the first simulation is dropped here, and its cores are free again
{
    let plan = orr::SimPlan::<Top>::new(Some("large.toml".into()), None)?;
    let mut sim = orr::Sim::new(plan, config)?;
    sim.run_until(orr::Time::us(10))?;
}
```

**Several processes.** A simulation in another process does not see the
cores this one holds, and may take the same ones. To run several at once,
each on several threads, give each its own `cpus`, with no CPU in common
(`lscpu -e` gives each CPU's L3 cache, last in its `L1d:L1i:L2:L3`
column):

```rust
// Where CPUs 0-7 and 8-15 are two L3 domains: in one process,
let config = orr::SimConfig {
    workers_per_group: 4,
    cpus: Some((0..8).collect()),
    ..orr::SimConfig::default()
};
// and in another, at the same time, cpus: Some((8..16).collect()).
```

**What a run on several threads does not take yet.** Two things in a
model fail the load on 2 or more threads (`LoadError::Config`, field
`parallel`), and load on one: a [clock gate](./time#clock-gates) that a
task [wakes](./time#wakes) and that is read in another execution unit
than its owner's; and a state location that two tasks may write at one
timepoint, where the model does not order the two.

**How much runs in parallel.** As the simulation is built, Orrery divides
the model into **execution units**: groups of calls that always run
together, on one worker at a time. A model divides into more of them the
more tasks it has, and the more instances of one kind of module (at least
as many as there are workers, to keep them all busy). Calls go into one
unit when they are the tasks of a module that has a single instance; when
events or wakes without delay run in a ring between them; when several
senders send to one event location; and when, within one timepoint, they
are ordered against each other in a cycle.

`sim.load_report().units` is the number of execution units. When there
are fewer than workers, building the simulation warns:

```text
warning: group 0: 2 execution units for 3 workers
```

At least one worker then has nothing to do, though it still holds its
core; results are unaffected. `1 execution unit` is a serial run. Which
tasks went into which unit cannot be listed yet.

### When more threads help {#when-threads}

More threads make a run faster only when the cycles that have work have
enough of it, in enough execution units, to keep the workers busy. Two
things are paid for besides the work:

- **Two threads run the tasks on one worker.** With
  `workers_per_group: 2`, one thread hands out the work and a single
  worker runs every task, one after another, as one thread does; the
  handing out comes on top. Two threads are not faster than one. Tasks
  run side by side from three threads on, on two workers.
- **Each execution unit costs something in every cycle**, besides the
  work of its tasks, whether its gates are open or closed: at least a
  few hundredths of a microsecond (rough; it varies with the model and
  the machine). Only stretches in which no task of the whole model runs
  are skipped. When the tasks do little, that costs more than running
  them side by side saves, and the run is slower than on one thread.
  Results are the same either way.

Three things about the model decide it:

- **How many instances have work in a typical cycle.** Count the ones
  that do something, not the ones that are built: a cache with 64 banks
  of which 4 are busy in a cycle gives 4 workers something to do, not 64.
  With one instance at work, every thread past the first only costs.
- **Whether those instances are neighbours.** The instances of one kind
  of module are divided among the execution units in runs of neighbours:
  of 64 instances and 7 workers, about the first 9 in one unit, the next 9
  in the next, and so on. When the few instances with work are
  neighbours, they are in the same unit and run one after another on one
  worker, however many threads there are ([dealing them out](#dealt)
  helps).
- **How much the working instances do in a cycle**, against what an
  execution unit costs. More kinds of task, and chains of events without
  a [declared delay](./time#declared-delays), give more units. On one
  thread a closed gate costs nothing.

### When to deal the instances out {#dealt}

Try `cut: orr::CutPolicy::Dealt` when few instances of one kind have work
at a time and they are neighbours. A dispatcher that hands out work
round-robin, starting after the last one it gave work to, does this when
there is little to hand out: the instances it keeps busy are the next
few in a row. `Dealt` deals the instances out, the *i*-th into the
(*i* mod *k*)-th of *k* units, so that neighbours go to different units
and run side by side. It changes no result.

Where every instance has a little work in every cycle, dealing out can
be slower than the default, `Runs`: by a few percent to two fifths, the
more so the less each instance does. Measure both. On a GPU model's eleven
test programs, on 8 threads and timed as whole processes, dealing out
took 0.71–0.89 of the default's time on the programs where only a few
cores have work at a time, and 2–7% more on those where every core works
in every cycle.

Two things cannot be seen yet. No report lists which instances went into
which unit, so whether the busy ones share a unit is known from the
model, not from Orrery. And `Dealt` can end with fewer units than
`Runs`, for reasons no report gives either: read
`sim.load_report().units` under both before comparing the runs.

### How many threads {#how-many-threads}

Choose by trying. Take a stretch of a representative input, run it once
on 1 thread and once on the most threads, and keep the faster:

- The most threads are as many as the free physical cores of one cache
  domain ([Threads](#threads)). If those are every CPU the process may
  use, set `host_spin` and `idle_spin` to 0 first. When few neighbouring
  instances work at a time, run it on the most threads with
  `cut: orr::CutPolicy::Dealt` as well ([above](#dealt)).
- Compare the runs alone, the `run_until` calls, not building the
  simulation: building on several threads takes longer, and on a run of
  a fraction of a second that can decide which is faster for the whole
  process.
- The counts in between were never the fastest in the models measured;
  try fewer only to leave cores for something else.

Try it on each kind of input the model runs. One model can come out on
opposite sides with another input: a model that ran faster on 8 threads
than on 1 on an input with much work in every cycle ran slower on an
input with little. So no size of model, and no number of execution units,
says in advance which is faster.

As a rule, a small model, or one with little work in each cycle, runs on
1 thread, the default. Several threads pay off on large runs: many
instances, each with much to do in a cycle.

## Running

`sim.run_until(time)` runs every timepoint up to and including `time`, all
of its delta cycles included, and returns how it ended:

```rust
match sim.run_until(orr::Time::us(10))? {
    orr::RunEnd::Reached(time) => println!("reached {time}"),
    orr::RunEnd::Finished(finished) => println!("stopped at {}", finished.at),
}
```

Runs can be repeated: `run_until(1 µs)`, then `run_until(2 µs)` continues
from where the first stopped, and gives the same result as running to
2 µs at once. A run reaches about 288 s of simulated time at most: a
`run_until` past that, or to a time that is not physical, fails with
`RunError::Request`, runs nothing, and leaves the simulation to go on
([Time](./time#delta-cycles)).

`sim.step()` runs exactly the next timepoint and returns
`StepEnd::Executed(timepoint)`, `StepEnd::Finished(..)` or
`StepEnd::Idle` when nothing is left to run. `sim.next_timepoint()` says
which timepoint that would be.

A timepoint is a time at which a task runs, or at which an event is
dropped at a closed [clock gate](./time#clock-gates). A clock edge at
which every gated clock is closed is not one, and neither is the arrival
of an event on a source that triggers no task: `step` passes over them,
and `next_timepoint` does not report them.

`sim.now()` is the last timepoint that ran, or `None` before the first.
It is not how far the simulation has run: after `run_until(1 µs)`,
`now()` is the last timepoint at or before 1 µs at which something
happened.

## Reading and writing state

Between runs, the host reads state through the handles in `sim.top()`,
the instance tree:

```rust
let total = sim.top().workers[0].sink.total;
println!("{}", sim.state(total)?);
*sim.state_mut(total)? = 0;
```

`state` reads a value; `state_mut` writes one, and is also how the host
reaches stores and external sources. A [clock gate](./time#clock-gates)
is opened and closed with `sim.gate_mut`. What the host writes is seen
from the first timepoint of the next run.

## Stopping

A task that may end the simulation returns `Result<(), orr::Stop>`:

```rust
#[orr::module]
pub struct Countdown {
    #[orr::param("ticks before stopping", default = 5u64)]
    limit: u64,

    #[orr::trigger]
    clock: orr::Clock,

    ticks: u64,
}

#[orr::module_impl]
impl Countdown {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))
    }

    #[orr::task(triggers = (clock,))]
    fn count(ctx: &mut orr::TaskCtx, ticks: &mut u64) -> Result<(), orr::Stop> {
        *ticks += 1;
        if *ticks == *ctx.module().limit {
            return Err(orr::Stop::finish(format!("counted to {ticks}")));
        }
        Ok(())
    }
}
```

```rust
match sim.run_until(orr::Time::ns(100))? {
    orr::RunEnd::Finished(finished) => {
        for finish in &finished.by {
            println!("{} at {}: {}", finish.path, finished.at, finish.message);
        }
        println!("ended at {}", finished.end);
    }
    orr::RunEnd::Reached(time) => println!("reached {time}"),
}
```

```text
top at 4ns: counted to 5
ended at 4ns
```

A stop returned at *T* ends the simulation at *T* + *L*, where *L* is the
stop delay: everything up to that time runs, nothing after. `finished`
records where the stop came from (`at`, `by`), where the simulation ended
(`end`), and any stops returned in between (`later`).

*L* is the model's **free delay**, which Orrery works out when it loads
the model. It depends on the model, never on the configuration, and
`sim.stop_delay()` reports it. Here nothing depends on the counter, so it
is 0; in the [GPGPU-Sim model](../examples/gpgpu-sim) it is 425 ps. The
stop delay cannot be set. For a model whose free delay Orrery cannot work
out, it is 0, and the load report says so (`reports: orr::Reports::All`).

With a stop delay, the end of a run and the end of the simulation may be
different calls:

- `run_until` returns `Finished` once it has run to the end, *T* + *L*;
  an earlier `run_until` returns `Reached` as usual. From then on, every
  `run_until` to a physical time in range, earlier or later, and every
  `step` return the same `Finished`; a request that does not hold, such
  as one to `Time::DELTA`, still returns `Request`
  ([Errors](./errors#while-it-runs)).
- `step` never runs past the timepoint it runs. With *L* = 0, the `step`
  that runs *T* returns `Finished`. With *L* > 0, `step` runs the
  timepoints up to the end one by one, then returns `Idle`, and
  `next_timepoint()` is `None`; a `run_until` past the end then returns
  `Finished`.

`Stop::Error(e)` and `Err(e)` from a `SimError` end the run with an error
instead; `?` converts. An error before the end of a stop is reported, and
the stops returned before it stay in `finished()`; nothing that would only
have happened after the end is reported ([Errors](./errors#while-it-runs)).

## Event log {#event-log}

A simulation can log every event the model sends, in order of the time it
arrives, and hand the log to a **sink** as it runs. The sink is given as
the simulation loads, through `Sim::with`. `orr::EventLogFile` is a sink
that writes the log to a file:

```rust
let log = orr::EventLogFile::create("events.log")?;
let mut sim = orr::Sim::with(plan, orr::SimConfig::default(), Some(Box::new(log)), |_, _| Ok(()))?;
sim.run_until(orr::Time::ns(100))?;
sim.close_event_log()?;
```

- Each run gives the sink the events up to where it stopped, and flushes
  it. `close_event_log` gives it the rest, events sent but due later, and
  ends the log; a run after it fails. Dropping the simulation closes the
  log too, but loses any error.
- An error from the sink fails the run, and every later one.
- Events at one timepoint come in a fixed order, so a model logs the same
  file on any number of threads, and two logs can be compared byte for
  byte.

`orr::EventLogReader` reads a log file back, event by event: the time, the
location and the payload's bytes.

```rust
let mut log = orr::EventLogReader::open("events.log")?;
while let Some(event) = log.next() {
    let (at, loc, payload) = event?;
    println!("{at} {} {payload:02x?}", log.path(loc));
}
```

```text
0ps+1 top.workers[0].producer.done [01, 00, 00, 00, 00, 00, 00, 00]
0ps+1 top.workers[1].producer.done [01, 00, 00, 00, 00, 00, 00, 00]
10ns+1 top.workers[0].producer.done [02, 00, 00, 00, 00, 00, 00, 00]
```

A log that was not closed, or was cut short, gives the events before the
cut, then an error of kind `UnexpectedEof`.

A sink of your own implements `orr::EventSink`: `event` takes each event,
`flush` and `end` are called as above, and `begin` once as the
simulation loads. It must be `Send`. `orr::EventLogFile::new(writer)` writes the
file format to any writer, a compressing one say.

A simulation [restored from a snapshot](./snapshots#restoring) takes its
sink the same way, and logs the events from the snapshot on, as the
original run logged them, but for events sent before the snapshot to an
event source that triggers no task:

```rust
let plan = orr::SimPlan::<Top>::new(params, Some(latest))?;
let log = orr::EventLogFile::create("restored.log")?;
let mut sim = orr::Sim::with(plan, config, Some(Box::new(log)), |_, _| Ok(()))?;
```

### The logging thread {#log-thread}

On one thread, the sink is called on the thread that calls `run_until`.
On several, the simulation runs one more thread, which calls the sink
([Threads](#threads)): 8 threads and a sink make 9, besides the thread
that calls `run_until`. Give the logging thread a core of its own too. On
fewer cores the simulation still ends and its results are the same, but
it may run slower.

### orr-events

The `orr-events` command prints a log file as text, one line per event,
its payload in hex. Run it from the Orrery checkout:

```sh
cargo run --release -p orrery-lang --bin orr-events -- --paths events.log
```

```text
0ps+1 top.workers[0].producer.done 0100000000000000
0ps+1 top.workers[1].producer.done 0100000000000000
10ns+1 top.workers[0].producer.done 0200000000000000
```

- Without `--paths`, a location is printed by a shorter name:
  `%Producer.done[0]`.
- `--windows <span>` (`50ns`, `1us`, `1500fs`) prints one line per window of time
  that has events, `<start> <events> <hash>`, to compare logs too large
  to keep: where two logs' lines differ, so do their events. With
  `--by column` too, each window's line is split by field
  (`%Producer.done`).
- `--fingerprint <log>` prints the fingerprint of the model the log is
  of; given a second log, it fails if the two are of different models.

It exits with 0, with 1 when it cannot read a file or the fingerprints
differ, and with 2 on arguments it does not take: no log, one too many,
or a `--windows` span that is not a time longer than 0. An option it
does not know is read as a file name. It reads the logs this version of
Orrery writes, and refuses those of earlier versions.

## After the run

- `sim.finished()` returns the stop, once any task has stopped.
- `sim.stop_report()` reports on the last `run_until` or `step` (`None`
  before the first), and `sim.load_report()` on building the simulation;
  `reports: orr::Reports::All` prints both.
- `sim.check_trace()`, with `checks` on, verifies that every two accesses
  to the same state, at least one of them a write, ran in the serial
  order (on several threads, that the run's synchronization ordered
  them), and returns the violations otherwise. Without `checks` it
  returns an error saying so.
- `sim.load_report().warnings` holds the warnings of the load, one a
  line, as stderr has them after `warning: `
  ([Errors](./errors#warnings)).
