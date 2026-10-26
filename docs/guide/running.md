# Running a simulation

## From plan to simulation

```rust
use orrery_lang as orr;

let plan = orr::SimPlan::<Top>::new(Some("params.toml".into()), None)?;
let mut sim = plan.into_sim(orr::SimConfig::default())?;
```

- `SimPlan::new(params, snapshot)` elaborates and checks the model. Both
  arguments are optional paths: without a parameter file every parameter
  keeps its default; with a snapshot the simulation starts from it
  ([Snapshots](./snapshots)).
- `into_sim(config)` builds the simulation.
  `into_sim_with(config, event_log, |top, ports| ..)` takes two more
  arguments: a sink for the [event log](#event-log), or `None`, and a
  closure that binds the [external ports](./external-ports) of a model
  that has them.

## Configuration

`orr::SimConfig`:

| Field | Default | Meaning |
|---|---|---|
| `workers` | 1 | the threads that run the tasks; the placement may use fewer; with more than one and an [event log](#event-log), one more runs the log ([Threads](#threads)) |
| `overlap` | `true` | start later timepoints while earlier ones still run, where the model allows it; `false` runs one timepoint at a time, for tests |
| `checks` | `false` | record the order in which tasks access state, for `sim.check_trace()`; slow, for tests |
| `delta_limit` | 30 | the most delta cycles at one physical time, at most 65,534; a larger one shortens how far in time a run reaches ([Delta cycles](./time#delta-cycles)) |
| `stop_delay` | `None` | how long after a stop the simulation runs; `None` is the model's free delay ([Stopping](#stopping)) |
| `event_capacity` | 2²⁰ | the most events one task of one instance may have in flight on one event source: sent, and not yet arrived |
| `fail_on_drop` | `false` | an event that reaches a trigger while its gate is closed is an error, rather than dropped and counted ([Clock gates](./time#clock-gates)) |
| `snapshots` | `None` | when and where to take [snapshots](./snapshots) |
| `skip_closed` | `Some(64)` | pass over a stretch of at least this many clock edges at which every gated clock is closed and nothing else happens in one go; `None` passes over them one at a time |
| `lead_limit` | `Some(256)` | how far parts of the model may run ahead of the parts they send events to, which bounds the events in flight and the memory they take; at least 1, `None` for no bound |
| `wait` | `orr::exec::Adaptive` | how quickly workers answer against the CPU they hold while they have nothing to run: `Adaptive` answers fast and frees its core when idle for long; `AlwaysSpin` suits a machine the simulation has to itself, `AlwaysPark` more workers than free cores |
| `placement` | `orr::plan::Measured` | how the model is laid out on the workers; `Measured` measures the run to decide |
| `pin` | `orr::exec::NoPin` | which CPUs the workers run on: `NoPin` leaves it to the OS; `CacheDomain` keeps them on as many physical cores as there are workers, nearest the calling thread's |
| `topology` | `None` | the CPUs `pin` chooses from; `None` reads this machine's |
| `steal` | `orr::exec::DomainSteal` | which workers share work: `DomainSteal` those that share a cache, `FlatSteal` all of them, `NoSteal` none |
| `job_log` | `None` | a directory where each worker writes a record of its work, for performance analysis |
| `account` | `false` | keep an account of where the workers' CPU time goes, read with `sim.account()` |

None of these changes results, except `stop_delay`, `delta_limit`,
`event_capacity` and `fail_on_drop`, which change them deterministically: a given value gives
the same results on any machine and any number of workers.

```rust
let config = orr::SimConfig {
    checks: true,
    delta_limit: 100,
    ..orr::SimConfig::default()
};
```

The policies (`wait`, `placement`, `pin`, `steal`) are shared values:

```rust
use std::sync::Arc;

let config = orr::SimConfig {
    workers: 4,
    wait: Arc::new(orr::exec::AlwaysSpin),
    pin: Arc::new(orr::exec::CacheDomain),
    ..orr::SimConfig::default()
};
```

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
2 µs at once.

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
reaches stores, external sources and [clock gates](./time#clock-gates).
What the host writes is seen from the first timepoint of the next run.

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

The default *L* is the model's **free delay**, which Orrery works out from
the model to keep stopping cheap. It depends on the model, never on the
configuration, and `sim.stop_delay()` reports it. Here nothing depends on
the counter, so it is 0. Any other physical delay can be set as `stop_delay`,
shorter or longer: a shorter one, such as `Some(orr::Time::ps(0))`, gives
exact results too, but may make the simulation slower. For a few models
Orrery cannot work out the free delay; building the simulation then fails
and asks for a `stop_delay`.

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
arrives, and hand the log to a **sink** as it runs. `orr::EventLogFile` is
a sink that writes it to a file:

```rust
let mut sim = plan.into_sim(orr::SimConfig::default())?;
sim.log_events_to(orr::EventLogFile::create("events.log")?)?;
sim.run_until(orr::Time::ns(100))?;
sim.close_event_log()?;
```

- Install the sink before the first run, once.
- Each run gives the sink the events up to where it stopped, and flushes
  it. `close_event_log` gives it the rest, events sent but due later, and
  ends the log; a run after it fails. Dropping the simulation closes the
  log too, but loses any error.
- An error from the sink fails the run, and every later one.
- Events at one timepoint come in a fixed order, so a model logs the same
  file on any number of workers, and two logs can be compared byte for
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
`flush` and `end` are called as above, and `begin` once as it is
installed. It must be `Send`. `orr::EventLogFile::new(writer)` writes the
file format to any writer, a compressing one say.

A simulation [restored from a snapshot](./snapshots#restoring) takes its
sink as it is built, through `into_sim_with`, and logs the events from the
snapshot on, as the original run logged them; `log_events_to` fails on
it:

```rust
let plan = orr::SimPlan::<Top>::new(params, Some(latest))?;
let log = orr::EventLogFile::create("restored.log")?;
let mut sim = plan.into_sim_with(config, Some(Box::new(log)), |_, _| Ok(()))?;
```

### Threads {#threads}

With one worker, the sink is called on the thread that calls `run_until`.
With more, the simulation runs one more thread, which calls the sink: 8
workers and a sink make 9 threads. Give that thread a core of its own, as
each worker has. On fewer cores, the logging thread sharing one with a
worker or other work running on them, the simulation still ends and its
results are the same, but it may run slower.

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
- `--windows <span>` (`50ns`, `1us`) prints one line per window of time
  that has events, `<start> <events> <hash>`, to compare logs too large
  to keep: where two logs' lines differ, so do their events. With
  `--by column` too, each window's line is split by field
  (`%Producer.done`).
- `--fingerprint <log>` prints the fingerprint of the model the log is
  of; given a second log, it fails if the two are of different models.

It exits with 0, with 1 when it cannot read a file or the fingerprints
differ, and with 2 on arguments it does not take.

## After the run

- `sim.finished()` returns the stop, once any task has stopped.
- `sim.check_trace()`, with `checks` on, verifies that every two accesses
  to the same state, at least one of them a write, ran in the serial
  order, and returns the violations otherwise.
