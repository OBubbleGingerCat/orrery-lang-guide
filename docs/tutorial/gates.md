# 2. Close the gate of an idle unit

**The rule.** Some units have nothing to do until something reaches them:
a job, a request, a value in a FIFO. When such a unit runs out of work,
close its clock gate at the end of its task: `gate.close()`. The work
that comes next opens it again. It comes either as an event that is not
behind the gate, which runs the task, or as a wake ([chapter 4](./wakes)).

Two details matter:

- **Write the gate however reads most clearly.** Writing `open()` to a
  gate that is already open costs next to nothing, so there is no need
  to check `is_open()` first (the example below does; it need not).
- **Check it by switching it off.** Give the model a parameter that keeps
  the gate open. The results must be the same, at the same cycles, with
  the gate closing and without.

See also the guide: [Clock gates](../guide/time#clock-gates) and
[Wakes](../guide/time#wakes).

## Example: workers that sleep between jobs

A dispatcher hands jobs to 64 workers. A job takes a worker `len`
cycles, and each cycle of it is about 0.2 µs of work. A job arrives as an
event one cycle after it is sent. That event is not behind the gate, so
it runs the task even while the gate is closed:

```rust
#[orr::module]
pub struct Worker {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    /// Jobs: not behind the gate, so one can arrive while it is closed.
    #[orr::trigger]
    pub jobs: orr::EventRelay<u64>,
    #[orr::constant]
    pub gating: bool,
    #[orr::constant]
    pub work: u64,
    #[orr::constant]
    pub len: u64,
    gate: orr::Gate,
    /// Cycles of work left on the jobs taken so far.
    left: u64,
    pub state: u64,
}

#[orr::module_impl]
impl Worker {
    #[orr::task(triggers = (clock, jobs))]
    #[allow(clippy::too_many_arguments)]
    fn cycle(
        ctx: &mut orr::TaskCtx,
        gating: &'constant bool,
        work: &'constant u64,
        len: &'constant u64,
        gate: &mut orr::Gate,
        left: &mut u64,
        state: &mut u64,
    ) -> Result<(), orr::SimError> {
        let (clock, job) = ctx.triggered_triggers();
        // A cycle of work on the jobs taken before now.
        if clock.is_some() && *left > 0 {
            *state = mix(*state ^ *left, *work);
            *left -= 1;
        }
        // A job that arrives now is worked on from the next cycle.
        if let Some(job) = job {
            *state ^= *job;
            *left += *len;
        }
        if *gating {
            if *left == 0 {
                gate.close(); // until the next job
            } else if !gate.is_open() {
                gate.open(); // a job came while it was closed
            }
        }
        Ok(())
    }
}
```

The order inside the task keeps the two variants the same:

- the cycle's work is done on the jobs taken before now;
- a job that arrives now is taken after that work.

So a job that arrives at a closed worker starts on the next edge. That is
also when it would start at a worker whose gate never closes.

The dispatcher and the driver:

```rust
#[orr::module]
pub struct Office {
    #[orr::param("workers", default = 64usize)]
    n: usize,
    #[orr::param("a worker gets a job every this many cycles", default = 100u64)]
    every: u64,
    #[orr::param("cycles a job takes", default = 10u64)]
    len: u64,
    #[orr::param("mixer rounds in a cycle of a job", default = 100u64)]
    work: u64,
    #[orr::param("whether a worker closes its gate when it has no job", default = true)]
    gating: bool,
    #[orr::trigger]
    clock: orr::Clock,
    #[orr::submodule]
    workers: [Worker; orr::DynShape(n)],
    #[orr::trigger(min_delay = orr::Time::ps(PERIOD_PS))]
    jobs: [orr::EventSource<u64>; orr::DynShape(n)],
}

#[orr::module_impl]
impl Office {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, worker) in module.workers.iter().enumerate() {
            ctx.connect(module.clock, worker.clock)?;
            ctx.connect(module.jobs[i], worker.jobs)?;
            ctx.set_const(worker.gating, *module.gating)?;
            ctx.set_const(worker.work, *module.work)?;
            ctx.set_const(worker.len, *module.len)?;
        }
        Ok(())
    }

    /// Hands worker `i` a job, arriving at the next edge, every `every`
    /// cycles, the workers taking turns.
    #[orr::task(triggers = (clock,), emits = (jobs,))]
    fn dispatch(ctx: &mut orr::TaskCtx) -> Result<(), orr::SimError> {
        let edge = (ctx.now().ps() / u128::from(PERIOD_PS)) as u64;
        let every = *ctx.module().every;
        for (i, mut job) in ctx.emit_triggers().0.iter().enumerate() {
            if (edge + 7 * i as u64) % every == 0 {
                job.emit(edge, orr::Time::ps(PERIOD_PS))?;
            }
        }
        Ok(())
    }
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    // Busy a tenth of the time, and all the time.
    for (len, cycles) in [(10u64, 100_000u64), (100, 20_000)] {
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        let mut cases = Vec::new();
        for gating in [false, true] {
            let name = format!("ch2-len{len}-gating{gating}");
            let file = params(&name, &format!("[top]\nlen = {len}\ngating = {gating}\n"))?;
            cases.push(Case::new(
                format!("busy {}%, gating {gating}", len),
                orr::SimConfig::default(),
                move |config| Ok(orr::Sim::new(orr::SimPlan::<Office>::new(Some(file.clone()), None)?, config)?),
                move |sim| {
                    sim.run_until(end)?;
                    let mut sum = 0;
                    for worker in sim.top().workers.iter() {
                        sum = mix(sum ^ *sim.state(worker.state)?, 1);
                    }
                    Ok(sum)
                },
            ));
        }
        let results = compare(&threads, 3, &cases)?;
        // Closing the gate must not change the result.
        assert_eq!(results[0].1, results[1].1, "gating changed the result");
    }
    Ok(())
}
```

The driver measures both variants, at two loads:

- a worker busy a tenth of the time (a job of 10 cycles every 100);
- a worker busy all the time (a job of 100 cycles every 100).

It fails if closing the gate changed the result.

## What it does to the three measures

Seconds to run (rough). The execution units are 2, 2, 4 and 8 on 1, 2,
4 and 8 threads (the workers, and the dispatcher):

| | 1 | 2 | 4 | 8 | 8 threads vs 1 |
|---|---|---|---|---|---|
| busy a tenth of the time, gate never closed | 0.263 | 0.293 | 0.139 | 0.089 | 2.96× |
| busy a tenth of the time, gate closed when idle | 0.179 | 0.207 | 0.115 | 0.079 | 2.26× |
| busy all the time, gate never closed | 0.254 | 0.261 | 0.096 | 0.0486 | 5.22× |
| busy all the time, gate closed when idle | 0.254 | 0.260 | 0.095 | 0.0481 | 5.28× |

- **Busy a tenth of the time.** Closing the gate is **better**:
  - 1 thread is 1.47 times as fast;
  - 8 threads are 1.12 times as fast;
  - no thread count is slower.
  
  On 8 threads it is faster, but the speedup falls from 3.0 to 2.3. In
  this example closing the gate took a third off the time on one thread,
  and a ninth on 8 threads. On several threads, though, an execution unit
  takes time in every cycle where anything runs, even when the gates of
  all its workers are closed. So the gate saves more of 1 thread's time
  than of 8 threads'. That is how it should be: compare the times, not
  the speedups.
- **Busy all the time.** In this example closing the gate saves nothing
  and costs nothing (within noise on every thread count). Here a worker's
  gate never closes after its first job: the next job comes in the cycle
  the last one ends.
  A gate that closes and opens again often does cost; see
  [When not to](#when-not-to).
- **Reading** gains two things to keep in mind:
  - the condition the gate closes on (`left == 0`, one line);
  - what opens it again (the job event, one line).

  When the condition says something about the hardware ("no job left"),
  it reads as part of the unit's behaviour.

## In the GPU model

The GPU model that comes with Orrery closes the gates of nearly all its
units. An L2 bank, for example, is two tasks:

- The `reply` step moves the head of its queue into the reply network.
  When it leaves the queue empty, it closes its gate.
- The `cache` step pushes replies into that queue. Each push also wakes
  the `reply` step. The model does this by wrapping the sending end of
  the queue in a small type whose `push` also wakes.

Neither step writes `open()`: a closed step is opened only by a wake.

## Steps that share a gate {#shared-gate}

A unit often runs in steps, at several levels of one clock: fetch,
execute, retire. With no work, all the steps should sleep; with work,
all should wake. Such steps share one gate: their `triggers` name one
clock field with `gate = gate`. Only one task may write the gate, here
`retire`, at the last level, which takes the jobs too. This chapter's
worker, split into three steps, each job's event carrying its cycles of
work:

```rust
#[orr::module]
pub struct Worker {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    /// Jobs, each its cycles of work: not behind the gate.
    #[orr::trigger]
    pub jobs: orr::EventRelay<u64>,
    #[orr::constant]
    pub gating: bool,
    gate: orr::Gate,
    /// Cycles of work left on the jobs taken so far.
    left: u64,
    pub fetched: u64,
    pub executed: u64,
}

#[orr::module_impl]
impl Worker {
    /// Level 0: fetches, while there is work.
    #[orr::task(triggers = (clock,))]
    fn fetch(
        _ctx: &mut orr::TaskCtx,
        left: &u64,
        fetched: &mut u64,
    ) -> Result<(), orr::SimError> {
        if *left > 0 {
            *fetched = mix(*fetched, *left);
        }
        Ok(())
    }

    /// Level 1: executes what was fetched.
    #[orr::task(triggers = (clock + delta(1),))]
    fn execute(
        _ctx: &mut orr::TaskCtx,
        left: &u64,
        fetched: &u64,
        executed: &mut u64,
    ) -> Result<(), orr::SimError> {
        if *left > 0 {
            *executed = mix(*executed, *fetched);
        }
        Ok(())
    }

    /// Level 2: retires a cycle of work, takes the jobs, and closes
    /// the gate when no work is left: the gate's one writer.
    #[orr::task(triggers = (clock + delta(2), jobs))]
    fn retire(
        ctx: &mut orr::TaskCtx,
        gating: &'constant bool,
        gate: &mut orr::Gate,
        left: &mut u64,
    ) -> Result<(), orr::SimError> {
        let (clock, job) = ctx.triggered_triggers();
        if clock.is_some() && *left > 0 {
            *left -= 1;
        }
        if let Some(job) = job {
            *left += *job;
        }
        if *gating {
            if *left == 0 {
                gate.close(); // until the next job
            } else if !gate.is_open() {
                gate.open(); // a job came while it was closed
            }
        }
        Ok(())
    }
}
```

- `retire` closes the gate at the last level (level 2), so it is closed
  from the next edge: all three steps have run at the edge where it
  closes. A writer at the first level would stop the later levels at
  once, at the same edge.
- A job is an ungated event that arrives at level 0 of an edge.
  `retire` opens the gate there, and it is open from level 1: `execute`
  and `retire` run at that edge, `fetch` from the next. With the gate
  never closing it is the same: `fetch` reads `left` as `&u64`, before
  `retire` writes it at the same timepoint, and reads 0.
- `gate.gated_edges()` has three counts, in the order of the tasks'
  names: `execute`, `fetch`, `retire`. `fetch`'s is larger than the
  other two: at the edges where a job arrived and the gate opened from
  level 1, level 0 was still closed.
- Check with gating off: with `gating` `false` the gate never closes.
  Both ways were run, and the three workers' state is the same.

A gate used to control one task only. Such a unit needed three
`ClockRelay` fields and three gates, each step closing its own, and the
jobs connected to all three steps, each opening its own gate. Now one
gate and one writer do. Steps that are to sleep and wake apart still
want gates of their own.

## When not to

- **A unit that is almost always busy** gains nothing from a gate, and
  can lose. A gate saves the calls it skips, and a close with the reopen
  after it costs about as much as one or two calls. This chapter's
  workers, given a one-cycle job every *k* cycles (so that each close
  saves *k* − 2 calls), against the same workers with the gate kept
  open, on 1 thread: closing cost a little at *k* = 2 and 3, saved a
  little at 4, and saved clearly from 6 on. In a mesh of routers that
  closed when their buffers ran empty, busy traffic made 1 thread 13–16%
  slower than with the gates kept open.
- **A gate kept open is not free either.** On 1 thread the same workers
  took 4 to 28% longer with the gate kept open than with no gate at all,
  the idler the longer; the routers, 4 to 13%. Against no gate, closing
  only broke even at *k* = 6, and saved under a tenth even when the
  workers were idle nine cycles in ten. On 8 threads the three were
  mostly within a tenth of each other. So a gate is worth it only where
  the unit is idle a lot. Measure with the gate and with no gate, and
  where it does not pay, leave the gate out: no `gate` on the trigger.
  That is also one condition less to read. No parameter switches a gate
  out: a module with no gate is another module. To measure it, copy the
  module, take `gate` off its triggers and drop its gate code, and run
  the two in turns (the measurement above did that with its workers).
- **A unit that knows when its next work comes** (a fixed latency, a
  timer) should close until then, not wait to be woken
  ([chapter 3](./due)). Then no other unit has to send anything.
- **An event that reaches a gated trigger while the gate is closed is
  dropped** (and counted). The trigger that brings the work must not be
  behind the gate, or the sender must wake the unit first. Run your
  tests with `SimConfig { fail_on_drop: true, .. }` so that a dropped
  event fails the run.

## Checklist

- Can I say in one sentence when each gate closes?
- Of a gate several steps share, does one task write it? Is it at the
  last level?
- Does every way work can reach the unit open its gate? (An ungated
  event, or a wake from each sender: chapter 4's table.)
- With the gate never closing, are the results the same, at the same
  cycles? Did the tests run with `fail_on_drop`?
- Did I measure all four thread counts with the gate, and with no gate
  at all (not only with it kept open)?
