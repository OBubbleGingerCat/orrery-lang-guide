# Components and interfaces

## Plain types

Everything stored as state or sent as an event payload is a **plain**
type: a fixed size, copied by bytes, nothing to drop, no pointers.

- Built in: integers, floats, `bool`, `()`, `orr::Time`,
  `orr::ClockInfo`, `Option<T>`, arrays and tuples of plain types.
- Your own: `#[derive(orr::Plain)]` on a struct or enum whose fields are
  plain.

```rust
use orrery_lang as orr;

#[derive(Debug, Default, Clone, Copy, PartialEq, orr::Plain)]
pub enum Op {
    #[default]
    Idle,
    Read(u64),
    Write { addr: u64, data: [u8; 4] },
}
```

`Vec`, `String`, `Box` and references are not plain; use a container.
Parameters are not state and need only `orr::ParamValue`.

## Containers

Containers hold a bounded number of plain elements, with the capacity
fixed at compile time (`Queue<T, 8>`) or per instance when the model loads
(`Queue<T, orr::DynShape(depth)>`, sized by a parameter):

| Container | What it is |
|---|---|
| `orr::Array<T, N>` | N elements stored as **one** location, always taken whole |
| `orr::ArrayVec<T, N>` | a bounded vector: `push` returns the value back when full |
| `orr::Queue<T, N>` | a ring FIFO: `push`, `pop`, `front`, `iter`, … |
| `orr::Map<K, V, N>` | a small map with linear lookup |
| `orr::BitSet![N]` | a bit set with the usual set operations |
| `orr::Store<S>` | a sparse memory Orrery manages, such as `orr::SparseMem<orr::CacheLine<64>>`, cells allocated as they are written |

Inside a task, a load-time container appears in its `Dyn` form
(`orr::DynQueue<T>`), which is also what helper functions take.

## Components

A **component** is a reusable module that others use through an
**interface**. Orrery's standard component is `orr::Fifo<T>`, a bounded
queue between two modules: a producer side, `orr::FifoTx<T>`, and a
consumer side, `orr::FifoRx<T>`.

```rust
#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub out: orr::FifoTx<u64>,

    next: u64,
    pub stalls: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,))]
    fn send(
        _ctx: &mut orr::TaskCtx,
        out: &'latest mut orr::FifoTx<u64>,
        next: &mut u64,
        stalls: &mut u64,
    ) -> Result<(), orr::SimError> {
        match out.push(*next) {
            Ok(()) => *next += 1,
            Err(_) => *stalls += 1,
        }
        Ok(())
    }
}

#[orr::module]
pub struct Consumer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,

    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    #[orr::task(triggers = (clock,))]
    fn receive(
        _ctx: &mut orr::TaskCtx,
        inbox: &mut orr::FifoRx<u64>,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        Ok(())
    }
}
```

The module that owns the `Fifo` connects both sides to it:

```rust
#[orr::module]
pub struct Top {
    #[orr::trigger]
    clock: orr::Clock,

    #[orr::submodule]
    fifo: orr::Fifo<u64>,

    #[orr::submodule]
    producer: Producer,

    #[orr::submodule]
    consumer: Consumer,
}

#[orr::module_impl]
impl Top {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))?;
        ctx.connect(module.clock, module.producer.clock)?;
        ctx.connect(module.clock, module.consumer.clock)?;
        ctx.connect_interface(&module.fifo, module.producer.out)?;
        ctx.connect_interface(&module.fifo, module.consumer.inbox)
    }
}
```

The queue's depth is a parameter of the `Fifo` instance:

```toml
[top.fifo]
depth = 2
```

`FifoTx` has `push`, `is_full`, `free`, `len`, `capacity`; `FifoRx` has
`pop`, `front`, `get(i)`, `is_empty`, `len`, `capacity`.

## Who acts first

When both sides of a FIFO run in the same cycle, one of them must act
first, and the result depends on which: can a value pushed this cycle be
popped this cycle, and can a slot freed this cycle be reused? The
interface parameter's form says which, on each side, and **exactly one
side must be `&'latest mut`**:

| Producer | Consumer | Meaning |
|---|---|---|
| `&'latest mut FifoTx` | `&mut FifoRx` | the consumer acts first: a value pushed in cycle *k* can be popped from *k* + 1; a slot freed by a pop can be refilled in the same cycle |
| `&mut FifoTx` | `&'latest mut FifoRx` | the producer acts first: a value pushed can be popped in the same cycle; a freed slot is reusable from the next cycle |

Both `&mut`, or both `&'latest mut`, is an ordering cycle and an error when
the simulation loads, naming both tasks and the FIFO's state
([Errors](./errors#when-the-simulation-loads)). The order belongs to the
tasks, not to the component, and only matters in timepoints where both
run (where two clocks meet, say).

One task holds both sides of one FIFO only to look at both, as
`&FifoTx<T>` and `&FifoRx<T>`: it sees the whole FIFO as the cycle began.
If either side is `&mut` or `&'latest mut`, or one is `&` and the other
`&'latest`, which would read the same counter at two points of the
cycle, that is an error when the plan is built:

```rust
// Fine: both sides, both `&`.
#[orr::task(triggers = (clock,))]
fn watch(
    _ctx: &mut orr::TaskCtx,
    out: &orr::FifoTx<u64>,
    inbox: &orr::FifoRx<u64>,
    free: &mut u64,
) -> Result<(), orr::SimError> {
    *free = (out.capacity() - inbox.len()) as u64;
    Ok(())
}
```

Two tasks may not hold the same side with `&mut` or `&'latest mut`:
where they run together, that is a write conflict when the simulation
loads. Other tasks may still look at a side
([Looking without popping](#looking-without-popping)).

**Two units that send to each other.** When each of two units pushes into
a FIFO that the other pops (a request path and its reply path, two
neighbours in a mesh), one task per unit that both pops its inbox and
pushes its outbox is an ordering cycle, whichever sides are
`&'latest mut`: one FIFO puts A before B, the other B before A.

```text
dependency cycle (triggered by clock top.clock (1ns phase 0ps)): %OneTask.step at top.a reads top.ba.slots (curr) before %OneTask.step at top.b writes it; %OneTask.step at top.b reads top.ab.slots (curr) before %OneTask.step at top.a writes it
```

Give each unit a task that takes, acting first, and one that sends,
acting after; every unit's `take` then runs before every unit's `send`:

```rust
#[orr::module]
pub struct Node {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub out: orr::FifoTx<u64>,
    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,
    pub got: u64,
    sent: u64,
}

#[orr::module_impl]
impl Node {
    // Every node takes first, as the consumer of its inbox ...
    #[orr::task(triggers = (clock,))]
    fn take(_ctx: &mut orr::TaskCtx, inbox: &mut orr::FifoRx<u64>, got: &mut u64) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *got += value;
        }
        Ok(())
    }

    // ... and sends after, as the producer of its outbox.
    #[orr::task(triggers = (clock,))]
    fn send(_ctx: &mut orr::TaskCtx, out: &'latest mut orr::FifoTx<u64>, sent: &mut u64) -> Result<(), orr::SimError> {
        *sent += 1;
        let _ = out.push(*sent);
        Ok(())
    }
}
```

Where the link has a latency anyway, send over it with events whose
source declares that latency as its minimum
([Declared delays](./time#declared-delays)): nothing sent in a cycle
arrives in that cycle, the units need no order within it, and one task
per unit is enough.

## Looking without popping

Sometimes several tasks need to see what waits in a FIFO while only one
of them pops: an arbiter that looks at the front of each of its input
queues, a task that counts how full a buffer is. Such a task takes the
end by shared reference, `&FifoRx<T>` or `&'latest FifoRx<T>`, and calls
the methods that only look: `front`, `get(i)`, `len`, `is_empty`,
`capacity`. Any number of tasks may look at an end this way, besides the
one task that pops from it; they cannot change it, and `pop` through a
`&FifoRx` does not compile.

What a task sees of the current cycle, by the form it takes a side in,
and where it runs:

| Form | Sees the other side's changes this cycle | Sees this side's writer this cycle | Runs |
|---|---|---|---|
| `&mut` | no | it is the writer | before the other side |
| `&'latest mut` | yes | it is the writer | after the other side |
| `&` | no | no | before the writers of both sides |
| `&'latest` | yes | no | after the other side's writer, before this side's |

On the consumer's end, `&FifoRx` sees the FIFO as the cycle began, before
this cycle's push and before this cycle's pop. `&'latest FifoRx` sees
this cycle's push, but still not this cycle's pop. The producer's end
works the same way, `&FifoTx` and `&'latest FifoTx` with the sides
swapped: a looking task never sees its own side's changes of the cycle.

`&'latest FifoRx` has to come after the producer and before the pop, so
it needs the producer to act first (`&mut FifoTx` and
`&'latest mut FifoRx`). When the consumer acts first there is no such
place: that is an ordering cycle, an error when the simulation loads
([Errors](./errors#when-the-simulation-loads)). `&FifoRx` fits either
order. Arrays work alike: `&[orr::FifoRx<T>]` and
`&'latest [orr::FifoRx<T>]`.
A looking task takes its place in the cycle's order like any task: `&`
before the writers of both sides, `&'latest` between them. A task that
looks at many FIFOs is ordered against the writers of all of them.

A complete example: the producer acts first and pushes a value every
cycle, the consumer pops one every cycle, and two more tasks of the
consumer look at the FIFO.

```rust
use orrery_lang as orr;

#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub out: orr::FifoTx<u64>,

    next: u64,
}

#[orr::module_impl]
impl Producer {
    // Acts first: pushes 1, 2, 3, ... while there is room.
    #[orr::task(triggers = (clock,))]
    fn send(
        _ctx: &mut orr::TaskCtx,
        out: &mut orr::FifoTx<u64>,
        next: &mut u64,
    ) -> Result<(), orr::SimError> {
        if out.push(*next + 1).is_ok() {
            *next += 1;
        }
        Ok(())
    }
}

#[orr::module]
pub struct Consumer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,

    pub waiting: u64,
    pub newest: u64,
    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    // Acts after the producer: takes one value a cycle.
    #[orr::task(triggers = (clock,))]
    fn receive(
        _ctx: &mut orr::TaskCtx,
        inbox: &'latest mut orr::FifoRx<u64>,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        Ok(())
    }

    // Looks before this cycle's push and pop.
    #[orr::task(triggers = (clock,))]
    fn peek(
        _ctx: &mut orr::TaskCtx,
        inbox: &orr::FifoRx<u64>,
        waiting: &mut u64,
    ) -> Result<(), orr::SimError> {
        *waiting = inbox.len() as u64;
        Ok(())
    }

    // Looks after this cycle's push, before its pop.
    #[orr::task(triggers = (clock,))]
    fn look(
        _ctx: &mut orr::TaskCtx,
        inbox: &'latest orr::FifoRx<u64>,
        newest: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(last) = inbox.len().checked_sub(1) {
            *newest = *inbox.get(last).unwrap();
        }
        Ok(())
    }
}
```

`Top` connects them as in the first example. In every cycle `peek` finds
the FIFO empty, `look` finds the value just pushed, and `receive` then
pops it: after ten cycles (0 to 9 ns), `waiting` is 0, `newest` is 10 and
`sum` is 55.

## Delays {#delays}

A value that becomes visible a fixed number of cycles after it is made (a
pipeline's latency, a wire, a memory's access time) is stored together
with the edge it is due at, as an `orr::Due<T>`: `value`, and `at`, the
due edge in femtoseconds. Its consumer sees it at a time strictly after
`at`; on the same clock, that is *n* cycles after it was handed on with a
delay of *n* edges.

- `clock.nth_edge_from(now, n)` is the due edge of a value handed on at
  `now` with a delay of *n* edges (*n* a `NonZeroU64`). It counts from the
  first edge at or after `now`, and an edge is at delta 0: from a task
  that runs at a later delta level (`clock + delta(k)`), it counts from
  the next edge, and the value is a cycle late. Such a task counts from
  `orr::Timepoint::at_fs(ctx.now().fs())`, delta 0 of its time.
- `orr::DelayQueue<T, N>` (`orr::DynDelayQueue<T>` when sized at load) is
  a queue of such values in a module's state. `push_due(value, due)`
  appends, or hands the value back when full; `front_due(now)` and
  `pop_due(now)` give the front value only once it is seen. It also has
  the queue's `front`, `len`, `is_empty`, `is_full`, `iter`, ….
- Between two modules, an `orr::Fifo<orr::Due<T>>` does the same: the
  producer's `push_due(value, due)`, the consumer's `front_due(now)` and
  `pop_due(now)`.
- Only the front is looked at, so values come out in the order they went
  in: one queue fits one fixed delay.
- These compare femtoseconds, for a consumer that runs at a clock edge. A
  consumer that runs on an event, at any delta, uses `is_due_exact`,
  `front_due_exact` and `pop_due_exact`.

A unit that waits for its delays closes its gate until its front value is
seen, `gate.close_until(clock.edge_after(orr::Timepoint::at_fs(front.at)))`,
and the unit that hands it a value wakes it for the edge the value is
seen, `wake_after(orr::wake_delay(now, clock, n)?)`
([Wakes](./time#wakes)). A memory that answers each
request `LATENCY` edges after it takes it, and sends the answer on a
wire of `WIRE` edges:

```rust
#[orr::module]
pub struct Memory {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::wake(gate = gate)]
    pub wake: orr::WakeRelay,
    #[orr::interface]
    pub requests: orr::FifoRx<u64>,
    #[orr::interface]
    pub answers: orr::FifoTx<orr::Due<u64>>,
    pub sink: orr::Wake,
    pub gate: orr::Gate,
    pending: orr::DelayQueue<u64, 8>,
}

#[orr::module_impl]
impl Memory {
    #[orr::task(triggers = (clock,), wakes = (sink,))]
    fn cycle(
        ctx: &mut orr::TaskCtx,
        requests: &mut orr::FifoRx<u64>,
        answers: &'latest mut orr::FifoTx<orr::Due<u64>>,
        pending: &mut orr::DelayQueue<u64, 8>,
        gate: &mut orr::Gate,
    ) -> Result<(), orr::SimError> {
        let clock = ctx.triggered_triggers().0.expect("runs on its clock");
        let now = ctx.now();
        // An answer whose latency is over goes out, due WIRE edges on.
        if pending.front_due(now).is_some() && !answers.is_full() {
            let address = pending.pop_due(now).expect("due");
            answers.push_due(address * 10, clock.nth_edge_from(now, WIRE)).expect("not full");
            ctx.wakes().0.wake_after(orr::wake_delay(now, clock, WIRE)?)?;
        }
        // A new request is answered LATENCY edges on.
        if !pending.is_full() {
            if let Some(address) = requests.pop() {
                pending.push_due(address, clock.nth_edge_from(now, LATENCY)).expect("not full");
            }
        }
        // Nothing to do until the next request, or until the oldest
        // request's latency is over.
        if requests.is_empty() {
            match pending.front() {
                Some(oldest) => gate.close_until(clock.edge_after(orr::Timepoint::at_fs(oldest.at))),
                None => gate.close(), // until the source wakes it
            }
        }
        Ok(())
    }
}
```

With `LATENCY` 5 and `WIRE` 3 (`NonZeroU64` constants), a request that a
source acting after the memory pushes at edge 10 is taken at edge 11, its answer leaves at edge 16 and is seen at
edge 19. The unit that takes the answers pops them with `pop_due(now)` and
closes its gate the same way. For a value due on one clock and seen on
another, `orr::delay_to_edge(now, clock, n, consumer_clock)` is the delay
to [emit an event](./time#events) with.

## Arrays and forwarding

Interfaces can be arrays:

```rust
#[orr::interface]
pub outs: [orr::FifoTx<u64>; orr::DynShape(lanes)],
```

The owner's `init` connects them element by element:

```rust
ctx.connect_interface(&module.fifos[lane], module.splitter.outs[lane])?;
```

A module that is not the component's direct parent declares the same
interface and has its parent forward it:
`ctx.share_interface(module.inbox, module.consumer.inbox)?`.

## Your own interfaces

An interface is a struct of parts, each a `pub` field of the component,
marked as owned (written by the holder) or as the other side's (read
only). This is how `orr::Fifo` and its producer side are written:

```rust
#[orr::module]
pub struct Fifo<T: orr::Plain + Default> {
    #[orr::param("capacity in elements", default = 2usize)]
    depth: usize,

    pub slots: orr::Array<T, orr::DynShape(depth)>,
    pub tail: u64,
    pub head: u64,
}

#[orr::module_impl]
impl<T: orr::Plain + Default> Fifo<T> {}

#[orr::interface(component = Fifo<T>)]
pub struct FifoTx<T: orr::Plain + Default> {
    slots: orr::Own<orr::DynArray<T>>,
    tail: orr::Own<u64>,
    head: orr::Other<u64>,
}

impl<T: orr::Plain + Default> FifoTx<T> {
    pub fn push(&mut self, value: T) -> Result<(), T> {
        let capacity = self.slots.len() as u64;
        if *self.tail - *self.head == capacity {
            return Err(value); // full
        }
        self.slots[(*self.tail % capacity) as usize] = value;
        *self.tail += 1;
        Ok(())
    }
}
```

Each part's name must be a `pub` field of the component with a compatible
type; this is checked at compile time. Under `&mut I`, a task writes the
owned parts and reads the others as they were (`curr`); under
`&'latest mut I`, it reads them after the other side (`latest`). A task
that only looks takes `&I` or `&'latest I` (`&[I]`, `&'latest [I]` for
arrays): it reads what its side owns as it was before this cycle's
changes, and the rest as `&mut I` or `&'latest mut I` would, and can
call only the methods that take `&self`. If your interface has an
`impl Drop`, its `drop` does not run for a task that takes it this way. Any
module can be a component, including modules with tasks of their own:
interfaces are a general way to give a module a typed, ordered port, much
like a SystemVerilog interface and its modports.
