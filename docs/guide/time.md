# Time: clocks and events

## Time and timepoints

`orr::Time` is a physical time in integer picoseconds, or `Time::DELTA`,
one delta cycle:

```rust
use orrery_lang as orr;

orr::Time::ns(10)        // also ps, us, ms, s
orr::Time::from_ps(425)
orr::Time::DELTA
```

It has no arithmetic or ordering of its own; compare and add through
`as_ps()`, which is `None` for `DELTA`. In parameter files a time is a
string: `"10ns"`, `"250 ps"`, `"delta"`.

A **timepoint** is a physical time plus a delta cycle, ordered by time and
then delta, and displayed as `10ns` or `10ns+1`. Clocks fire at delta 0;
an event sent with `DELTA` arrives at the next delta of the same time; an
event sent with a physical delay arrives at delta 0 of that later time.
Timepoints run strictly in order: everything at `10ns`, then everything at
`10ns+1`, and so on.

## Clocks

An independent clock is an `orr::Clock` field, configured once in `init`
with a period and a phase (the time of its first edge, less than the
period):

```rust
ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
```

Modules that receive a clock declare an `orr::ClockRelay` and are
connected to it: `ctx.connect(module.clock, module.child.clock)?`. A task
on a clock sees the clock's `ClockInfo` (period and phase) as its trigger
payload.

**Several clock domains** are just several clocks. Their periods and
phases are exact integers, so edges that should coincide do, forever. The
GPGPU-Sim model runs its core at 1132 MHz and its DRAM at 850 MHz as
periods of 425 and 566 picoseconds, which keeps the ratio between them
exact.

There is no limit on the number of clocks, or on how their periods
combine. A clock's period and phase are fixed once `init` has set them; a
unit whose frequency changes while it runs can trigger itself with events
instead, sent each time with the period it keeps in its state.

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
module. Every emit is checked against it.

**`min_delay` is what lets event-driven parts run in parallel.** Declare
the real minimum latency of every link and port: declared minimums let
Orrery run more of the model at once. They never change results.

`max_delay = Time::DELTA` says the event only ever arrives a delta cycle
later, so it can never meet a clock edge: tasks it triggers never run in
the same timepoint as clocked ones. Conflicts between the two then cannot
happen, and are neither reported as possible nor checked while the
simulation runs.

A minimum delay on a source used by a task that may stop the simulation
also sets how long after a stop the simulation keeps running
([Stopping](./running#stopping)).

## Delta cycles

Delta cycles order things within one physical time: a chain of events,
each sent with `DELTA`, runs as a sequence of timepoints at the same time.
More than `delta_limit` delta cycles at one time (30 by default) is an
error, which catches models that never settle. Set `SimConfig::delta_limit`
for models that need longer chains, up to 65,534.

The limit also sets how far in simulated time a run reaches: about 160
hours at the default, 5 hours at 1000, 281 seconds at 65,534. A run that
would go later fails with an error that names the limit.

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
the gate is closed does not run the task, and costs nothing; when every
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
        gate.open(); // woken: run again from the next edge
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

The macros and the plan check these.

Between runs, `sim.state_mut(handle)` gives the host the gate; what it
writes takes effect when the next run starts. `sim.gated_edges(handle)`
reads the counts. `sim.dropped_events()` lists the dropped events per
instance, task and trigger, and dropping the simulation prints a warning
if there were any; with `SimConfig::fail_on_drop` the first one is an
error instead, and no task runs at its timepoint. Snapshots carry the gates, the dropped events and the
gated edges, so a restored run continues the counts.
