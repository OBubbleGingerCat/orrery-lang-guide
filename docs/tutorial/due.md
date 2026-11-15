# 3. Close until the next deadline

**The rule.** A unit often waits for something whose time it knows: a
fixed latency, a timer, a timing parameter it must wait out. Do not
count such waits down cycle by cycle.

- Keep each waiting value together with the edge it is due at, as an
  `orr::Due<T>`. Put several waiting values in an `orr::DelayQueue` (in
  the module's state), or in an `orr::Fifo<orr::Due<T>>` (between
  modules).
- When nothing is due, close the gate until the first value is seen:
  `gate.close_until(clock.edge_after(orr::Timepoint::at_fs(first.at)))`.

Nobody has to wake the unit, so there is no wake to get right.

**How to check.** Write the unit once the plain way, counting down every
cycle, and compare. The two must do the same things at the same cycles.
Keep the plain version as a test.

The guide has the details: [Delays](../guide/components#delays) and
[Clock gates](../guide/time#clock-gates).

## Example: memories with a fixed latency

Each of 64 memories answers a request 100 cycles after it arrives.
Requests come every 20 cycles, so about five are waiting at any time.

The plain way: each waiting request keeps the cycles it has left, and
the memory counts them all down every cycle:

```rust
/// Counts every waiting request down, every cycle.
#[orr::module]
pub struct Countdown {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::trigger]
    pub requests: orr::EventRelay<u64>,
    #[orr::constant]
    pub work: u64,
    gate: orr::Gate,
    /// Per slot: cycles left, 0 for a free slot; and the request.
    left: [u64; SLOTS],
    value: [u64; SLOTS],
    pub answered: u64,
}

#[orr::module_impl]
impl Countdown {
    #[orr::task(triggers = (clock, requests))]
    #[allow(clippy::too_many_arguments)]
    fn cycle(
        ctx: &mut orr::TaskCtx,
        work: &'constant u64,
        gate: &mut orr::Gate,
        left: &mut [u64],
        value: &mut [u64],
        answered: &mut u64,
    ) -> Result<(), orr::SimError> {
        let (clock, request) = ctx.triggered_triggers();
        let now = edge(ctx.now());
        if clock.is_some() {
            for slot in 0..SLOTS {
                if left[slot] > 0 {
                    left[slot] -= 1;
                    if left[slot] == 0 {
                        *answered = mix(*answered ^ value[slot] ^ now, *work);
                    }
                }
            }
        }
        if let Some(&request) = request {
            let slot = left.iter().position(|&l| l == 0).ok_or_else(|| orr::SimError::new("no free slot"))?;
            (left[slot], value[slot]) = (LATENCY.get(), request);
        }
        if left.iter().all(|&l| l == 0) {
            gate.close(); // until the next request
        } else {
            gate.open();
        }
        Ok(())
    }
}
```

The way of this chapter: each waiting request keeps the edge it is due
at, and the memory sleeps until the first is due:

```rust
/// Keeps every waiting request with the edge it is due at, and sleeps
/// until the first is due.
#[orr::module]
pub struct Due {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::trigger]
    pub requests: orr::EventRelay<u64>,
    #[orr::constant]
    pub work: u64,
    gate: orr::Gate,
    waiting: orr::DelayQueue<u64, SLOTS>,
    pub answered: u64,
}

#[orr::module_impl]
impl Due {
    #[orr::task(triggers = (clock, requests))]
    fn cycle(
        ctx: &mut orr::TaskCtx,
        work: &'constant u64,
        gate: &mut orr::Gate,
        waiting: &mut orr::DelayQueue<u64, SLOTS>,
        answered: &mut u64,
    ) -> Result<(), orr::SimError> {
        let now = ctx.now();
        while let Some(value) = waiting.pop_due(now) {
            *answered = mix(*answered ^ value ^ edge(now), *work);
        }
        if let Some(&request) = ctx.triggered_triggers().1 {
            let due = CLOCK.nth_edge_from(now, LATENCY);
            waiting.push_due(request, due).map_err(|_| orr::SimError::new("no free slot"))?;
        }
        match waiting.front() {
            // Seen at the first edge after the edge it is due at.
            Some(first) => gate.close_until(CLOCK.edge_after(orr::Timepoint::at_fs(first.at))),
            None => gate.close(), // until the next request
        }
        Ok(())
    }
}
```

The clock and the latency are constants:

```rust
pub const PERIOD_PS: u64 = 1000;
/// The memories' clock, for working out due edges.
pub const CLOCK: orr::ClockInfo = orr::ClockInfo::from_parts(orr::Time::ps(PERIOD_PS), orr::Time::ps(0));
pub const LATENCY: NonZeroU64 = match NonZeroU64::new(100) {
    Some(n) => n,
    None => panic!(),
};
/// The most requests a memory has waiting.
pub const SLOTS: usize = 16;

fn edge(now: orr::Timepoint) -> u64 {
    (now.ps() / u128::from(PERIOD_PS)) as u64
}
```

How the due edge is worked out:

- `CLOCK.nth_edge_from(now, LATENCY)` is the 100th edge counting the one
  at `now` as the first. A request that arrives at edge *E* is therefore
  due at edge *E* + 99.
- A value is seen at a time strictly after its due edge. So
  `pop_due(now)` gives the request at edge *E* + 100, the same edge at
  which the countdown reaches 0.
- The memory works these out from a constant `ClockInfo` rather than
  from its clock trigger. A task run by the request event, with its gate
  closed, has no clock slot.

The tops, and the driver that measures both and checks that they
answered the same requests at the same cycles:

```rust
macro_rules! top {
    ($top:ident, $memory:ident) => {
        #[orr::module]
        pub struct $top {
            #[orr::param("memories", default = 64usize)]
            n: usize,
            #[orr::param("a memory gets a request every this many cycles", default = 20u64)]
            every: u64,
            #[orr::param("mixer rounds to answer a request", default = 100u64)]
            work: u64,
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            memories: [$memory; orr::DynShape(n)],
            #[orr::trigger(min_delay = orr::Time::ps(PERIOD_PS))]
            requests: [orr::EventSource<u64>; orr::DynShape(n)],
        }

        #[orr::module_impl]
        impl $top {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
                for (i, memory) in module.memories.iter().enumerate() {
                    ctx.connect(module.clock, memory.clock)?;
                    ctx.connect(module.requests[i], memory.requests)?;
                    ctx.set_const(memory.work, *module.work)?;
                }
                Ok(())
            }

            /// Sends memory `i` a request every `every` cycles, arriving at
            /// the next edge.
            #[orr::task(triggers = (clock,), emits = (requests,))]
            fn source(ctx: &mut orr::TaskCtx) -> Result<(), orr::SimError> {
                let now = edge(ctx.now());
                let every = *ctx.module().every;
                for (i, mut request) in ctx.emit_triggers().0.iter().enumerate() {
                    if (now + 3 * i as u64) % every == 0 {
                        request.emit(now * 64 + i as u64, orr::Time::ps(PERIOD_PS))?;
                    }
                }
                Ok(())
            }
        }
    };
}

top!(CountdownTop, Countdown);
top!(DueTop, Due);

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    let cycles = 50_000u64;
    let end = orr::Time::ps(PERIOD_PS * cycles - 1);
    let file = params("ch3", "[top]\n")?;
    let countdown = Case::new(
        "countdown",
        orr::SimConfig::default(),
        |config| Ok(orr::Sim::new(orr::SimPlan::<CountdownTop>::new(Some(file.clone()), None)?, config)?),
        |sim| {
            sim.run_until(end)?;
            let mut sum = 0;
            for memory in sim.top().memories.iter() {
                sum = mix(sum ^ *sim.state(memory.answered)?, 1);
            }
            Ok(sum)
        },
    );
    let due = Case::new(
        "due",
        orr::SimConfig::default(),
        |config| Ok(orr::Sim::new(orr::SimPlan::<DueTop>::new(Some(file.clone()), None)?, config)?),
        |sim| {
            sim.run_until(end)?;
            let mut sum = 0;
            for memory in sim.top().memories.iter() {
                sum = mix(sum ^ *sim.state(memory.answered)?, 1);
            }
            Ok(sum)
        },
    );
    let results = compare(&threads, 3, &[countdown, due])?;
    // Both must answer the same requests at the same cycles.
    assert_eq!(results[0].1, results[1].1, "the two memories answered differently");
    Ok(())
}
```

## What it does to the three measures

Seconds to run (rough). The execution units are 2, 2, 4 and 8:

| | 1 | 2 | 4 | 8 | 8 threads vs 1 |
|---|---|---|---|---|---|
| counting down every cycle | 0.139 | 0.158 | 0.084 | 0.057 | 2.43× |
| closing until the first is due | 0.068 | 0.083 | 0.057 | 0.047 | 1.43× |

- **Fast.** Closing until due is **better**: 2.1 times as fast on 1
  thread, 1.9 on 2, 1.5 on 4. On 8 threads it is faster, 1.2 times.
  - The countdown runs every memory in every cycle, about 3.2 million
    calls in all.
  - Closing until due runs a memory twice per request (when it arrives
    and when it is due), about a tenth as many calls.
  - What is left is the answering, the same in both.
- **Speedup** falls, from 2.4 to 1.4: what this saves is mostly 1
  thread's time ([chapter 2](./gates) says why). Compare the times.
- **Reading.** One field (`waiting`) replaces two arrays and a loop. The
  "cycles left" a reader had to follow disappear. There is no wake to
  check, because the unit opens its own gate.

The deadline must be right to the edge. Get it one edge off and the unit
answers a cycle early or late: the results change, which the comparison
with the countdown catches. The common mistake is to count from the
wrong edge. A task that runs at a later delta level (`clock + delta(k)`)
must count from delta 0 of its time,
`orr::Timepoint::at_fs(ctx.now().fs())`, or it counts from the next edge
([Delays](../guide/components#delays)).

## In the GPU model

The GPU model that comes with Orrery keeps every fixed latency as a due
edge:

- the SM's execution pipelines hold `orr::Due` entries;
- an L2 bank's ROP delay queue is an `orr::DelayQueue`;
- a DRAM channel's latency queue holds its requests as `orr::Due`
  values.

An L2 bank's cache step, and each DRAM channel, ends with the same three
cases:

- busy: stay open;
- nothing to do: `gate.close()`, until a wake;
- nothing until a known edge: `gate.close_until(clock.edge_after(..))`.

## When not to

- **The wait is not known in advance**, for example until another unit
  has room. Then close until woken ([chapter 2](./gates)), or stay open.
- **Values with different delays share one queue.** A `DelayQueue` looks
  only at its front, so values come out in the order they went in. Use
  one queue per fixed delay.
- **The unit is busy most cycles anyway.** Keeping due edges is still
  clearer than counting down, but it will not be faster.

## Checklist

- Is there a counter that goes down by one every cycle? Replace it by a
  due edge.
- Is the edge the gate closes until the edge where the front value is
  seen (`edge_after(at_fs(front.at))`)?
- If the task runs at a later delta level, does it count from delta 0 of
  its time?
- Does the plain version, counting down, give the same results at the
  same cycles?
