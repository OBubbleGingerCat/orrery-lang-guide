# 10. Getting faster with more threads

This chapter puts the earlier ones together. It covers when a model gets
faster with more threads, what to look at before and after one run, and
what to do when it does not get faster. The guide states the facts in
[When more threads help](../guide/running#when-threads). Here they are
rules to check in order, each with an example from this tutorial.

## What a cycle costs on several threads {#what-a-cycle-costs}

On several threads, a run pays for its execution units in every cycle,
besides the work its tasks do. This is what the runs behind this
tutorial showed:

- Each kind of task that comes in many instances makes about
  min(instances, *N* − 1) units on *N* threads, and a chain of events
  without a delay makes more (rule 5 below). `sim.load_report().units`
  gives the count.
- Each unit takes time in every cycle in which anything runs, whether
  its tasks run or not. In a small model on 8 threads, seven kinds of
  task with their gates closed for the whole run took about as long as
  seven kinds that ran in every cycle.
- A cycle in which no task runs is not always skipped. In one model with
  a sink triggered by events alone, with no clock and no gate, every
  unit was paid for in every cycle on 8 threads, though tasks ran in
  fewer than a quarter of them. Without the sink, each unit was paid for
  in about a third of the cycles.

So the cost of a cycle grows with the kinds of task and the threads,
more than with the work done in it. Fewer kinds of task (rule 4 below)
is often the first thing to try.

## Check these in order

**1. Two threads are not faster than one.** A run on *N* threads has one
thread handing out work and *N* − 1 running tasks. On 2 threads, one
thread runs tasks, as on 1, and pays for the handing out too. On 2
threads the examples in this tutorial are as fast as on 1, or a little
slower.
Parallelism starts at 4. At most *N* − 1 execution units run at once,
so 8 threads can be at most about 7 times as fast as 1.

**2. The instances that work must not be neighbours.** Orrery divides
the instances of one kind of module among the execution units in runs of
neighbours. If only a few instances work in a cycle and they are
neighbours, they all fall into one unit, and run one after another
however many threads there are. This is the first thing to check,
because it can make every other number look fine while 8 threads are no
faster at all.

**3. Each cycle must have enough work for the units it is divided into.**
Every execution unit costs something in every cycle in which anything
runs, whether its gates are open or not. Nothing tells in advance
whether the work pays for that: try it (Rule 3 below).

**4. Fewer kinds of task, fewer units.** Each kind of task that comes in
many instances gives about min(instances, *N* − 1) units. So every extra
kind of task adds up to *N* − 1 units, paid for in every cycle, even when
all its gates are closed. On 8 threads that was about 0.3 µs a cycle per
kind, in a small model.
Tasks that always run together in the same cycles are candidates to
merge (chapter 6, to come). So is a small unit that only feeds one
other, even when the two do not run in the same cycles: a source that
fills one input buffer can be written into the unit that owns the
buffer, and that is one kind of task less. In a model of a network of
switches, the sources kept apart took 1.1–1.2 times as long as the
sources written into the first stage on 1 thread, and up to 1.9 times
on 8; they were slower for every input and thread count.

**5. Lines with a delay run side by side** ([chapter 5](./lines)). A
chain of lines without a delay is a chain of units, each waiting for the
one before.

## Rule 2: spread the working instances

Sixty-four lanes, of which only 4 or 8 work; the rest close their gates
for good. The working lanes are either neighbours (lanes 0, 1, 2, ...)
or spread out (every 16th or 8th lane):

```rust
#[orr::module]
pub struct Lane {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub working: bool,
    #[orr::constant]
    pub work: u64,
    gate: orr::Gate,
    pub state: u64,
}

#[orr::module_impl]
impl Lane {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, working: &'constant bool, work: &'constant u64, gate: &mut orr::Gate, state: &mut u64) -> Result<(), orr::SimError> {
        if !*working {
            gate.close(); // for good
            return Ok(());
        }
        *state = mix(*state ^ 1, *work);
        Ok(())
    }
}

#[orr::module]
pub struct Lanes {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("working lanes", default = 4usize)]
    working: usize,
    #[orr::param("how far apart the working lanes are: 1 for neighbours", default = 1usize)]
    stride: usize,
    #[orr::param("mixer rounds a working lane does in a cycle", default = 1000u64)]
    work: u64,
    #[orr::trigger]
    clock: orr::Clock,
    #[orr::submodule]
    lanes: [Lane; orr::DynShape(n)],
}

#[orr::module_impl]
impl Lanes {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        let (working, stride) = (*module.working, *module.stride);
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, lane) in module.lanes.iter().enumerate() {
            ctx.connect(module.clock, lane.clock)?;
            ctx.set_const(lane.working, i % stride == 0 && i / stride < working)?;
            ctx.set_const(lane.work, *module.work)?;
        }
        Ok(())
    }
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    let end = orr::Time::ps(PERIOD_PS * 20_000 - 1);
    for working in [4usize, 8] {
        let mut cases = Vec::new();
        for (stride, cut, how) in [
            (1usize, orr::CutPolicy::Runs, "neighbours, Runs"),
            (1, orr::CutPolicy::Dealt, "neighbours, Dealt"),
            (64 / working, orr::CutPolicy::Runs, "spread out, Runs"),
        ] {
            let file = params(&format!("ch10-w{working}-s{stride}"), &format!("[top]\nworking = {working}\nstride = {stride}\n"))?;
            cases.push(Case::new(
                format!("{working} of 64 working, {how}"),
                orr::SimConfig { cut, ..orr::SimConfig::default() },
                move |config| Ok(orr::Sim::new(orr::SimPlan::<Lanes>::new(Some(file.clone()), None)?, config)?),
                move |sim| {
                    sim.run_until(end)?;
                    let mut sum = 0;
                    for lane in sim.top().lanes.iter() {
                        sum = mix(sum ^ *sim.state(lane.state)?, 1);
                    }
                    Ok(sum)
                },
            ));
        }
        compare(&threads, 3, &cases)?;
    }
    Ok(())
}
```

Seconds to run (rough). The execution units are 1, 1, 3 and 7:

| | 1 | 2 | 4 | 8 | 8 threads vs 1 |
|---|---|---|---|---|---|
| 4 neighbours, `Runs` | 0.154 | 0.157 | 0.159 | 0.159 | 0.96× |
| 4 neighbours, `Dealt` | 0.153 | 0.157 | 0.082 | 0.045 | 3.38× |
| 4 spread out, `Runs` | 0.154 | 0.157 | 0.083 | 0.045 | 3.42× |
| 8 neighbours, `Runs` | 0.305 | 0.309 | 0.310 | 0.311 | 0.98× |
| 8 neighbours, `Dealt` | 0.305 | 0.309 | 0.121 | 0.084 | 3.63× |
| 8 spread out, `Runs` | 0.305 | 0.309 | 0.121 | 0.084 | 3.64× |

- **Neighbours get nothing from more threads.** With the default cut,
  `Runs`, 8 threads take as long as 1.
- **`cut: orr::CutPolicy::Dealt` deals the instances out one by one**,
  so neighbours go to different units. 8 threads are then 3.4 to 3.6
  times as fast: about as fast as with the lanes spread out by hand.
- **Find out beforehand whether your working instances are
  neighbours**, from how your model hands out work. A dispatcher that
  hands out work round robin, starting after the last unit it gave work
  to, gives neighbours whenever there is little to hand out. Today no
  report lists which instances are in which unit, so you work it out
  from the model.
- **`Dealt` has a cost**: where every instance has a little work in every
  cycle, it can be slower, by a few percent to a fifth, depending on the
  model. Measure both.
  - In the GPU model that comes with Orrery, the command processor hands
    thread blocks to the next cores in turn. Programs with few thread
    blocks therefore keep their busy cores together.
  - Timed by the Orrery developers on the eleven test programs, 8
    threads, whole processes: dealing out took 0.71 to 0.89 of the
    default's time on the programs where only a few cores work at a
    time. It took 2 to 7% more on those where every core works in every
    cycle.

## Rule 3: is there enough work a cycle?

No size of model and no number of execution units says in advance
whether its work in a cycle pays for the units on several threads.
Choose by trying, as the guide says
([How many threads](../guide/running#how-many-threads)):

1. Take a stretch of a typical input. Run it once on 1 thread and once
   on the most threads (the free physical cores of one cache domain),
   and keep the faster. Compare the runs alone, not building the
   simulation.
2. Do it for each kind of input the model runs. A busy input and a
   mostly idle one can come out on opposite sides: one model ran faster
   on 8 threads than on 1 on an input with much work in every cycle, and
   slower on one with little.
3. Read `sim.load_report().units` on the most threads. It does not say
   which is faster, but it helps explain the result.

Why several threads can be slower: every unit takes time in every cycle
in which anything runs ([What a cycle costs](#what-a-cycle-costs)).
When the work in a cycle is little, that costs more than running side by
side saves. And where the work cannot be spread, more work does not
help: a model written as one loop is one unit ([chapter 1](./measure)),
and four neighbouring lanes with `Runs` fall into one unit (above). So check rules 1 and 2
first.

As a rule, a small model, or one with little work in each cycle, runs on
1 thread. Several threads pay off on large runs: many instances, each
with much to do in a cycle.

## Configuration

- `workers_per_group`: the threads. Run 1 and the most (the free
  physical cores of one cache domain) once each, and keep the faster;
  try counts in between only to leave cores for something else. 2 is
  never faster than 1.
- `cut`: `Runs` (the default) or `Dealt`. When few instances work at a
  time, try both.
- `cpus`: the CPUs the run may take its cores from. Give it CPUs nothing
  else uses, or other work will slow it unevenly.
- `host_spin` and `idle_spin`: set both to 0 when the process may use
  only as many CPUs as the run has threads: the thread that calls
  `run_until` then has none of its own.

See the guide: [Configuration](../guide/running#configuration),
[Threads](../guide/running#threads).

## When it does not get faster

Go through what you can see from outside:

1. **The load report**: the execution units on each thread count. Are
   there many more than the instances that work in a cycle? Then look
   for kinds of task that could be merged, and for chains of lines
   without a delay.
2. **Your own counts**: in a typical cycle, how many instances work,
   and which ones? Are they neighbours? Then try `Dealt`.
3. **The run on 1 thread against the most.** If the most threads are
   slower although the work is spread (rules 1 and 2), the work in a
   cycle is too little for that many units. Fewer units (rules 4 and 5),
   or 1 thread, is then the answer.
4. **The stop report** (`reports: orr::Reports::All`) gives the run's
   wall time, and on several threads how the work was shared out.

**Changing the number of threads in the middle of a run** is possible
today, through a snapshot: take one at the switch, then build a new
simulation from it on another number of threads. It costs about as much
as building the simulation again. On the GPU model's eleven test
programs, no program had long stretches where a different number of
threads would pay for that. So pick one thread count for a whole run.

## Checklist

- In a typical cycle, how many instances work? Are they neighbours? Did
  I try `Dealt`?
- How many execution units are there on 1, 2, 4 and 8 threads?
- Is there a kind of task that runs, or keeps its gate closed, in every
  cycle while doing almost nothing? (On several threads it is paid for
  either way.)
- Do the lines that have a delay in the hardware declare it?
- Did I run 1 thread and the most once each, on each kind of input,
  comparing the run alone, and skip 2?
