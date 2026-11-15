# 8. Arbitration across modules

A crossbar, a bus, a scheduler: many requesters, a few resources, and
each cycle each resource goes to one requester. An arbiter can decide
only once all of the cycle's requests are in. So it reads many units'
state, and how you split it into tasks decides both its speed and its
speedup.

**The rule.** Give one arbitration domain (a subnet of a crossbar, one
scheduler) **one task**. In that task:

1. Go over the requesters **once**. Note, for each resource, who asks for
   it.
2. Pick a winner **only for the resources someone asked for**, each by
   its own policy (round robin from a pointer, oldest first, ...).
3. Let the winners go: pop their heads, push to the resources.

Do not write:

- **a task per resource and a task per requester.** The resource's task
  picks the winner. The requester's task then has to read every
  resource's choice, to see whether it won. That is a `winner` passed
  between tasks, an order to keep within the cycle, and many more
  execution units.
- **one task that goes resource by resource, scanning every requester
  each time.** That costs resources × requesters a cycle. On several
  threads that one task is what everything else waits for.

**How to check.** Write the plain version too, resource by resource. The
two must deliver the same values at the same cycles. Keep the plain
version as a test.

## Example: an 80 × 64 crossbar, three ways

Eighty sources each send a value now and then to one of 64 sinks,
through a FIFO into the crossbar. Each output port grants, round robin,
the first input after its pointer whose head goes to it. The granted
input pops its head, and the port pushes it to its sink. Sources and
sinks can also do some work of their own each cycle (`work`), standing
for the rest of a model.

The sources and the sinks:

```rust
const INPUTS: usize = 80;
const PORTS: usize = 64;
const NONE: u8 = u8::MAX;

/// Whether source `s` sends at its `k`-th edge, and where to.
fn send(s: usize, k: u64) -> Option<u32> {
    let h = (k.wrapping_mul(0x9E37_79B9_7F4A_7C15) ^ (s as u64).wrapping_mul(0xC2B2_AE3D_27D4_EB4F)) >> 32;
    (h % 4 == 0).then_some(((h >> 8) % PORTS as u64) as u32)
}

fn fold(sum: u64, value: u32, cycle: u64) -> u64 {
    sum.wrapping_mul(1_000_003).wrapping_add(u64::from(value) << 24 | cycle)
}

/// `rounds` rounds of a xorshift-multiply mix, each depending on the last:
/// work a component does that has nothing to do with the crossbar.
fn busy(state: u64, rounds: u32) -> u64 {
    let mut x = state | 1;
    for _ in 0..std::hint::black_box(rounds) {
        x ^= x >> 29;
        x = x.wrapping_mul(0xBF58_476D_1CE4_E5B9);
    }
    x
}

#[orr::module]
pub struct Source {
    #[orr::param("busy rounds a cycle", default = 0u32)]
    work: u32,
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub id: u8,
    #[orr::interface]
    pub out: orr::FifoTx<u32>,
    k: u64,
    noise: u64,
}

#[orr::module_impl]
impl Source {
    #[orr::task(triggers = (clock,))]
    fn push(
        ctx: &mut orr::TaskCtx,
        id: &'constant u8,
        out: &'latest mut orr::FifoTx<u32>,
        k: &mut u64,
        noise: &mut u64,
    ) -> Result<(), orr::SimError> {
        *noise = busy(*noise, *ctx.module().work);
        *k += 1;
        if let Some(port) = send(usize::from(*id), *k) {
            // The payload: the port, and which source and when.
            let _ = out.push(port | u32::from(*id) << 8 | ((*k as u32) & 0xffff) << 16);
        }
        Ok(())
    }
}

#[orr::module]
pub struct Sink {
    #[orr::param("busy rounds a cycle", default = 0u32)]
    work: u32,
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub inbox: orr::FifoRx<u32>,
    pub sum: u64,
    noise: u64,
}

#[orr::module_impl]
impl Sink {
    #[orr::task(triggers = (clock,))]
    fn pop(ctx: &mut orr::TaskCtx, inbox: &mut orr::FifoRx<u32>, sum: &mut u64, noise: &mut u64) -> Result<(), orr::SimError> {
        *noise = busy(*noise, *ctx.module().work);
        let clock = ctx.triggered_triggers().0.expect("runs on its clock");
        if let Some(value) = inbox.pop() {
            *sum = fold(*sum, value, clock.cycle_of(ctx.now()));
        }
        Ok(())
    }
}
```

A sink folds in the cycle with `clock.cycle_of(ctx.now())`, which counts
the clock's first edge as 1. For cycles counted from 0, as a spec
usually counts them, subtract one.

**A module per port and per input.** Each port picks its winner and
writes it. Each input reads every port's winner (`&'latest`), and pops
its head if it won. A third module holds them and shares each port's
winner with every input:

```rust
/// The input after `next` whose head goes to `port`, round robin.
fn grant(port: usize, next: &mut u8, heads: impl Fn(usize) -> Option<u32>) -> u8 {
    let first = (usize::from(*next) + 1) % INPUTS;
    let won = (first..INPUTS).chain(0..first).find(|&i| heads(i).is_some_and(|v| (v & 0xff) as usize == port));
    match won {
        Some(i) => {
            *next = i as u8;
            i as u8
        }
        None => NONE,
    }
}

// --- split: a module per port and per input ---

#[orr::module]
pub struct Port {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub port: u8,
    #[orr::reset(INPUTS as u8 - 1)]
    next: u8,
    pub winner: u8,
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub out: orr::FifoTx<u32>,
}

#[orr::module_impl]
impl Port {
    #[orr::task(triggers = (clock,))]
    fn grant(
        _ctx: &mut orr::TaskCtx,
        port: &'constant u8,
        next: &mut u8,
        winner: &mut u8,
        ins: &orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        out: &'latest mut orr::FifoTx<u32>,
    ) -> Result<(), orr::SimError> {
        *winner = NONE;
        if !out.is_full() {
            *winner = grant(usize::from(*port), next, |i| ins.at(i).front().copied());
            if *winner != NONE {
                let value = *ins.at(usize::from(*winner)).front().expect("won with its head");
                out.push(value).expect("checked for room");
            }
        }
        Ok(())
    }
}

#[orr::module]
pub struct Input {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub input: u8,
    pub winners: [orr::StateRef<u8>; 64],
    #[orr::interface]
    pub rx: orr::FifoRx<u32>,
}

#[orr::module_impl]
impl Input {
    #[orr::task(triggers = (clock,))]
    fn pop(_ctx: &mut orr::TaskCtx, input: &'constant u8, winners: &'latest [u8], rx: &mut orr::FifoRx<u32>) -> Result<(), orr::SimError> {
        if let Some(&head) = rx.front() {
            if winners[(head & 0xff) as usize] == *input {
                rx.pop();
            }
        }
        Ok(())
    }
}

#[orr::module]
pub struct Split {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::submodule]
    ports: [Port; 64],
    #[orr::submodule]
    inputs: [Input; 80],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl Split {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        for (p, port) in module.ports.iter().enumerate() {
            ctx.connect(module.clock, port.clock)?;
            ctx.set_const(port.port, p as u8)?;
            for i in 0..INPUTS {
                ctx.share_interface(module.ins[i], port.ins[i])?;
            }
            ctx.share_interface(module.outs[p], port.out)?;
        }
        for (i, input) in module.inputs.iter().enumerate() {
            ctx.connect(module.clock, input.clock)?;
            ctx.set_const(input.input, i as u8)?;
            for p in 0..PORTS {
                ctx.share_state(module.ports[p].winner, input.winners[p])?;
            }
            ctx.share_interface(module.ins[i], input.rx)?;
        }
        Ok(())
    }
}
```

**One task, port by port.** For each port, it scans the inputs:

```rust
// --- one: one task for the crossbar ---

#[orr::module]
pub struct One {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::reset(INPUTS as u8 - 1)]
    next: [u8; 64],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl One {
    #[orr::task(triggers = (clock,))]
    fn cycle(
        _ctx: &mut orr::TaskCtx,
        next: &mut [u8],
        ins: &mut orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        outs: &'latest mut orr::ArrayView<[orr::FifoTx<u32>; 64]>,
    ) -> Result<(), orr::SimError> {
        let mut winners = [NONE; PORTS];
        for (port, winner) in winners.iter_mut().enumerate() {
            if !outs.at(port).is_full() {
                *winner = grant(port, &mut next[port], |i| ins.at(i).front().copied());
                if *winner != NONE {
                    let value = *ins.at(usize::from(*winner)).front().expect("won with its head");
                    outs.at_mut(port).push(value).expect("checked for room");
                }
            }
        }
        for i in 0..INPUTS {
            let head = ins.at(i).front().copied();
            if let Some(head) = head {
                if winners[(head & 0xff) as usize] == i as u8 {
                    ins.at_mut(i).pop();
                }
            }
        }
        Ok(())
    }
}
```

**One task, one pass over the inputs.** First it notes, for each port,
the inputs that ask for it, as a bit set. Then it picks a winner only for
the ports someone asked for, and pops the winners:

```rust
// --- fast: one task, which finds each port's requests once ---

#[orr::module]
pub struct Fast {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::reset(INPUTS as u8 - 1)]
    next: [u8; 64],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl Fast {
    #[orr::task(triggers = (clock,))]
    fn cycle(
        _ctx: &mut orr::TaskCtx,
        next: &mut [u8],
        ins: &mut orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        outs: &'latest mut orr::ArrayView<[orr::FifoTx<u32>; 64]>,
    ) -> Result<(), orr::SimError> {
        // Which inputs' heads go to each port, as a bit set over inputs.
        let mut requests = [0u128; PORTS];
        for i in 0..INPUTS {
            if let Some(&head) = ins.at(i).front() {
                requests[(head & 0xff) as usize] |= 1 << i;
            }
        }
        let mut granted = 0u128;
        for (port, &asking) in requests.iter().enumerate() {
            if asking == 0 || outs.at(port).is_full() {
                continue;
            }
            // The first asking input after the pointer, round robin.
            let first = (usize::from(next[port]) + 1) % INPUTS;
            let after = asking >> first << first;
            let won = if after != 0 { after.trailing_zeros() } else { asking.trailing_zeros() } as usize;
            next[port] = won as u8;
            granted |= 1 << won;
            let value = *ins.at(won).front().expect("won with its head");
            outs.at_mut(port).push(value).expect("checked for room");
        }
        while granted != 0 {
            let i = granted.trailing_zeros() as usize;
            granted &= granted - 1;
            ins.at_mut(i).pop();
        }
        Ok(())
    }
}
```

The tops and the driver, which checks that all three delivered the same
values at the same cycles:

```rust
macro_rules! top {
    ($name:ident, $xbar:ident) => {
        #[orr::module]
        pub struct $name {
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            sources: [Source; 80],
            #[orr::submodule]
            sinks: [Sink; 64],
            #[orr::submodule]
            ins: [orr::Fifo<u32>; 80],
            #[orr::submodule]
            outs: [orr::Fifo<u32>; 64],
            #[orr::submodule]
            xbar: $xbar,
        }

        #[orr::module_impl]
        impl $name {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))?;
                ctx.connect(module.clock, module.xbar.clock)?;
                for (s, source) in module.sources.iter().enumerate() {
                    ctx.connect(module.clock, source.clock)?;
                    ctx.set_const(source.id, s as u8)?;
                    ctx.connect_interface(&module.ins[s], source.out)?;
                    ctx.connect_interface(&module.ins[s], module.xbar.ins[s])?;
                }
                for (p, sink) in module.sinks.iter().enumerate() {
                    ctx.connect(module.clock, sink.clock)?;
                    ctx.connect_interface(&module.outs[p], sink.inbox)?;
                    ctx.connect_interface(&module.outs[p], module.xbar.outs[p])?;
                }
                Ok(())
            }
        }
    };
}

top!(SplitTop, Split);
top!(OneTop, One);
top!(FastTop, Fast);

/// A checksum of what the sinks saw.
fn checksum<M: orr::Module>(sim: &orr::Sim<M>, sums: Vec<orr::StateHandle<u64>>) -> Res<u64> {
    let mut check = 0u64;
    for sum in sums {
        check = check.wrapping_mul(31).wrapping_add(*sim.state(sum)?);
    }
    Ok(check)
}

macro_rules! variant {
    ($top:ty, $name:expr, $file:expr, $end:expr) => {
        Case::new(
            $name,
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<$top>::new(Some($file.clone()), None)?, config)?),
            |sim| {
                sim.run_until($end)?;
                let sums = sim.top().sinks.iter().map(|s| s.sum).collect();
                checksum(sim, sums)
            },
        )
    };
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(0u32, 50_000u64), (300, 5_000)] {
        let file = params(&format!("ch8-work{work}"), &format!("[[top.sources]]\nwork = {work}\n\n[[top.sinks]]\nwork = {work}\n"))?;
        let end = orr::Time::ns(cycles - 1);
        let results = compare(
            &threads,
            3,
            &[
                variant!(SplitTop, format!("a module per port and per input, work {work}"), file, end),
                variant!(OneTop, format!("one task, port by port, work {work}"), file, end),
                variant!(FastTop, format!("one task, one pass over the inputs, work {work}"), file, end),
            ],
        )?;
        let (split, one, fast) = (results[0].1, results[1].1, results[2].1);
        assert!(split == one && one == fast, "the three crossbars delivered differently");
    }
    Ok(())
}
```

## What it does to the three measures

Seconds to run (rough), with no work around the crossbar (50 000 cycles)
and with about 0.4 µs of work per source and sink a cycle (5 000
cycles):

| | 1 | 2 | 4 | 8 | 8 threads vs 1 | units |
|---|---|---|---|---|---|---|
| a module per port and per input, no work | 0.429 | 0.459 | 0.243 | 0.173 | 2.48× | 4, 4, 12, 28 |
| one task, port by port, no work | 0.436 | 0.458 | 0.466 | 0.462 | 0.94× | 3, 3, 7, 15 |
| one task, one pass, no work | 0.057 | 0.079 | 0.090 | 0.087 | 0.66× | 3, 3, 7, 15 |
| a module per port and per input, work | 0.320 | 0.324 | 0.119 | 0.059 | 5.47× | 4, 4, 12, 28 |
| one task, port by port, work | 0.321 | 0.323 | 0.140 | 0.088 | 3.66× | 3, 3, 7, 15 |
| one task, one pass, work | 0.283 | 0.285 | 0.102 | 0.051 | 5.59× | 3, 3, 7, 15 |

- **One task, one pass is better than a module per port and per input.**
  - Every thread count is faster.
  - With no work around it: 7.5 times as fast on 1 thread, 2.0 times on
    8.
  - With work around it: 1.13 to 1.16 times as fast.
  - On 8 threads it is faster.
- **Its speedup is lower with nothing around it** (0.66), and it does not
  matter. It has made the crossbar so cheap that, with nothing else in
  the model, there is nothing left to share among threads. With work
  around it, its speedup is the highest of the three (5.6). By the rule
  of the [introduction](./#when-the-three-disagree), compare the times,
  not the speedups.
- **One task, port by port, is mixed.**
  - On 1 thread it is as fast as a module per port.
  - On 8 threads it is 2.7 times as slow with no work, and 1.5 times
    with work.
  - It does 64 × 80 checks a cycle in one task, which no thread can
    share.
- **Reading.**
  - The one-pass arbiter is one function you can read top to bottom,
    with no `winner` passed between tasks and no order to keep.
  - The module per port and per input needs two kinds of module.
  - It also needs the winners shared between them, and an order within
    the cycle: inputs read the winners `&'latest`.

## In the GPU model

The GPU model that comes with Orrery has a crossbar of two subnets, one
for requests and one for replies. It once had a module per port and a
module per input, with winners passed between them, as above. Now each
subnet is one task that goes over its inputs once.

Timed by the Orrery developers on the eleven test programs, against the
module-per-port version (with nothing else changed):

- 1 thread: 0.923 of the time in all, every program faster;
- 8 threads: 0.752 of the time in all, every program faster;
- execution units: from 106 to 80.

The crossbar's code went from about 320 lines to about 240, and from
four kinds of module to two. The 144 `winner` fields are gone.

## When not to

- **A domain so large that one pass is itself much of the cycle's work.**
  Split it where the hardware splits it: one task per subnet, or per
  scheduler. Do not split it per resource.
- **Several domains that never share a requester** are separate
  arbiters: give each its own task, so they can run side by side.

## Checklist

- Does the arbiter see all of this cycle's requests (the requesters act
  before it, or it reads them as the cycle began)?
- Is the pointer moved the way the hardware moves it? (Compare with the
  plain version, cycle by cycle.)
- Is each requester looked at once a cycle?
- Is each arbitration domain one task? Count its kinds of task in the
  load report.
