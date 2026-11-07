# Errors and debugging

Most modeling mistakes are caught before anything runs: by the compiler,
when the plan is built, or when the simulation loads, named by the
instance paths in your model. The rest are caught when they happen, with
the timepoint. Every error converts to an `orr::SimError` with `?`.

## When it compiles

The compiler rejects:

- a task parameter whose type is not the field's:
  ``a task parameter of type `u32` cannot bind a field of type `u64` ``;
- a connection between kinds that do not connect: a clock to an event
  relay, anything into a clock or an event source;
- a task that moves its gate out of the call
  ([Clock gates](./time#clock-gates)).

## When the plan is built

`SimPlan::new` fails with a message for:

**Parameters**
- a table that names no instance, or a key that is not a parameter:
  `top.x is not a parameter`;
- a value of the wrong type, or a failed `validate`:
  `top.limits: validation failed: ...`;
- a `DynShape` length of zero:
  `top.n: dynamic shape must be greater than zero`.

**Connections**
- a relay, reference or interface never connected:
  `x: unbound handle (referenced by y)`;
- one connected twice: `x: duplicate driver (from y)`;
- a cycle of references: `connection cycle: a -> b -> a`;
- a clock never configured: `top.clock: Clock was not initialized`, or
  configured twice: `top.clock: clock initialized more than once`;
- a [constant](./modules#constants) never assigned, or assigned twice:
  ``top.child: constant `depth` is assigned twice``.

**Tasks**
- two parameters of one task on one location with different modes:
  ``state parameters `value` and `alias` conflict on ...``;
- a gate written by a task other than the one whose triggers it
  controls.

## When the simulation loads {#when-the-simulation-loads}

`orr::Sim::new` and `orr::Sim::with` fail with an `orr::LoadError` for:

**Ordering** (`LoadError::Hazards`), which Orrery works out from the
access modes ([State and tasks](./state-and-tasks#access-modes)):

- two tasks that certainly write one location in the same timepoint:

  ```text
  write conflict: %WriteConflict.bump at top and %WriteConflict.clear at top
  both write top.value (both triggered by clock top.clock (1ns phase 0ps))
  ```

- a cycle among tasks that always run together:

  ```text
  dependency cycle (triggered by clock top.clock (1ns phase 0ps)):
  %Producer.send at top.producer writes top.fifo.slots before
  %BothFirst.receive at top.consumer reads it (latest); ...
  ```

  For a FIFO, this is both sides claiming to act first, or both after
  ([Components](./components#who-acts-first)), or a task looking at the
  consumer's end with `&'latest FifoRx` when the consumer acts first
  ([Components](./components#looking-without-popping)).

**The configuration** (`LoadError::Config`, with the field): a
`delta_limit` above 62, `groups` other than 1, `workers_per_group` of 0
or more than an L3 domain allows, and what does not run on several
workers yet ([Workers](./running#workers)).

**Setting up** (`LoadError::Init`, with the stage):

- `clocks`: two clocks whose edges come within less than 1 fs of each
  other without meeting ([Clocks](./time#clocks));
- `external`: an external port left unbound,
  `external port %Ports.log row 0 is not bound`, or bound twice,
  `top.log: external port is bound twice`;
- `snapshots`: with snapshots configured, an external port that tasks
  write (`&mut`) whose source cannot be saved, or a snapshot directory
  that cannot be made.

**Restoring** (`LoadError::Restore`, with the file): a snapshot that does
not read, or that is of another model ([Snapshots](./snapshots#restoring)).

### Warnings {#warnings}

Conflicts and cycles that only *may* happen, because events are involved,
are warnings instead, checked when the simulation runs. The load prints
each to stderr, `warning: ...`, unless `reports` is `Off`, and keeps them
in its report:

```rust
for warning in &sim.load_report().warnings {
    eprintln!("{warning}");
}
```

```text
possible write conflict, checked at run time: %MayConflict.on_a at top and
%MayConflict.on_b at top both write top.value (triggered by event top.a_out
and event top.b_out)
```

## While it runs

`run_until` and `step` fail with an `orr::RunError`. What happens in the
model is a `RunError::Occurrence`, which ends the run: its `at` is the
timepoint, `call` the task and instance (`None` when no one task is at
fault), `kind` an `orr::FailureKind`, and `message` what it displays:

| Error | `kind` | Cause |
|---|---|---|
| `%Sm.cycle at top.sms[3] failed at 1200ns: ...` | `Task(Body)` | a task returned an error |
| `... panicked: ...` | `Task(Panic)` | a task panicked; the error names the task and the instance that panicked |
| `multi-drive at 10ns+1: A and B both emit to L` | `MultiDrive` | two events for one location and timepoint |
| `write conflict at 37ns: A and B both write L` | `WriteConflict` | a possible write conflict happened |
| `dependency cycle among ...` | `BlockCycle` | tasks that may run together formed an ordering cycle in this timepoint |
| `more than 30 delta cycles at 5ns` | `DeltaLimit` | a chain of delta cycles that never settles (`delta_limit`) |
| `more than N events in flight from one location` | `Task(Capacity)` | `event_capacity` exceeded |
| `event delay ... is below min_delay ...` | `Task(Delay)` | an emit outside the source's declared delays, or with a delay of zero |
| `... is after the latest time a run ... reaches, ...` | `Task(Range)` | an emit that would arrive after the latest time a run reaches ([Delta cycles](./time#delta-cycles)) |
| `a wake delay must be Time::DELTA or a positive physical time` | `Task(WakeDelay)` | a `wake_after` with a delay of zero ([Wakes](./time#wakes)) |
| `... is after the latest time a run ... reaches, ...` | `OutOfRange` | a `step` whose next timepoint is after the latest time a run reaches |
| `an event at ... reached the trigger ..., whose gate ... is closed, and dropped events are errors` | `DroppedEvent` | an event dropped at a closed gate, with `fail_on_drop` ([Clock gates](./time#clock-gates)) |

The other `RunError`s:

- `Request`: a request that does not hold, such as a `run_until` past
  the latest time a run reaches, or to `Time::DELTA`, or a handle of
  another simulation. Nothing runs, and the simulation goes on. A `step`
  whose next timepoint is past that time is not a request: it fails with
  `OutOfRange`, as above.
- `Failed`: an earlier run failed, `an earlier run failed; the
  simulation cannot continue`.
- `EventLog`: the [event log](./running#event-log)'s sink returned an
  error; `LogClosed`: a run after `close_event_log`.

### Which error is reported

When more than one thing fails, Orrery reports the failure that comes
first in serial order: the order in which a run that takes one task at a
time would meet them, timepoint by timepoint and, within a timepoint, in
a fixed order that respects the access modes. The same model with the
same input reports the same error, whatever the number of workers.

- A failure that would only come after the end of a stop is not
  reported: the run ends with the stop ([Stopping](./running#stopping)).
- With `fail_on_drop`, a dropped event fails its timepoint before any
  task in it runs.
- A task whose event makes a multi-drive, and that then returns an error
  or panics, reports the multi-drive.

After an error, everything that comes before the failure in serial order
has run, and the host can read the state it left with `sim.state`. The
failing task may have written only part of what it would have, and work
after it may have run too: that part of the state is not specified, and
may differ with the number of workers. Every later `run_until` and `step`
returns an error.

## Debugging

- **Step.** `sim.step()` runs one timepoint; `sim.next_timepoint()` says
  which. Inspect state with `sim.state(handle)` between steps.
- **Check the run.** With `SimConfig { checks: true, .. }`, on one
  worker, `sim.check_trace()` checks after a run that every two accesses
  to the same state, at least one of them a write, ran in serial order.
- **Compare worker counts.** For a model that runs on several workers,
  the results of a run that ends without an error, and the error a run
  reports, never depend on their number. A model that gives other such
  results on another number of workers is a bug in Orrery. The state read
  after an error may differ, as [Which error is
  reported](#which-error-is-reported) says: that is not a bug.
- **Snapshot and restore.** Take snapshots in a long run and restore the
  one before the problem, with a `step` loop or more logging
  ([Snapshots](./snapshots)).
