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

`orr::SimConfig`:

| Field | Default | Meaning |
|---|---|---|
| `workers_per_group` | 1 | the workers that run the tasks ([Workers](#workers)); 0 fails the load |
| `groups` | 1 | the groups of workers; for now 1 is the only number that loads |
| `checks` | `false` | record the order in which tasks access state, for `sim.check_trace()`; slow, for tests; one worker only |
| `delta_limit` | 30 | the most delta cycles at one physical time, at most 62 ([Delta cycles](./time#delta-cycles)) |
| `event_capacity` | 2²⁰ | the most events in flight from one event source of one instance: sent, and not yet arrived |
| `fail_on_drop` | `false` | an event that reaches a trigger while its gate is closed is an error, rather than dropped and counted ([Clock gates](./time#clock-gates)) |
| `snapshots` | `None` | when and where to take [snapshots](./snapshots); one worker only |
| `reports` | `Warnings` | what the simulation prints to stderr: `Warnings`, the warnings of the load and, as it is dropped, the count of events dropped at closed gates; `Off`, nothing; `All`, also a report of the load and of each run |
| `cpus` | `None` | on several workers, the CPUs the simulation may take its cores from; `None` is any the process may run on |
| `idle_spin` | 1 ms | on several workers, how long the simulation's threads keep their cores busy between runs before they sleep |
| `slots` | 16 | on several workers, a tuning setting: a power of two from 2 to 1024 |

None of these changes results, except `delta_limit`, `event_capacity`
and `fail_on_drop`, which change them deterministically: a given value
gives the same results on any machine and any number of workers.

Fields will be added. A configuration written with
`..orr::SimConfig::default()` keeps compiling:

```rust
let config = orr::SimConfig {
    checks: true,
    delta_limit: 50,
    ..orr::SimConfig::default()
};
```

## Workers {#workers}

With one worker, the default, the simulation runs its tasks on the thread
that calls `run_until` or `step`. With 2 or more, it runs them on that
many threads of its own, and takes one physical core more than it has
workers, all in one L3 cache domain:

```rust
let config = orr::SimConfig {
    workers_per_group: 4,
    ..orr::SimConfig::default()
};
```

- *W* workers take a domain with at least *W* + 1 physical cores, all of
  them among the CPUs that `cpus` allows. A machine whose domains are
  smaller fails the load (`LoadError::Config`, field
  `workers_per_group`), and so does a `cpus` that holds no domain whole
  with enough cores (field `cpus`). On a machine with 8 cores to a
  domain, that makes 7 workers at most.
- The simulation holds its domain until it is dropped. Another
  simulation in the same process, also on several workers, takes another
  domain, and fails to load if none is left (field `cpus`).

On several workers a model gives what it gives on one: the same state,
end and errors. For now, though, a run on several workers takes models of
clocks, [gates](./time#clock-gates), state, components such as `Fifo`,
and [external ports](./external-ports). These fail the load on 2 or more
workers (`LoadError::Config`, field `parallel`), and run on one:

- a task that emits events, or wakes a task ([Wakes](./time#wakes));
- `checks`;
- snapshots, taken or restored from;
- an event log.

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
([Time](./time#the-time-type)).

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
`sim.stop_delay()` reports it. Here it is 0; in the
[GPGPU-Sim model](../examples/gpgpu-sim) it is 425 ps. It is 0 too for a
model whose free delay Orrery cannot work out. It cannot be set.

With a stop delay, the end of a run and the end of the simulation may be
different calls:

- `run_until` returns `Finished` once it has run to the end, *T* + *L*;
  an earlier `run_until` returns `Reached` as usual. From then on, every
  `run_until`, whatever its time, and every `step` return the same
  `Finished`.
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

- For now, a simulation with an event log runs on one worker.
- Each run gives the sink the events up to where it stopped, and flushes
  it. `close_event_log` gives it the rest, events sent but due later, and
  ends the log; a run after it fails. Dropping the simulation closes the
  log too, but loses any error.
- An error from the sink fails the run, and every later one.
- Events at one timepoint come in a fixed order, so a model logs the same
  file in every run, and two logs can be compared byte for byte.

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

The sink is called on the thread that calls `run_until` or `step`.

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
- `sim.check_trace()`, with `checks` on, verifies that every two accesses
  to the same state, at least one of them a write, ran in the serial
  order, and returns the violations otherwise. Without `checks` it
  returns an error saying so.
- `sim.load_report().warnings` holds the warnings of the load, one a
  line, as stderr has them after `warning: `
  ([Errors](./errors#warnings)).
