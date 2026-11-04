# Time: clocks and events

## Time and timepoints

`orr::Time` is a physical time in integer femtoseconds, or `Time::DELTA`,
one delta cycle:

```rust
use orrery_lang as orr;

let period = orr::Time::ns(10); // also fs, ps, us, ms, s
let link = orr::Time::from_ps(425);
let next = orr::Time::DELTA;
```

It has no arithmetic or ordering of its own; compare and add through
`as_fs()`, which is `None` for `DELTA`. `from_ps` and `as_ps()` work in
whole picoseconds: `as_ps()` drops any part of a picosecond. A time is
displayed in the largest unit that keeps it exact: `10ns`, `1500ps`,
`1500fs`. In parameter files a time is a string: `"10ns"`, `"250 ps"`,
`"1500fs"`, `"delta"`.

A **timepoint** is a physical time plus a delta cycle, ordered by time and
then delta, and displayed as `10ns` or `10ns+1`. `Timepoint::at(ps)` and
`Timepoint::at_fs(fs)` make one at delta 0. Clocks fire at delta 0;
an event sent with `DELTA` arrives at the next delta of the same time; an
event sent with a physical delay arrives at delta 0 of that later time.
Timepoints run strictly in order: everything at `10ns`, then everything at
`10ns+1`, and so on.

## Clocks

An independent clock is an `orr::Clock` field, configured once in `init`
with a period and a phase (the time of its first edge, less than the
period), or with a frequency in hertz and a phase:

```rust
ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
ctx.init_clock_hz(module.core, 1_132_000_000, orr::Time::ps(0))?;
```

A clock given by its frequency has a period of 10¹⁵ / *hz* fs, which need
not be whole. Each edge is at its exact time rounded to the nearest
femtosecond, so the edges never drift: at 1132 MHz they are at 0,
883392 fs, 1766784 fs, 2650177 fs, and so on.

Modules that receive a clock declare an `orr::ClockRelay` and are
connected to it: `ctx.connect(module.clock, module.child.clock)?`. A task
on a clock sees the clock's `orr::ClockInfo` as its trigger payload:

- `period()` is the period as a fraction, `(num, den)` femtoseconds;
  `interval()` the least gap between two edges; `phase()` the phase.
- `cycle_of(now)` is the edge a timepoint falls on, counting the edge at
  the phase as 1; `edge(cycle)` is the timepoint of an edge;
  `is_edge(now)` and `edges_before(now)` say whether a timepoint is an
  edge and how many come before it.

An `init` reads a clock's `ClockInfo` with `ctx.clock_info(clock)`, once
the clock is initialized and, for a relay, connected
([`init`](./modules#init)); the host reads it with
`plan.clock_config(handle)`.

**Several clock domains** are just several clocks. Their periods and
phases are exact, so edges that should coincide do, forever. The
GPGPU-Sim model runs its core at 1132 MHz and its DRAM at 850 MHz as
periods of 425 and 566 picoseconds, which keeps the ratio between them
exact.

There is no limit on the number of clocks, or on how their periods
combine, but one: two clocks whose edges come within less than 1 fs of
each other without meeting, such as one at 3 GHz and one with a period of
333333 fs, fail the load (`LoadError::Init`, stage `clocks`). A clock's
period and phase are fixed once `init` has set them; a unit whose
frequency changes while it runs can trigger itself with events instead,
sent each time with the period it keeps in its state.

## Events

An event source has a payload type, which must be a plain type:

```rust
#[orr::trigger]
pub done: orr::EventSource<u64>,
```

A task that may send on it lists it in `emits`, and sends with a delay:

```rust
ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;   // next delta cycle
ctx.emit_triggers().0.emit(*value, orr::Time::ns(2))?;   // 2 ns later
```

A delay is `DELTA` or a positive physical time; a physical delay of zero
is an error. The receiving side declares an `orr::EventRelay<T>` and is
connected to the source, or to another relay; a relay chain adds no delay,
and a relay has exactly one source. A task triggered by the relay finds
the payload in its trigger slot.

A source may receive at most one event per timepoint: two events for the
same location and timepoint, from two tasks or from one, are a
**multi-drive** error. Send several values at once as one payload, or use
an array of sources.

## Declared delays

An event source can declare the delays its events may use:

```rust
#[orr::param("link delay", default = orr::Time::ns(2))]
link: orr::Time,

#[orr::trigger(min_delay = link)]
pub out: [orr::EventSource<Packet>; 4],

#[orr::trigger(max_delay = orr::Time::DELTA)]
pub hint: orr::EventSource<()>,
```

A bound is a constant `orr::Time` or the name of a `Time` parameter of the
module. Every emit is checked against it: an emit outside the bounds
fails the run.

Declare the real minimum latency of every link and port. Declared
minimums never change results. They may change the model's stop delay,
how long the simulation keeps running after a stop
([Stopping](./running#stopping)); `sim.stop_delay()` reads its value.

`max_delay = Time::DELTA` says the event only ever arrives a delta cycle
later, so it can never meet a clock edge: tasks it triggers never run in
the same timepoint as clocked ones. Conflicts between the two then cannot
happen, and are neither reported as possible nor checked while the
simulation runs.

## Delta cycles

Delta cycles order things within one physical time: a chain of events,
each sent with `DELTA`, runs as a sequence of timepoints at the same time.
More than `delta_limit` delta cycles at one time (30 by default) is an
error, which catches models that never settle. Set `SimConfig::delta_limit`
for models that need longer chains, up to 62.

A run reaches about 288 seconds of simulated time, whatever the limit. A
`run_until` past that fails with `RunError::Request` and runs nothing;
the simulation goes on with a time it reaches.

## Clock gates

A unit with nothing to do can switch its clock off. It declares a gate as
a state field and names it on the triggers the gate controls:

```rust
#[orr::module]
pub struct Sm {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::trigger(gate = gate)]
    pub request: orr::EventRelay<Request>,
    #[orr::trigger]
    pub wake: orr::EventRelay<()>,
    pub gate: orr::Gate,
    pub next_work: u64,
}
```

A gated trigger fires only while its gate is open. A clock edge at which
the gate is closed does not run the task; when every
gated clock at an edge is closed and nothing else happens there, the edge
is not a timepoint at all, and `sim.step()` passes over it. An event that
arrives at a gated trigger while the gate is closed is dropped and
counted. Triggers without `gate =` are not affected, so an ungated event
can run the task while its gate is closed (its gated slots are then
`None`) and open the gate.

The task takes the gate as `&mut orr::Gate`:

```rust
#[orr::task(triggers = (clock, request, wake))]
fn step(
    ctx: &mut orr::TaskCtx,
    gate: &mut orr::Gate,
    next_work: &mut u64,
) -> Result<(), orr::SimError> {
    if ctx.triggered_triggers().2.is_some() {
        gate.open(); // woken: open from the next timepoint
        return Ok(());
    }
    // ... a cycle of work, which sets `next_work` ...
    match *next_work {
        0 => {}                                                    // busy: stay open
        u64::MAX => gate.close(),                                  // until woken
        at => gate.close_until(orr::Timepoint::at(u128::from(at))), // a known deadline
    }
    Ok(())
}
```

- A write takes effect from the next timepoint. Within the timepoint the
  gate keeps the value the triggers saw, which `is_open()` reports.
- `close_until(t)` opens the gate at `t`: events that arrive at `t` or
  later pass, and the clock runs again from its first edge at or after
  `t`. `t` need not be an edge.
- A gate starts open, or as its reset says:
  `#[orr::reset(orr::Gate::closed())]`,
  `#[orr::reset(orr::Gate::closed_until(orr::Timepoint::at(4_000)))]`.
- `gate.opens()` is the timepoint the gate opens at as last written
  (`Timepoint::ZERO` when open, `Timepoint::MAX` when closed).
- `gate.gated_edges()` counts the edges at which the gate was closed, up
  to and including now: one count per clock the gated triggers name, in
  the order they are declared. It is what power or utilisation statistics
  need.

A gate, its triggers and the task that writes it belong together:

- the gate is a scalar state field of the module itself, of type
  `orr::Gate`;
- a gate controls the triggers of one task, and a task's gated triggers
  share one gate;
- only that task writes the gate; other tasks may read it.

The macros and `SimPlan::new` check these. The gate a task takes stays
in its call: `std::mem::take` and `std::mem::replace` on it do not
compile.

Between runs, `sim.gate_mut(handle)` gives the host the gate:
`open()`, `close()`, `close_until(t)`, `is_open()` and `opens()`. What it
writes takes effect when the next run starts. `sim.state` and
`sim.state_mut` do not take a gate. `sim.gated_edges(handle)` reads the
counts. `sim.dropped_events()` lists the dropped events per
instance, task and trigger, and dropping the simulation prints a warning
if there were any; with `SimConfig::fail_on_drop` the first one is an
error instead, and no task runs at its timepoint. Snapshots carry the gates, the dropped events and the
gated edges, so a restored run continues the counts.

## Wakes {#wakes}

A task can open another module's gate without sending an event, by a
**wake**. The sending module declares an `orr::Wake` field and lists it in
its task's `wakes`; the receiving module declares an `orr::WakeRelay`
that names the gate it opens; an `init` connects the two with
`ctx.connect`:

```rust
#[orr::module]
pub struct Unit {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::wake(gate = gate)]
    pub wake: orr::WakeRelay,
    gate: orr::Gate,
    pub runs: u64,
}

#[orr::module_impl]
impl Unit {
    #[orr::task(triggers = (clock,))]
    fn step(_ctx: &mut orr::TaskCtx, gate: &mut orr::Gate, runs: &mut u64) -> Result<(), orr::SimError> {
        *runs += 1;
        gate.close(); // nothing to do until woken
        Ok(())
    }
}

#[orr::module]
pub struct Feeder {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    pub units: [orr::Wake; 2],
}

#[orr::module_impl]
impl Feeder {
    #[orr::task(triggers = (clock,), wakes = (units,))]
    fn feed(ctx: &mut orr::TaskCtx) -> Result<(), orr::SimError> {
        let clock = ctx.triggered_triggers().0.expect("runs on its clock");
        let wakes = &ctx.wakes().0;
        if clock.cycle_of(ctx.now()) % 4 == 0 {
            wakes.wake(1); // from the next timepoint
            wakes.wake_after(0, clock.interval())?; // from the next edge on
        }
        Ok(())
    }
}
```

```rust
ctx.connect(module.feeder.units[i], unit.wake)?;
```

- `wake()` opens the gate from the next timepoint, and wins over a
  `close()` written at the same timepoint, by the gate's own task too. A
  unit that runs only on its clock, like the one above, runs again at
  its next edge.
- `wake_after(delay)`, with `delay` `DELTA` or a positive physical time,
  opens the gate from now + `delay`. It takes effect just before that
  timepoint: it wins over every `close()` written before, and a `close()`
  written at that timepoint or later closes the gate again. A unit that
  runs only on its clock runs at its first edge at or after that time;
  an event that arrives at a gated trigger at that time or later passes
  too.
- Several wakes of one gate at one timepoint are one.
- `ctx.wakes()` has one entry per `wakes = (..)` entry: one wake, or, for
  an array, `wake(i)`, `wake_after(i, delay)` and `len()`.
- A `Wake` or a `WakeRelay` marked `#[orr::wake(forward)]` passes wakes
  out of a submodule or in to one, as relays pass triggers.

With the `Unit` above on a 1 ns clock, and a feeder on the same clock
that wakes two of them:

```rust
let wakes = &ctx.wakes().0;
match ctx.now().ps() {
    0 => {
        wakes.wake_after(0, orr::Time::ns(2))?; // unit 0 runs at 0 and 2 ns
        wakes.wake_after(1, orr::Time::ns(2))?; // unit 1 runs at 0, 2 and 3 ns
    }
    2_000 => wakes.wake(1), // wins over unit 1's close at 2 ns
    _ => {}
}
```

Both gates open just before 2 ns, and both units close them again when
they run there. Unit 0's stays closed; unit 1's opens again from the next
timepoint, by the `wake()` at 2 ns.

For now, a model whose tasks wake runs on one worker
([Workers](./running#workers)).
