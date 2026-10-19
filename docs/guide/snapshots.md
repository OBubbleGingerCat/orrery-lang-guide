# Snapshots

Snapshots save a running simulation's complete state at regular
boundaries, without stopping it, so that a long run can be resumed from
shortly before a problem instead of from the start.

## Taking snapshots

```rust
use orrery_lang as orr;

let config = orr::SimConfig {
    snapshots: Some(orr::SnapshotConfig::new(
        orr::SnapshotEvery::Time(orr::Time::us(100)),
        "snapshots".into(),
    )),
    ..orr::SimConfig::default()
};
let mut sim = plan.into_sim(config)?;
sim.run_until(orr::Time::ms(5))?;
```

Boundaries are at every 100 µs (100, 200, …; never at 0). They can also
follow a clock, every *N* of its cycles:

```rust
let clock = plan.top.core_clock;            // a clock's handle, taken before into_sim
orr::SnapshotEvery::Cycles { clock, cycles: 100_000 }
```

A snapshot at a boundary *B* is the state after every timepoint before
*B* and before any at or after it.

`SnapshotConfig::new(every, dir)` keeps the newest 3 snapshots (`keep`)
and makes every 8th one full (`full_every`); the others are
**incremental**, holding only the memory cells written since the one
before. Each snapshot is a file in `dir` named by its boundary in
picoseconds, twenty digits wide (`00000000000100000000` for 100 µs);
`orr::snapshot_path(dir, time)` gives the full name. Files are written by
a background thread, through a temporary file and a rename, so a file
that exists is complete.

## Best effort

Snapshots never hold the simulation up: they are compressed and written
to disk in the background while it runs on. In exchange, a boundary is
**skipped** if the previous snapshot is still being written when the run
reaches it. Every snapshot that is written is exact, and the same on any
number of workers, and taking snapshots never changes the results of a
run.

```rust
if let Some(status) = sim.wait_snapshots() {
    println!("{} written, {} skipped, latest {:?}", status.written, status.skipped, status.latest);
}
```

`snapshot_status()` reports without waiting; `wait_snapshots()` waits for
the writer first. A write failure is reported there and does not fail the
run.

A model that runs faster than snapshots are written skips boundaries: run
to 100 ns at once with a snapshot every 30 ns, and the one at 30 ns may
still be writing when 60 and 90 pass. Where every boundary matters, in a
test say, run in steps and wait in between:

```rust
for until in [30, 60, 90, 100] {
    sim.run_until(orr::Time::ns(until))?;
    sim.wait_snapshots();
}
```

## Restoring

Give the latest snapshot file to `SimPlan::new`:

```rust
let latest = status.latest.expect("a snapshot");   // from wait_snapshots() above
let plan = orr::SimPlan::<Top>::new(params, Some(latest))?;
let mut sim = plan.into_sim(config)?;
println!("restored from {:?}", sim.restored_from());
sim.run_until(orr::Time::ms(5))?;   // continues exactly as the original run
```

An incremental snapshot finds the files it builds on by name, in the same
directory. The restored simulation may use a different number of workers
or a different configuration; it continues exactly as the original run
would have:

- events in flight at the boundary arrive as they would have, and gates,
  dropped events and gated edges carry on from where they were;
- if a task had already stopped the simulation before the boundary, and
  the boundary falls before the end the stop set
  ([Stopping](./running#stopping)), the snapshot keeps the stop: the
  restored simulation ends where the original would have, with the same
  `finished()`;
- writing state with `state_mut` before the first run of the restored
  simulation is the same as writing it between two runs of the original,
  the first of which ended just before the boundary.

The one exception is an event in flight on a source that triggers no
task: it is not saved, so after a restore it no longer counts towards a
multi-drive or `event_capacity`.

Until its first run, a restored simulation's `now()` is `None`;
`next_timepoint()` already gives the next timepoint.

A snapshot only restores into the same model: the file records a
fingerprint of the model (its structure, types, clocks, connections and
reset values), and a mismatch is an error. Task bodies and parameters that
only tasks read are not part of the fingerprint, so a snapshot can be
restored with a fixed bug in a task, which is often the point.

## External ports

Sources that tasks write (`&mut External<S>`) must implement `save` and
`restore` ([External ports](./external-ports#snapshots)); with snapshots
configured, building a simulation whose sources cannot be saved fails and
names the port. Read-only sources are not saved: on restore, the host
binds them again.
