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

Declare the real minimum latency of every link and port.
The minimums may change the model's stop delay,
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

When a unit knows when it will next have work (a fixed latency, a timer,
a timing parameter it waits out), close its gate until then rather than
closing it and waiting to be woken: no other unit has to send the wake, or
work out how many edges ahead to send it.

- A write takes effect from the next timepoint. Within the timepoint the
  gate keeps the value the triggers saw, which `is_open()` reports (but
  to a `&'latest orr::Gate` reader, below).
- `close_until(t)` opens the gate at `t`: events that arrive at `t` or
  later pass, and the clock runs again from its first edge at or after
  `t`. `t` need not be an edge. `orr::Timepoint::at(ps)` is delta 0 of a
  time in picoseconds (`Timepoint::at_fs` in femtoseconds).
- A gate starts open, or as its reset says:
  `#[orr::reset(orr::Gate::closed())]`,
  `#[orr::reset(orr::Gate::closed_until(orr::Timepoint::at(4_000)))]`.
- `gate.opens()` is the timepoint the gate opens at as last written
  (`Timepoint::ZERO` when open, `Timepoint::MAX` when closed).
- `gate.gated_edges()` counts the edges at which the gate was closed, up
  to and including now: one count per clock the gated triggers name, in
  the order they are declared. Of a gate that controls several tasks, the
  tasks come in the order of their names (the order the macros give a
  module's tasks, which is the program's order of tasks), each task's
  counts in the order of its triggers; a clock named at several levels
  has a count for each, each at its level. It is what power or
  utilisation statistics need.

A gate belongs to its module instance, not to a trigger or to a task:

- the gate is a scalar state field of the module itself, of type
  `orr::Gate`;
- a task's gated triggers share one gate; a gate may control the
  triggers of several tasks of the instance: their `triggers` name one
  field with `gate = g`, or several fields each say `gate = g`;
- one task at most writes the gate (takes `&mut orr::Gate`): one of the
  tasks it controls, or another task of the module;
- other tasks may read it, but for a gate a task wakes ([Wakes](#wakes)):
  only the tasks it controls and its writer read that one.

The macros and `SimPlan::new` check these. While the gate is closed,
none of the tasks it controls runs on a gated trigger. Give tasks gates
of their own when they are to sleep and wake apart: several `ClockRelay`
fields, each with its own `gate = ..`, all connected to the same clock.
Tasks that have nothing to do with each other and should run in parallel
want gates of their own too: the tasks a gate controls and its writer
are in one executor unit, and run one after another.

When a gate controls several tasks:

- The levels of one clock are different timepoints
  ([Delta cycles](#delta-cycles)). A writer that closes the gate at level
  0 of edge e stops the levels it controls from level 1 of that edge on;
  one that closes it at the last level it controls stops them all from
  the next edge. To decide once per edge, run the writer at the last
  level.
- `close_until(t)`: each gated clock runs again from its first point at
  or after `t`, each at its level; events pass from `t` on.
- Reading the gate in a task's body: `&orr::Gate`, and the writer's
  `&mut orr::Gate`, take the value as the timepoint begins, the one the
  gated triggers saw; `&'latest orr::Gate` takes it after the writer's
  write at the timepoint, which may differ: if the writer closed the
  gate, the triggers fired and yet the `&'latest` reader's `is_open()` is
  `false`. A task the gate controls that asks whether the gate is open
  at the timepoint takes `&orr::Gate`. `gated_edges()` is the same to the
  writer and to every reader.
- A writer that closes the gate at the last level of edge e, and writes
  it only there, makes every level count the same edges. An ungated
  event that makes the writer open the gate at level 0 of edge e opens it
  from level 1: the count of level 0 has that edge more.
- `sim.dropped_events()` lists by instance, task and trigger as before:
  an event that two gated tasks subscribe to, arriving while the gate is
  closed, is dropped once at each of their triggers.
- A writer that runs on a clock the gate does not control runs every
  cycle: a closed gate does not spare the unit its cycles. Make the
  writer one of the tasks the gate controls, or one an event triggers.

The `&mut orr::Gate` a task takes is the gate itself, for the call. No
gate is made outside the simulation, so a task cannot move the one it
holds out of the call: `std::mem::take(gate)` and
`std::mem::replace(gate, ..)` do not compile, and neither does
`orr::Gate::default()`. Two gates one call holds may be swapped with
`std::mem::swap`; a write through either reaches the gate it then holds.

Between runs, `sim.gate_mut(handle)` gives the host the gate:
`open()`, `close()`, `close_until(t)`, `is_open()` and `opens()`. What it
writes takes effect when the next run starts. `sim.state` and
`sim.state_mut` do not take a gate. `sim.gated_edges(handle)`
reads the counts. `sim.dropped_events()` lists the dropped events per
instance, task and trigger, and dropping the simulation prints a warning
if there were any; with `SimConfig::fail_on_drop` the first one is an
error instead, and no task runs at its timepoint. Snapshots carry the gates, the dropped events and the
gated edges, so a restored run continues the counts.

## Wakes {#wakes}

A unit that closed its gate until there is work is opened by the unit that
brings the work, through a **wake**. A wake opens a gate and does nothing
else: it runs no task and carries no value.

- The sending module has a field of type `orr::Wake` (or an array,
  `[orr::Wake; N]`), and the task that wakes names it in
  `wakes = (..)`. `ctx.wakes()` has one entry per `wakes = (..)` entry:
  for one wake, `ctx.wakes().0.wake()` sends it; for an array,
  `wake(i)` wakes element `i`, with `wake_after(i, delay)` and `len()`.
- The receiving module has a field of type `orr::WakeRelay`, marked with
  the gate it opens: `#[orr::wake(gate = gate)]`. A relay that names a
  gate is a single field, not an array.
- The parent connects the two in its init:
  `ctx.connect(module.producer.consumer, module.consumer.wake)?`.

A consumer that closes its gate whenever its FIFO is empty, and the
producer that wakes it whenever it pushes:

```rust
#[orr::module]
pub struct Consumer {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::wake(gate = gate)]
    pub wake: orr::WakeRelay,
    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,
    pub gate: orr::Gate,
    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    #[orr::task(triggers = (clock,))]
    fn take(
        _ctx: &mut orr::TaskCtx,
        inbox: &mut orr::FifoRx<u64>,
        gate: &mut orr::Gate,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        if inbox.is_empty() {
            gate.close(); // until the producer wakes it
        }
        Ok(())
    }
}

#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub out: orr::FifoTx<u64>,
    pub consumer: orr::Wake,
    next: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,), wakes = (consumer,))]
    fn send(
        ctx: &mut orr::TaskCtx,
        out: &'latest mut orr::FifoTx<u64>,
        next: &mut u64,
    ) -> Result<(), orr::SimError> {
        *next += 1;
        if *next % 4 == 0 && out.push(*next).is_ok() {
            ctx.wakes().0.wake();
        }
        Ok(())
    }
}
```

**When a wake takes effect.**

- `wake()` opens the gate from the next timepoint: a closed unit runs
  again at its gated clocks' first edge after now. Several wakes of one
  gate at one timepoint are one.
- A wake wins over a `close()` that the gate's writer writes at the same
  timepoint. Above, the consumer may find its FIFO empty and close in the
  very cycle the producer pushes and wakes it; the gate stays open.
- Of a gate that controls several tasks, a wake wakes them all: each
  gated trigger runs again from its first point after the gate opens,
  each at its level.
- `wake_after(delay)` opens the gate from its first edge at or after
  now + `delay` (`orr::Time::DELTA` or a positive time). It takes effect
  just before then: it wins over every `close()` written before, and a
  `close()` from then on wins over it. An event that arrives at a gated
  trigger at that time or later passes too. For a value that is due *n* edges
  on, `orr::wake_delay(now, clock, n)?` is the delay that opens the gate
  at the edge where the value is seen ([Delays](./components#delays)).

A unit that closes its gate each time it runs, and a feeder on the same
1 ns clock that wakes two of them (`units: [orr::Wake; 2]`, named in its
task's `wakes`):

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

// In the feeder's task:
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

A wake and the writer's `close()` go in the same order whether the gate
controls one task or several:

| Case | Result |
|---|---|
| The writer calls `close()` at timepoint τ, and a `wake()` comes at τ | The gate is open from the timepoint after τ (the wake wins) |
| `wake()` at τ; the writer calls `close()` at τ′, the timepoint after τ | The gate is open at τ′, where its tasks run; closed from the timepoint after τ′ (`close()` wins) |
| `wake()` at level 0 of edge e; gated tasks at levels 1 to 5 of that edge | Open from level 1: the later levels of the edge run |
| `wake_after(d)` at τ; the writer calls `close()` before τ + d | The wake wins: the gate opens from its first point at or after τ + d |
| `wake_after(d)` at τ; the writer calls `close()` at τ + d or later | `close()` wins |
| Several tasks wake one gate at one timepoint | One wake |

**Who acts first decides whether a wake is in time.** A wake cannot make
the woken task run in the timepoint that sends it. Whether that matters
depends on the order of the two tasks in the cycle
([Who acts first](./components#who-acts-first)):

| Producer | Consumer | A value pushed while the consumer is closed is popped |
|---|---|---|
| `&'latest mut FifoTx` (acts second) | `&mut FifoRx` (acts first) | in the next cycle, as with the gate always open: in time |
| `&mut FifoTx` (acts first) | `&'latest mut FifoRx`, triggered at `clock + delta(1)` | in the same cycle, as with the gate always open: in time |
| `&mut FifoTx` (acts first) | `&'latest mut FifoRx`, at the same level | **a cycle late**: with the gate always open it is popped in the same cycle |

In the third pairing, move the consumer to a later delta level, or keep
its gate open while a value can arrive. To check a gated unit, run the
same model with its gate never closing: the results must be the same, at
the same cycles.

**Through submodules.** `#[orr::wake(forward)]` passes wakes across a
module's boundary:

- on an `orr::Wake` field, it passes a submodule's wake out: the module's
  init connects the submodule's wake to it, and the parent connects it as
  it would a task's wake. A task does not wake through it.
- on an `orr::WakeRelay` field, which may be an array, it passes wakes in:
  the module's init connects it to a submodule's relays.

Building the simulation fails when a forwarding wake has no wake
connected to it, or a forwarding relay leads to no gate.

**A wake or an ungated event.** The `wake` trigger in the example in
[Clock gates](#clock-gates) is an ungated event: it runs the task when it
arrives, and the task opens the gate. Use a wake when the woken unit finds
its work in its own state or in its FIFOs; use an ungated event when the
wake-up carries a value, or the unit must act at the moment it arrives.
