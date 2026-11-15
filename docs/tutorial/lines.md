# 5. Lines and delays

Units pass things to each other over lines: events, FIFOs, shared state.
How you write a line decides two things. It decides the order in which
the two ends run within a cycle. And it decides whether they can run at
the same time on different threads.

**The rules.**

1. **A line that has a delay in the hardware gets that delay in the
   model, declared.** A wire, a link, or a pipeline register between two
   stages takes at least a cycle. Write it as an event with
   `#[orr::trigger(min_delay = ..)]`, or as a FIFO of `orr::Due` values
   ([chapter 3](./due)).
   - Then the two ends never run in the same cycle as cause and effect,
     so they can run side by side.
   - Without a delay, the receiving end runs after the sending end, in
     the same cycle, at a later delta level. A chain of such lines is a
     chain of delta levels, each waiting for the one before.
2. **Within a cycle, say the order with access modes, not with delta
   levels.**
   - A task that reads state with `&` sees it as the cycle began, before
     any task wrote it. `&'latest` sees it after the writers of this
     cycle.
   - The mode is on the parameter, where the reader sees it. A delta
     level (`clock + delta(k)`) is a number that the whole model must
     agree on. Use a level only where a mode cannot say it, as in the
     third pairing of [chapter 4](./wakes).
3. **Two units that send to each other need two tasks each, or a delay.**
   With one FIFO each way and one task per unit, the order is a cycle,
   an error when the simulation is built. Give each unit a task that
   takes and one that sends. Or, if the link has a delay in the
   hardware, use events with `min_delay`: they have no order within a
   cycle, and one task per unit is enough
   ([Two units that send to each other](../guide/components#who-acts-first)).

The guide covers each line in detail:
[Declared delays](../guide/time#declared-delays),
[Delta cycles](../guide/time#delta-cycles),
[Who acts first](../guide/components#who-acts-first).

## Example: lanes of stages, with and without a delay

Eight lanes of eight stages. The head of each lane runs on the clock.
Each later stage runs when the stage before it passes on its state,
mixes it into its own, and passes that on. The same code is built twice,
from a macro:

- once with each line delayed one cycle and the delay declared
  (`min_delay`);
- once with no delay (`max_delay = DELTA`, and each event sent a delta
  cycle on).

```rust
macro_rules! lanes {
    ($link:expr; $($bounds:tt)*) => {
        #[orr::module]
        pub struct Head {
            #[orr::trigger]
            pub clock: orr::ClockRelay,
            #[orr::trigger($($bounds)*)]
            pub out: orr::EventSource<u64>,
            #[orr::constant]
            pub work: u64,
            pub state: u64,
        }

        #[orr::module_impl]
        impl Head {
            #[orr::task(triggers = (clock,), emits = (out,))]
            fn cycle(ctx: &mut orr::TaskCtx, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
                *state = $crate::mix(*state ^ 1, *work);
                ctx.emit_triggers().0.emit(*state, $link)
            }
        }

        #[orr::module]
        pub struct Stage {
            #[orr::trigger]
            pub input: orr::EventRelay<u64>,
            #[orr::trigger($($bounds)*)]
            pub out: orr::EventSource<u64>,
            #[orr::constant]
            pub work: u64,
            pub state: u64,
        }

        #[orr::module_impl]
        impl Stage {
            #[orr::task(triggers = (input,), emits = (out,))]
            fn cycle(ctx: &mut orr::TaskCtx, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
                let input = ctx.triggered_triggers().0.copied().unwrap_or(0);
                *state = $crate::mix(*state ^ input, *work);
                ctx.emit_triggers().0.emit(*state, $link)
            }
        }

        /// `lanes` lanes of a head and `stages - 1` later stages.
        #[orr::module]
        pub struct Lanes {
            #[orr::param("lanes", default = 8usize)]
            lanes: usize,
            #[orr::param("later stages, lanes x (stages a lane - 1)", default = 56usize)]
            rest: usize,
            #[orr::param("mixer rounds a stage does in a cycle", default = 100u64)]
            work: u64,
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            heads: [Head; orr::DynShape(lanes)],
            #[orr::submodule]
            stages: [Stage; orr::DynShape(rest)],
        }

        #[orr::module_impl]
        impl Lanes {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                let per_lane = *module.rest / *module.lanes;
                ctx.init_clock(module.clock, orr::Time::ps($crate::PERIOD_PS), orr::Time::ps(0))?;
                for (l, head) in module.heads.iter().enumerate() {
                    ctx.connect(module.clock, head.clock)?;
                    ctx.set_const(head.work, *module.work)?;
                    let mut previous = head.out;
                    for stage in &module.stages[l * per_lane..(l + 1) * per_lane] {
                        ctx.connect(previous, stage.input)?;
                        ctx.set_const(stage.work, *module.work)?;
                        previous = stage.out;
                    }
                }
                Ok(())
            }
        }

        /// A checksum of every stage's state.
        pub fn checksum(sim: &orr::Sim<Lanes>) -> $crate::Res<u64> {
            let mut sum = 0;
            for head in sim.top().heads.iter() {
                sum = $crate::mix(sum ^ *sim.state(head.state)?, 1);
            }
            for stage in sim.top().stages.iter() {
                sum = $crate::mix(sum ^ *sim.state(stage.state)?, 1);
            }
            Ok(sum)
        }
    };
}

/// Lines with a delay of one cycle, declared as their minimum.
pub mod timed {
    use orrery_lang as orr;
    lanes!(orr::Time::ps(super::PERIOD_PS); min_delay = orr::Time::ps(super::PERIOD_PS));
}

/// Lines without a delay: the next stage runs in the same cycle.
pub mod untimed {
    use orrery_lang as orr;
    lanes!(orr::Time::DELTA; max_delay = orr::Time::DELTA);
}
```

The driver measures both, with light work (10 rounds a stage, about
10 ns) and heavier work (100 rounds, about 0.2 µs):

```rust
fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(10u64, 50_000u64), (100, 10_000)] {
        let file = params(&format!("ch5-work{work}"), &format!("[top]\nwork = {work}\n"))?;
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        // Two different models: their checksums differ, and that is right.
        let timed = Case::new(
            format!("8 lanes x 8 stages, a delay of one cycle, work {work}"),
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<timed::Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                timed::checksum(sim)
            },
        );
        let untimed = Case::new(
            format!("8 lanes x 8 stages, no delay, work {work}"),
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<untimed::Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                untimed::checksum(sim)
            },
        );
        compare(&threads, 3, &[timed, untimed])?;
    }
    Ok(())
}
```

The two are different models: with a delay, a value takes a cycle to go
from one stage to the next. Their checksums differ, and that is right.
What they share is the work: 64 stages, each mixing once a cycle.

## What it does to the three measures

Seconds to run (rough), and the execution units on 1, 2, 4 and 8
threads:

| | 1 | 2 | 4 | 8 | 8 threads vs 1 | units |
|---|---|---|---|---|---|---|
| a delay of one cycle, light work | 0.186 | 0.202 | 0.110 | 0.078 | 2.37× | 2, 2, 6, 14 |
| no delay, light work | 0.227 | 0.239 | 0.241 | 0.233 | 0.97× | 2, 2, 24, 56 |
| a delay of one cycle, heavier work | 0.147 | 0.149 | 0.059 | 0.033 | 4.47× | 2, 2, 6, 14 |
| no delay, heavier work | 0.155 | 0.158 | 0.084 | 0.060 | 2.58× | 2, 2, 24, 56 |

- **With the delay declared, the stages run side by side.**
  - On 8 threads, light work: 3.0 times as fast as without the delay.
  - Heavier work: 1.8 times as fast.
  - Without a delay, light work gets no faster with more threads: 8
    threads take as long as 1.
- **The execution units show why.**
  - Without a delay, a stage must run after the stage that feeds it, in
    the same cycle. So each level runs as a step of its own: the heads,
    then the second stages, and so on. Each of the eight levels is split
    over the 7 workers: 8 × 7 = 56 units on 8 threads.
  - With the delay there are two kinds of task, heads and stages:
    2 × 7 = 14 units.
  - Every unit costs something in every cycle
    ([chapter 10](./threads)).
- **On 1 thread**, with light work, the delayed one is a fifth faster
  too. With heavier work the two are the same.
- **Reading.** A declared delay says on the line what the hardware does:
  "this takes a cycle". A line without one makes the reader work out the
  order of the two ends.

## When not to

- **Do not add a delay the hardware does not have.** A delay changes
  when things happen, and so it changes the results.
- **Do not add one to match a reference model either.** A model that must
  match a reference model cycle by cycle has, within a cycle, whatever
  order the reference has. Often that is "the value written this cycle"
  or "the slot freed this cycle". Adding a cycle there makes it differ
  from the reference.
- So this rule is for writing a model: where the hardware's line has a
  delay, declare it. It is not a way to speed up a model that already
  matches its reference.
- **Declare the true minimum.** Declaring more than the hardware has is
  wrong. Declaring less is allowed, but gives up some of the parallelism.

## Checklist

- Does every line that takes time in the hardware have its delay
  declared (`min_delay`, or a FIFO of `Due` values)?
- Is every order within a cycle said by an access mode (`&`, `&'latest`)?
  Is each delta level there for a reason I can write down?
- Do two units that send to each other have a take task and a send task
  each, or a delayed line?
- Did I compare the execution units before and after? Do the units grow
  with chains of lines that have no delay?
