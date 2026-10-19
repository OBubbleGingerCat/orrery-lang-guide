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
- `into_sim(config)` builds the simulation. A model with
  [external ports](./external-ports) uses `into_sim_with(config, |top, ports| ..)`
  to bind them.

## Configuration

`orr::SimConfig`:

| Field | Default | Meaning |
|---|---|---|
| `checks` | `false` | record the order in which tasks access state, for `sim.check_trace()`; slow, for tests |
| `delta_limit` | 30 | the most delta cycles at one physical time, at most 65,534; a larger one shortens how far in time a run reaches ([Delta cycles](./time#delta-cycles)) |
| `stop_delay` | `None` | how long after a stop the simulation runs; `None` is the model's free delay ([Stopping](#stopping)) |
| `event_capacity` | 2²⁰ | the most events one task of one instance may have in flight on one event source: sent, and not yet arrived |
| `fail_on_drop` | `false` | an event that reaches a trigger while its gate is closed is an error, rather than dropped and counted ([Clock gates](./time#clock-gates)) |
| `snapshots` | `None` | when and where to take snapshots |

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

## After the run

- `sim.finished()` returns the stop, once any task has stopped.
- `sim.check_trace()`, with `checks` on, verifies that every two accesses
  to the same state, at least one of them a write, ran in the serial
  order, and returns the violations otherwise.
