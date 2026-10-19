# Components and interfaces

## Plain types

Everything stored as state or sent as an event payload is a **plain**
type: a fixed size, copied by bytes, nothing to drop, no pointers. That is
what lets Orrery store state in place and copy it into snapshots without
allocating.

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
the plan is built, naming both tasks and the FIFO. The order belongs to
the tasks, not to the component, and only matters in timepoints where
both run (where two clocks meet, say).

Interfaces are never read-only: `&FifoTx` does not compile. One task may
not hold both sides of one FIFO, and two tasks may not hold the same side.

## Arrays and forwarding

Interfaces can be arrays, connected element by element:

```rust
#[orr::interface]
pub outs: [orr::FifoTx<u64>; orr::DynShape(lanes)],
// in the owner's init:
ctx.connect_interface(&module.fifos[lane], module.splitter.outs[lane])?;
```

A module that is not the component's direct parent declares the same
interface and has its parent forward it:
`ctx.share_interface(module.inbox, module.consumer.inbox)?`.

## Your own interfaces

An interface is a struct of parts, each a `pub` field of the component,
marked as owned (written by the holder) or as the other side's (read
only):

```rust
#[orr::interface(component = Fifo<T>)]
pub struct FifoTx<T: orr::Plain + Default> {
    slots: orr::Own<orr::DynArray<T>>,
    tail: orr::Own<u64>,
    head: orr::Other<u64>,
}

impl<T: orr::Plain + Default> FifoTx<T> {
    pub fn push(&mut self, value: T) -> Result<(), T> {
        // ...
    }
}
```

Each part's name must be a `pub` field of the component with a compatible
type; this is checked at compile time. Under `&mut I`, a task writes the
owned parts and reads the others as they were (`curr`); under
`&'latest mut I`, it reads them after the other side (`latest`). Any
module can be a component, including modules with tasks of their own:
interfaces are a general way to give a module a typed, ordered port, much
like a SystemVerilog interface and its modports.
