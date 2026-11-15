# 1. Measure first

**The rule.** Before you change how a model is written, keep the old way
and measure the two in one run of your driver, taking turns. Measure four
things:

- the time to run on 1, 2, 4 and 8 threads;
- the time to build, apart from the run;
- the speedup, 1 thread's time ÷ *N* threads' time;
- the execution units the simulation is divided into, for each thread
  count.

Then count what the change does to reading. Every later chapter
measures this way.

## A measuring helper

Put this in your model's package (it is the library of the tutorial's
examples). In a small driver, describe each way of writing as a `Case`
and pass them all to one call of `compare`:

```rust
//! The measuring helper of the tutorial's first chapter. Every example
//! uses it: copy it into your own project as it is.
use orrery_lang as orr;
use std::error::Error;
use std::path::PathBuf;
use std::time::Instant;

pub type Res<T> = Result<T, Box<dyn Error>>;

/// Writes a parameter file `params/<name>.toml` and returns its path.
pub fn params(name: &str, text: &str) -> Res<PathBuf> {
    std::fs::create_dir_all("params")?;
    let path = PathBuf::from(format!("params/{name}.toml"));
    std::fs::write(&path, text)?;
    Ok(path)
}

/// One run: seconds to build and to run, execution units, and a checksum
/// of the result.
#[derive(Clone, Copy)]
pub struct Run {
    pub build: f64,
    pub run: f64,
    pub units: usize,
    pub check: u64,
}

/// Builds a simulation with `config` and runs it, timing the two apart.
pub fn run_once<M: orr::Module>(
    config: orr::SimConfig,
    build: &dyn Fn(orr::SimConfig) -> Res<orr::Sim<M>>,
    run: &dyn Fn(&mut orr::Sim<M>) -> Res<u64>,
) -> Res<Run> {
    let start = Instant::now();
    let mut sim = build(config)?;
    let built = start.elapsed().as_secs_f64();
    let units = sim.load_report().units;
    let start = Instant::now();
    let check = run(&mut sim)?;
    let ran = start.elapsed().as_secs_f64();
    Ok(Run { build: built, run: ran, units, check })
}

fn median(mut v: Vec<f64>) -> f64 {
    v.sort_by(f64::total_cmp);
    v[v.len() / 2]
}

/// One way of writing a model, for `compare`: its name, its configuration
/// (the thread count is set by `compare`), how to build it and how to run
/// it, returning a checksum of the result.
pub struct Case<'a> {
    name: String,
    once: Box<dyn Fn(u32) -> Res<Run> + 'a>,
}

impl<'a> Case<'a> {
    pub fn new<M: orr::Module>(
        name: impl Into<String>,
        config: orr::SimConfig,
        build: impl Fn(orr::SimConfig) -> Res<orr::Sim<M>> + 'a,
        run: impl Fn(&mut orr::Sim<M>) -> Res<u64> + 'a,
    ) -> Self {
        let once = move |threads| {
            let config = orr::SimConfig { workers_per_group: threads, ..config.clone() };
            run_once(config, &build, &run)
        };
        Case { name: name.into(), once: Box::new(once) }
    }
}

/// Runs every case on each of `threads`, `reps` times. In each round, on
/// each thread count, every case runs once: the cases and the thread counts
/// take turns, so a slow spell on the machine is shared out among them.
/// Prints, for each case and thread count, the median run and build time,
/// the speedup over the first thread count and the execution units. Fails
/// if a case's checksum changes from run to run: more threads must not
/// change the result. Returns each case's median run times and checksum.
pub fn compare(threads: &[u32], reps: usize, cases: &[Case]) -> Res<Vec<(Vec<f64>, u64)>> {
    let mut runs: Vec<Vec<Vec<Run>>> = vec![vec![Vec::new(); threads.len()]; cases.len()];
    for _ in 0..reps {
        for (t, &n) in threads.iter().enumerate() {
            for (c, case) in cases.iter().enumerate() {
                runs[c][t].push((case.once)(n)?);
            }
        }
    }
    let mut results = Vec::new();
    for (case, runs) in cases.iter().zip(&runs) {
        let check = runs[0][0].check;
        if let Some(r) = runs.iter().flatten().find(|r| r.check != check) {
            return Err(format!("{}: checksum {:016x}, not {check:016x}", case.name, r.check).into());
        }
        let medians: Vec<f64> = runs.iter().map(|r| median(r.iter().map(|x| x.run).collect())).collect();
        for (t, &n) in threads.iter().enumerate() {
            let build = median(runs[t].iter().map(|x| x.build).collect());
            println!(
                "{}: {n} thread(s): run {:.4} s, {:.2}x of {} thread(s), build {build:.4} s, {} execution unit(s), checksum {check:016x}",
                case.name,
                medians[t],
                medians[0] / medians[t],
                threads[0],
                runs[t][0].units,
            );
        }
        results.push((medians, check));
    }
    Ok(results)
}

/// A mixer: `rounds` rounds of MurmurHash3's finalizer, the stand-in for
/// a unit's work in a cycle.
pub fn mix(mut x: u64, rounds: u64) -> u64 {
    for _ in 0..rounds {
        x ^= x >> 33;
        x = x.wrapping_mul(0xff51_afd7_ed55_8ccd);
        x ^= x >> 29;
    }
    x
}
```

What it does, and why:

- **It times building and running apart.** Building is a fixed cost that
  more threads do not shrink. On 8 threads it is usually longer than on 1.
  Time it separately, or a short run on several threads looks worse than
  it is. Compare whole processes only when the build is a small part.
- **It runs every case on every thread count three times and takes the
  median.** In each round the cases and the thread counts take turns, so
  a slow spell on the machine is shared out among them. Differences
  within 10% are noise.
- **It compares ways of writing within one run.** Do not compare numbers
  from separate runs. Another process, at another time, perhaps on other
  cores, can be faster or slower by more than the difference you are
  looking for. In a model of 64 and 256 independent units, two separate
  runs made one way look a quarter to a third slower on 8 threads. Run in
  turns in one process, the two took the same time.
- **It prints the execution units** (`sim.load_report().units`). A run
  on *N* threads has one thread handing out work and *N* − 1 running
  tasks, so it runs at most *N* − 1 units at once. When there are fewer
  units than workers, building the simulation also warns:
  `1 execution unit for 7 workers: the same as serial`.
- **It checks a checksum of the result.** The `run` closure returns a
  number that sums up the model's final state. More threads must not
  change the result, and neither should a better way of writing the
  same thing. The checksum is timed with the run, so keep it cheap: fold
  a few numbers of the final state, rather than build the whole output
  as text.

Pick a stretch of input that runs for at least about 0.1 s on 1 thread.
If the input is given and shorter, run each case more times (`compare`'s
`reps`), and still read small differences as noise.
When the run ends on a condition the whole model holds (every packet has
arrived), the driver can call `run_until` in slices of some thousands of
cycles and check the condition between them; a task that returns
`orr::Stop` would have to see the whole condition itself (guide:
[Stopping](../guide/running#stopping)).
Time on a machine with at least 8 free cores: a run on 8 threads takes 8
of them, and other work on those cores slows it unevenly.

## An example: a module per lane, or one loop

Sixty-four lanes each mix their own state every cycle. Written as one
module instance per lane:

```rust
/// A lane: `work` mixer rounds on its own state every cycle.
#[orr::module]
pub struct Lane {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub id: u64,
    #[orr::constant]
    pub work: u64,
    pub state: u64,
}

#[orr::module_impl]
impl Lane {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, id: &'constant u64, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
        *state = mix(*state ^ *id, *work);
        Ok(())
    }
}

/// `n` lanes, one instance each.
#[orr::module]
pub struct Lanes {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("mixer rounds a lane does in a cycle", default = 10u64)]
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
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, lane) in module.lanes.iter().enumerate() {
            ctx.connect(module.clock, lane.clock)?;
            ctx.set_const(lane.id, i as u64)?;
            ctx.set_const(lane.work, *module.work)?;
        }
        Ok(())
    }
}
```

`id` and `work` are *constants*: fields marked `#[orr::constant]`. The
parent sets each one once, in its `init`, with `ctx.set_const`, so the
field must be `pub`. A task reads it as `&'constant u64`. Unlike a
parameter, each instance can get its own value; unlike state, reading it
puts no task after another. The API documentation of `ConstHandle` has
the rules.

Written as one module whose one task loops over the lanes:

```rust
/// The same `n` lanes in one module, one task looping over them.
#[orr::module]
pub struct Loop {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("mixer rounds a lane does in a cycle", default = 10u64)]
    work: u64,
    #[orr::trigger]
    clock: orr::Clock,
    pub states: [u64; MAX_LANES],
}

#[orr::module_impl]
impl Loop {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        assert!(*module.n <= MAX_LANES);
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn cycle(ctx: &mut orr::TaskCtx, states: &mut [u64]) -> Result<(), orr::SimError> {
        let (n, work) = (*ctx.module().n, *ctx.module().work);
        for (i, state) in states[..n].iter_mut().enumerate() {
            *state = mix(*state ^ i as u64, work);
        }
        Ok(())
    }
}
```

The driver compares the two, with light work (10 rounds, about 10 ns a
lane and cycle) and heavy work (1000 rounds, about 2 µs):

```rust
fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(10u64, 100_000u64), (1000, 2_000)] {
        let file = params(&format!("ch1-work{work}"), &format!("[top]\nn = 64\nwork = {work}\n"))?;
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        let config = orr::SimConfig::default();
        let lanes = Case::new(
            format!("lanes, work {work}"),
            config.clone(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                let mut sum = 0;
                for lane in sim.top().lanes.iter() {
                    sum = mix(sum ^ *sim.state(lane.state)?, 1);
                }
                Ok(sum)
            },
        );
        let one_loop = Case::new(
            format!("loop, work {work}"),
            config,
            |config| Ok(orr::Sim::new(orr::SimPlan::<Loop>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                let mut sum = 0;
                for &state in sim.top().states.iter().take(64) {
                    sum = mix(sum ^ *sim.state(state)?, 1);
                }
                Ok(sum)
            },
        );
        // The two ways take turns, so they meet the same machine.
        let results = compare(&threads, 3, &[lanes, one_loop])?;
        assert_eq!(results[0].1, results[1].1, "the two ways computed different states");
    }
    Ok(())
}
```

Seconds to run (rough), and the execution units on 1, 2, 4 and 8
threads:

| | 1 | 2 | 4 | 8 | 8 threads vs 1 | units |
|---|---|---|---|---|---|---|
| a module per lane, light work | 0.067 | 0.078 | 0.054 | 0.041 | 1.62× | 1, 1, 3, 7 |
| one loop, light work | 0.059 | 0.073 | 0.102 | 0.108 | 0.55× | 1, 1, 1, 1 |
| a module per lane, heavy work | 0.242 | 0.243 | 0.086 | 0.041 | 5.98× | 1, 1, 3, 7 |
| one loop, heavy work | 0.242 | 0.242 | 0.243 | 0.243 | 0.99× | 1, 1, 1, 1 |

What the table shows:

- **On 1 thread the loop is as fast or a little faster.** A task call
  costs little, but not nothing. With heavy work the two are the same.
  With light work, 64 calls a cycle take 13% longer than one call with a
  loop. Had you timed only 1 thread, you would take the loop.
- **On 8 threads they are far apart.** The module per lane is 1.6 times
  as fast as on 1 thread with light work, and 6.0 times with heavy work.
  The loop is one execution unit: more threads cannot share it out, and
  with light work 8 threads only add the cost of handing out work. They
  take 1.8 times as long as 1 thread.
- **The units say it before you time anything.** The loop has one unit
  on every thread count, and building it on 4 and 8 threads warns.
- **2 threads are not faster than 1.** One of the two hands out work,
  and only the other runs tasks. More threads help from 4 on.

By the rule of the [introduction](./#when-the-three-disagree):

- **With heavy work, a module per lane is better.** No thread count is
  slower, and 4 and 8 threads are faster. On 8 threads it is faster.
- **With light work it is mixed.** 1 thread is 13% slower; 4 and 8
  threads are 1.9 and 2.6 times as fast. Choose by the thread count you
  will run: on 4 or 8 threads, a module per lane.

A module per lane also reads like the hardware, one module for one unit.

An earlier driver ran the two ways one after the other, and on 1 thread
they came out the same. Taking turns, in four runs, the loop was faster
every time.

## Counting readability

Count three things on the code you change, before and after:

1. **Lines.** Count the module and its tasks, not the driver.
2. **What a reader must keep in mind to follow one unit.** Count:
   - the unit's tasks;
   - its gates, and for each the condition it closes on;
   - who wakes it, and from where;
   - what orders its tasks against other tasks in a cycle (access
     modes, delta levels).
   
   Every one of these is something the reader must find and remember.
3. **Whether one hardware unit's behaviour is in one place.** Is it in
   one task of one module, or spread over several tasks or files?

The lane above has one task, no gate, no wake and no order to keep. Its
behaviour is one line, in one place.

## Finding where the time goes

When a model is slower than you expect:

- **Count the calls.** Give a task a counter in its state, add one per
  call, and another for the calls that did work. Read both after the
  run with `sim.state`. The chapters on gates ([2](./gates), [3](./due))
  are about the calls that do nothing. With `Reports::All` (below), the
  report on each stop lists, for each gate, the edges it skipped
  (`gated`): a task that only its clock runs was called on the other
  edges, with no counter.
- **Work out the time a cycle.** Divide 1 thread's run time by the
  cycles. Then compare it with what the threads cost
  ([chapter 10](./threads)).
- **Time a plain program.** Write the same hardware as a plain Rust
  loop, a cycle at a time, without Orrery. It also checks your model's
  results. Its time a cycle on 1 thread is roughly what the work itself
  costs; the rest of your model's time is what the calls and the gates
  cost.
- **Look at the reports.** `SimConfig { reports: orr::Reports::All, .. }`
  prints the load report and a report on each stop.
- **Look for a task on the slow path.** In the load report,
  `counts.fits.generic` lists the tasks that run on a slower, general
  path. `CapDiffers` there means the task's instances meet FIFOs or
  queues of different capacities: one FIFO deeper than the others is
  enough. Give them all the same capacity. In the exam network, one
  stage's FIFOs at depth 4 instead of 2 made 1 thread 1.2 to 3.4 times
  slower, with the same results and no warning.

## Checklist

- Did I keep the old way, and measure old and new in one run, taking
  turns?
- Did I time 1, 2, 4 and 8 threads, each three times?
- Did I time building apart from running?
- Did I work out the speedup, and write down the execution units for
  each thread count?
- Is the result the same on every thread count (a checksum)?
- Did I count the three readability numbers, before and after?
- Did I write down, for the change, "better" or "mixed", and what it
  does on 8 threads?
