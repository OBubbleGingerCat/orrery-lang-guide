# Getting started

This page builds the smallest useful model, the event pipeline: a clock,
and a number of workers, each a producer that counts on the clock and a
sink that sums what the producer sends it.

```text
Top
└── workers: [Worker; worker_count]
    ├── producer: Producer   on every clock edge: value += step; send value
    └── sink: Sink           on every value received: total += value
```

## Setting up

Orrery is not published on crates.io. Depend on it from a checkout you
have access to:

```toml
[dependencies]
orrery-lang = { path = "../orrery-lang/crates/orrery-lang" }

# Recommended: task bodies, the code the macros generate and Orrery itself
# are in different crates, and inlining across them makes models faster.
[profile.release]
lto = "fat"
codegen-units = 1
```

Orrery needs a recent stable Rust. The examples on this site name it
`orr`, with `use orrery_lang as orr;` at the top of each file.

## The producer

A module is a struct marked `#[orr::module]`. Its fields are parameters
(`#[orr::param]`), triggers (`#[orr::trigger]`), submodules
(`#[orr::submodule]`) and, without an attribute, state.

```rust
use orrery_lang as orr;

#[orr::module]
pub struct Producer {
    #[orr::param("increment per tick", default = 1)]
    step: u64,

    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::trigger]
    pub done: orr::EventSource<u64>,

    value: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,), emits = (done,))]
    fn produce(ctx: &mut orr::TaskCtx, value: &mut u64) -> Result<(), orr::SimError> {
        *value += *ctx.module().step;
        ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;
        Ok(())
    }
}
```

- `clock` is a **relay**: the clock comes from the module above.
- `done` is an **event source** with a `u64` payload.
- The task runs on every clock edge. `value: &mut u64` binds the state
  field of the same name for writing. `emit(.., orr::Time::DELTA)` sends
  the value one delta cycle later, at the same time.

## The sink

```rust
#[orr::module]
pub struct Sink {
    #[orr::trigger]
    pub done: orr::EventRelay<u64>,

    pub total: u64,
}

#[orr::module_impl]
impl Sink {
    #[orr::task(triggers = (done,))]
    fn consume(ctx: &mut orr::TaskCtx, total: &mut u64) -> Result<(), orr::SimError> {
        if let Some(value) = ctx.triggered_triggers().0 {
            *total += *value;
        }
        Ok(())
    }
}
```

`triggered_triggers()` gives one slot per trigger, `Some(payload)` for
those that fired.

## Connecting them

Connections are made in a module's `#[orr::init]`, with explicit calls;
nothing is connected by name.

```rust
#[orr::module]
pub struct Worker {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::submodule]
    producer: Producer,

    #[orr::submodule]
    sink: Sink,
}

#[orr::module_impl]
impl Worker {
    #[orr::init]
    fn init_worker(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.connect(module.clock, module.producer.clock)?;
        ctx.connect(module.producer.done, module.sink.done)?;
        Ok(())
    }
}

#[orr::module]
pub struct Top {
    #[orr::param("number of workers", default = 2)]
    worker_count: usize,

    #[orr::trigger]
    pub clock: orr::Clock,

    #[orr::submodule]
    workers: [Worker; orr::DynShape(worker_count)],
}

#[orr::module_impl]
impl Top {
    #[orr::init]
    fn init_top(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
        for worker in module.workers.iter() {
            ctx.connect(module.clock, worker.clock)?;
        }
        Ok(())
    }
}
```

- `orr::Clock` is an independent clock; `init_clock` gives it a period of
  10 ns and phase 0.
- `[Worker; orr::DynShape(worker_count)]` is an array whose length is the
  `worker_count` parameter.

## Running it

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let params = std::env::args_os().nth(1).map(std::path::PathBuf::from);
    let plan = orr::SimPlan::<Top>::new(params, None)?;
    let mut sim = plan.into_sim(orr::SimConfig::default())?;
    match sim.run_until(orr::Time::ns(100))? {
        orr::RunEnd::Reached(time) => println!("ran until {time}"),
        orr::RunEnd::Finished(finished) => println!("finished at {}", finished.at),
    }
    let totals: Vec<_> = (sim.top().workers.iter())
        .map(|worker| worker.sink.total)
        .collect();
    for (index, total) in totals.into_iter().enumerate() {
        println!("workers[{index}].sink.total = {}", sim.state(total)?);
    }
    Ok(())
}
```

- `SimPlan::new` reads the parameters (or uses the defaults when given
  `None`), builds the instances, runs the `init`s and checks the model.
- `into_sim` makes the simulation, here with the default configuration
  ([Running a simulation](./running#configuration)).
- `run_until` runs every timepoint up to 100 ns. The clock ticks at 0, 10,
  …, 100 ns, so each sink ends at 1 + 2 + … + 11 = 66.
- `sim.top()` gives handles to the model's state, read with `sim.state`.

## Parameters

Parameters come from a TOML file whose tables follow the instance tree.
Three workers, the second counting in tens:

```toml
[top]
worker_count = 3

[[top.workers]]
__select_index__ = 1
[top.workers.producer]
step = 10
```

```text
ran until 100ns
workers[0].sink.total = 66
workers[1].sink.total = 660
workers[2].sink.total = 66
```

## Next

- [Modules and parameters](./modules) and [State and tasks](./state-and-tasks)
  explain what you just wrote in full.
- [Time](./time) covers clocks, events and delays.
- [Running a simulation](./running) covers the configuration, stopping and
  reading results.
