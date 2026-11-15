# 4. Wake in time

**The rule.** A unit closed until there is work is opened by the unit
that brings the work, with a wake ([chapter 2](./gates)). A wake opens
the gate from the *next* timepoint: it cannot make the woken task run in
the timepoint that sends it. Whether the work is then taken in time
depends on which of the two tasks acts first in a cycle.

For a value passed through a FIFO, find your pair of tasks in this
table:

| Producer | Consumer | A value pushed while the consumer is closed is taken |
|---|---|---|
| `&'latest mut FifoTx` (acts second) | `&mut FifoRx` (acts first) | in the next cycle, as with the gate always open: in time |
| `&mut FifoTx` (acts first) | `&'latest mut FifoRx`, at `clock + delta(1)` | in the same cycle, as with the gate always open: in time |
| `&mut FifoTx` (acts first) | `&'latest mut FifoRx`, at the same delta level | **a cycle late**: with the gate always open it is taken in the same cycle |

In the third pairing, do one of two things:

- move the consumer to a later delta level;
- or keep its gate open while a value can arrive.

**Waiting for room is the other way round.** A producer that holds a
value for a full FIFO waits for its consumer to pop. In the first row
the consumer acts first: a slot it frees in a cycle is filled in that
same cycle by a producer whose gate is open. A wake from the consumer
opens the producer's gate only from the next timepoint: a cycle late.
So keep such a producer's gate open while it waits for room. When the
producer acts first, it sees a freed slot only in the next cycle anyway,
and a wake from the consumer is in time.

The guide explains the order of tasks in a cycle
([Who acts first](../guide/components#who-acts-first)) and the wake
calls ([Wakes](../guide/time#wakes)).

**How to check.** Give the model a parameter that keeps the gate open,
and run it both ways. The results must be the same, at the same cycles.
A wake that comes too late does not stop the model. It only makes the
woken unit a cycle late, now and then, and so it changes the results.

## Example: the third pairing, and its fix

A producer pushes a value every third cycle and wakes its consumer. It
acts first (`&mut FifoTx`). Each value is the cycle it was pushed in:

```rust
#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub out: orr::FifoTx<u64>,
    pub consumer: orr::Wake,
}

#[orr::module_impl]
impl Producer {
    /// Pushes the edge it is at, every third edge, and wakes the consumer.
    #[orr::task(triggers = (clock,), wakes = (consumer,))]
    fn send(ctx: &mut orr::TaskCtx, out: &mut orr::FifoTx<u64>) -> Result<(), orr::SimError> {
        let now = edge(ctx.now());
        if now % 3 == 0 && out.push(now).is_ok() {
            ctx.wakes().0.wake();
        }
        Ok(())
    }
}
```

There are two consumers. Each pops a value a cycle with `&'latest`, and
closes its gate when the FIFO is empty:

- `Same` runs at the same delta level as the producer;
- `Later` runs one delta level later.

Each counts how many cycles its values waited:

```rust
macro_rules! consumer {
    ($name:ident, $($trigger:tt)*) => {
        #[orr::module]
        pub struct $name {
            #[orr::trigger(gate = gate)]
            pub clock: orr::ClockRelay,
            #[orr::wake(gate = gate)]
            pub wake: orr::WakeRelay,
            #[orr::interface]
            pub inbox: orr::FifoRx<u64>,
            #[orr::constant]
            pub gating: bool,
            gate: orr::Gate,
            /// Values popped, and the cycles they waited in all.
            pub popped: u64,
            pub waited: u64,
        }

        #[orr::module_impl]
        impl $name {
            #[orr::task(triggers = ($($trigger)*,))]
            fn take(
                ctx: &mut orr::TaskCtx,
                gating: &'constant bool,
                inbox: &'latest mut orr::FifoRx<u64>,
                gate: &mut orr::Gate,
                popped: &mut u64,
                waited: &mut u64,
            ) -> Result<(), orr::SimError> {
                if let Some(pushed) = inbox.pop() {
                    *popped += 1;
                    *waited += edge(ctx.now()) - pushed;
                }
                if *gating && inbox.is_empty() {
                    gate.close(); // until the producer wakes it
                }
                Ok(())
            }
        }
    };
}

consumer!(Same, clock);
consumer!(Later, clock + delta(1));
```

The tops connect the wake and the FIFO. The driver runs each consumer
for 1000 cycles, with the gate never closing and with it closing:

```rust
macro_rules! top {
    ($top:ident, $consumer:ident) => {
        #[orr::module]
        pub struct $top {
            #[orr::param("whether the consumer closes its gate when its FIFO is empty", default = true)]
            gating: bool,
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            fifo: orr::Fifo<u64>,
            #[orr::submodule]
            producer: Producer,
            #[orr::submodule]
            consumer: $consumer,
        }

        #[orr::module_impl]
        impl $top {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
                ctx.connect(module.clock, module.producer.clock)?;
                ctx.connect(module.clock, module.consumer.clock)?;
                ctx.connect(module.producer.consumer, module.consumer.wake)?;
                ctx.set_const(module.consumer.gating, *module.gating)?;
                ctx.connect_interface(&module.fifo, module.producer.out)?;
                ctx.connect_interface(&module.fifo, module.consumer.inbox)
            }
        }
    };
}

top!(SameTop, Same);
top!(LaterTop, Later);

fn main() -> Res<()> {
    let end = orr::Time::ps(PERIOD_PS * 1000 - 1);
    for gating in [false, true] {
        let file = params(&format!("ch4-gating{gating}"), &format!("[top]\ngating = {gating}\n"))?;
        let mut sim = orr::Sim::new(orr::SimPlan::<SameTop>::new(Some(file.clone()), None)?, orr::SimConfig::default())?;
        sim.run_until(end)?;
        let (popped, waited) = (*sim.state(sim.top().consumer.popped)?, *sim.state(sim.top().consumer.waited)?);
        println!("same level,    gating {gating}: {popped} values, waited {:.2} cycles on average", waited as f64 / popped as f64);
        let mut sim = orr::Sim::new(orr::SimPlan::<LaterTop>::new(Some(file), None)?, orr::SimConfig::default())?;
        sim.run_until(end)?;
        let (popped, waited) = (*sim.state(sim.top().consumer.popped)?, *sim.state(sim.top().consumer.waited)?);
        println!("delta level 1, gating {gating}: {popped} values, waited {:.2} cycles on average", waited as f64 / popped as f64);
    }
    Ok(())
}
```

It prints:

```text
same level,    gating false: 334 values, waited 0.00 cycles on average
delta level 1, gating false: 334 values, waited 0.00 cycles on average
same level,    gating true: 333 values, waited 1.00 cycles on average
delta level 1, gating true: 334 values, waited 0.00 cycles on average
```

- With the gate always open, both consumers take every value in the
  cycle it is pushed.
- With the gate closing, `Same` takes every value a cycle late, and one
  value fewer before the end. The producer's wake comes in the same
  timepoint in which `Same` would have read: too late.
- `Later` runs one delta level after the wake, so the wake is in time.
  Its results with the gate closing are the same as without.

## Which to use: a wake, or an event that is not behind the gate

- **Use a wake** when the woken unit finds its work in its own state or
  in its FIFOs. The wake carries nothing and runs nothing; it only opens
  the gate.
- **Use an event that is not behind the gate** when the work *is* the
  event: a job, a request. The event runs the task, which takes the work
  and opens its own gate ([chapter 2](./gates)'s workers). Then there is
  no order to get right: the event arrives at a time its sender chose.
- **A unit that knows when its next work comes** needs neither: it
  closes until then ([chapter 3](./due)).
- **For a value due *n* edges on**, wake with
  `wake_after(orr::wake_delay(now, clock, n)?)`. That opens the gate at
  the edge where the value is seen, not at the next one.

## What it does to the three measures

- A wake is about being right, not about speed. Get it wrong and the
  model is a cycle late, now and then. The comparison with the gate
  always open catches it; the speed does not.
- **Reading.** Each wake is one more thing to keep in mind: who wakes,
  from which delta level, and whether that is in time. For each wake,
  write in a comment the pairing from the table that makes it in time.

## In the GPU model

In the GPU model that comes with Orrery, an L2 bank's `reply` step moves
a reply into the network. It wakes the network's port, which moves the
reply on the next cycle at the earliest. The model's top-level comment
works out, for each wake, which side acts first and at which level.

## Checklist

- For each wake, does the woken task act before or after the task that
  wakes it, in the same cycle? Which row of the table is it?
- In the third row, did I move the consumer to a later level, or keep
  its gate open while a value can arrive?
- Run with the gate never closing: are the results the same, at the same
  cycles?
- Where the work itself is an event, did I use an event that is not
  behind the gate instead of a wake?
