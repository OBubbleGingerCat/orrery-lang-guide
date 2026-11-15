# Migrating from the previous version

This guide is for Orrery v2. A model written for the previous version
keeps its modules, tasks, parameters and access modes; what changes is
how the simulation is built and configured, the unit of time, and a few
names. This page lists them. Where a name below has no replacement, there
is none: drop it.

## Building and running

| Previous version | v2 |
|---|---|
| `plan.into_sim(config)` | `orr::Sim::new(plan, config)` |
| `plan.into_sim_with(config, log, bind)` | `orr::Sim::with(plan, config, event_log, bind)` |
| `sim.log_events_to(sink)` | gone: the event log is given as the simulation is built, as `Sim::with`'s `event_log` ([Event log](./running#event-log)); `close_event_log` stays |
| `sim.state_mut(gate)` | `sim.gate_mut(gate)` |
| `plan.analysis().warnings()` | `sim.load_report().warnings` ([Warnings](./errors#warnings)) |
| `SimPlan::new` reports write conflicts and ordering cycles | `Sim::new` (and `Sim::with`) reports them ([Errors](./errors#when-the-simulation-loads)) |
| `SnapshotEvery::Cycles { clock, cycles }` | `SnapshotEvery::cycles(clock, cycles)`, with `cycles` a `NonZeroU64` |
| snapshot files named by picoseconds | named by femtoseconds, `<20 digits>.fs.orrs`; `orr::snapshot_path` gives the name |

## Configuration

| Previous version | v2 |
|---|---|
| `workers: N`, the threads that run tasks | `workers_per_group`, which **counts differently**: it is every thread of the run, one of them handing out the work. `workers: 4` is `workers_per_group: 5` ([Threads](./running#threads)) |
| `overlap` | gone, not needed |
| `stop_delay` | gone: Orrery works the stop delay out from the model; `sim.stop_delay()` reads it. A model whose free delay cannot be worked out no longer fails to build: its stop delay is 0, and the load report says so |
| `skip_closed` | gone: edges where every gated clock is closed are passed over on their own |
| `lead_limit`, `wait`, `placement`, `pin`, `topology`, `steal`, and the modules `orr::exec` and `orr::plan` | gone: Orrery chooses. `cpus` limits the CPUs a run on several threads may take |
| `job_log`, `account`, `sim.account()` | gone. The nearest are `reports: orr::Reports::All`, `sim.load_report()` and `sim.stop_report()` |
| `delta_limit` up to 65,534 | up to 62, 30 by default |

The fields of `SimConfig` a model's user sets are in
[Configuration](./running#configuration).

## Time

| Previous version | v2 |
|---|---|
| `Time` in whole picoseconds | in whole femtoseconds. `Time::ps`, `from_ps` and `as_ps` still take and give picoseconds, exactly for whole ones; `Time::fs` and `as_fs` are new |
| `Timepoint::at(ps)` | the same, in picoseconds; `Timepoint::at_fs` takes femtoseconds |
| how far a run can go depends on `delta_limit` | it does not: simulated time ends at `u64::MAX` femtoseconds, some 5.1 hours ([Time](./time)) |

## Gates

| Previous version | v2 |
|---|---|
| `orr::Gate::default()`, `std::mem::take` on a `&mut orr::Gate` | do not compile: no gate is made outside the simulation ([Clock gates](./time#clock-gates)) |
| the host opens a gate with `sim.state_mut(gate)` | `sim.gate_mut(gate)?.open()` |
