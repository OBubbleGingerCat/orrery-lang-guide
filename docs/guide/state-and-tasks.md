# State and tasks

A task is a function in a module's `#[orr::module_impl]` block that the
module's triggers run. It reads and writes the module's state, and the way
it declares each state says how it is ordered against the other tasks
that touch it in the same timepoint.

## A task

```rust
use orrery_lang as orr;

#[orr::task(triggers = (clock,), emits = (done,))]
fn produce(ctx: &mut orr::TaskCtx, value: &mut u64) -> Result<(), orr::SimError> {
    *value += *ctx.module().step;
    ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;
    Ok(())
}
```

- `triggers = (..)` lists the triggers that run it (any of them firing
  runs it, at most once per timepoint and instance); `emits = (..)` lists
  the event sources it may send to, and `wakes = (..)` the
  [wakes](./time#wakes) it may use. An entry is a field, `field[i]` for
  one element, or a whole array.
- The first parameter is `ctx: &mut orr::TaskCtx`; then come the state
  parameters, in any order.
- It returns `Result<(), orr::SimError>`, or `Result<(), orr::Stop>` if it
  may end the simulation ([Running](./running#stopping)).

## Access modes

A state parameter binds the state field of the same name, and its
reference type is its **access mode**:

| Parameter | Mode | Sees |
|---|---|---|
| `x: &T` | `curr` | the value before this timepoint's writer |
| `x: &'latest T` | `latest` | the value after this timepoint's writer |
| `x: &mut T` | `write` | it is the writer: reads and writes the value |

A task also reads its module's [constants](./modules#constants), as
`name: &'constant T`. A constant is not state: it has no access mode and
orders nothing.

Each location has at most one writer per timepoint. Within a timepoint,
every `curr` reader runs before the writer and every `latest` reader after
it, whatever order the tasks are declared in. If nothing writes in a
timepoint, both readers see the same value.

```rust
#[orr::module]
pub struct Stage {
    #[orr::trigger]
    clock: orr::Clock,

    reg: u64,
    pub seen_before: u64,
    pub seen_after: u64,
}

#[orr::module_impl]
impl Stage {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))
    }

    #[orr::task(triggers = (clock,))]
    fn advance(_ctx: &mut orr::TaskCtx, reg: &mut u64) -> Result<(), orr::SimError> {
        *reg += 1;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn before(_ctx: &mut orr::TaskCtx, reg: &u64, seen_before: &mut u64) -> Result<(), orr::SimError> {
        *seen_before = *reg;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn after(_ctx: &mut orr::TaskCtx, reg: &'latest u64, seen_after: &mut u64) -> Result<(), orr::SimError> {
        *seen_after = *reg;
        Ok(())
    }
}
```

After five clock edges (0 to 4 ns), `reg` is 5, `seen_before` is 4 and
`seen_after` is 5, on any number of threads.

This is how registers and wires are modeled: a pipeline register read with
`&` gives last cycle's value; a combinational signal read with `&'latest`
gives this cycle's.

`&` does not read a copy kept from the last cycle: it runs the reader
before this timepoint's writer. So a chain of registers, each read by the
stage after it, can be split into a task per stage, because the readers
can always go first. Two registers that read each other cannot, when each
is written by a task of its own: each task would have to run before the
other, and `Sim::new` reports a dependency cycle:

```text
dependency cycle (triggered by clock top.clock (1ns phase 0ps)): %Pair.step_a at top reads top.b (curr) before %Pair.step_b at top writes it; %Pair.step_b at top reads top.a (curr) before %Pair.step_a at top writes it
```

Write the two in one task, which reads both old values before it writes
either:

```rust
#[orr::task(triggers = (clock,))]
fn step(_ctx: &mut orr::TaskCtx, a: &mut u64, b: &mut u64) -> Result<(), orr::SimError> {
    let (old_a, old_b) = (*a, *b);
    *a = old_b + 1;
    *b = old_a * 10;
    Ok(())
}
```

or keep each register's next value in a field of its own, and copy it in
with a task that reads it with `&'latest`:

```rust
#[orr::task(triggers = (clock,))]
fn step_a(_ctx: &mut orr::TaskCtx, a_next: &mut u64, b: &u64) -> Result<(), orr::SimError> {
    *a_next = *b + 1;
    Ok(())
}

#[orr::task(triggers = (clock,))]
fn update_a(_ctx: &mut orr::TaskCtx, a: &mut u64, a_next: &'latest u64) -> Result<(), orr::SimError> {
    *a = *a_next;
    Ok(())
}

// `step_b` and `update_b` the same way
```

Either way, after the edges at 0, 1 and 2 ns `a` is 11 and `b` is 10, as
with two flip-flops, on any number of threads.

## Binding other fields

`#[orr::bind(field)]` binds a parameter to a field of another name, and
`#[orr::bind(field[3])]` to one element of an array:

```rust
fn tick(_ctx: &mut orr::TaskCtx,
        #[orr::bind(values[1])] second: &mut u64) -> Result<(), orr::SimError>
```

A whole array is `&[T]`, `&'latest [T]` or `&mut [T]`. Part of one is
`orr::PartialAccess`, with literal indices or ranges; here elements 0
and 2:

```rust
#[orr::bind(values)] even: &mut [u64; orr::PartialAccess(0, 2)],
```

| Written | Binds |
|---|---|
| `PartialAccess(3)` | element 3 |
| `PartialAccess(0, 2)` | elements 0 and 2, not the first two |
| `PartialAccess(0..2)` | elements 0 and 1: a half-open range |
| `PartialAccess(3, 1..3)` | elements 3, 1 and 2, in the order written |

Indices and the bounds of ranges are integer literals, not constants,
parameters or expressions (`N`, `2 * K`). A range is written
`start..end` and is not empty (`1..=3`, `..3` and `2..` are errors); an
index may not repeat, ranges may not overlap, and `PartialAccess()` is an
error.

A container field takes its own type, or its load-time form:
`&mut orr::Queue<Req, orr::DynShape(depth)>` is a `&mut orr::DynQueue<Req>`
inside the task.

An array of load-time containers is taken whole as a view,
`orr::ArrayView`, written as the field is:
`buffers: &mut orr::ArrayView<[orr::Queue<Req, orr::DynShape(depth)>]>`.
`buffers.at(i)` reads element *i* and `buffers.at_mut(i)` writes it, each a
`DynQueue`; `&` and `&'latest` take it read only. One element alone is
`#[orr::bind(buffers[2])]`, as above.

The type a parameter names must be the field's type; a mismatch does not
compile: "a task parameter of type `u32` cannot bind a field of type
`u64`".

## Groups of columns {#columns}

A unit whose state several tasks share, field by field (a core's
counters, credits and flags), can keep those fields as one **group of
columns**: a task takes the whole group by one parameter instead of one
parameter per field, while to the rest of the program each member stays
a field of its own.

- The group is a struct of named members, each a value of a fixed-size
  plain type, with `#[derive(orr::Columns)]`. A member starts at its
  type's default, or as `#[reset(value)]` or `#[reset(with = f)]` on the
  member says.
- The module has a field of the group, marked `#[orr::columns]`.
- A task binds the field whole by one parameter: `&mut Core` to write it,
  `&Core` to read it as the timepoint began, `&'latest Core` to read it
  after this timepoint's writes. Not `&'latest mut Core`.
- In the task, `.cols()` gives one handle per member, named as the member,
  which dereferences to its value: `*c.credits -= 1`. On a `&mut` group,
  `cols()` borrows the parameter, so write it `mut core: &mut Core`;
  `core.reborrow()` lends it to a function and keeps it, and
  `core.into_cols()` takes it.
- A function the task calls takes `orr::ColsMut<'_, Core>` (to write) or
  `orr::ColsRef<'_, Core>` (to read).
- A task that uses only a few members binds each alone:
  `#[orr::bind(core.issued)] issued: &'latest u64`.

```rust
#[derive(orr::Columns)]
pub struct Core {
    pub issued: u64,
    #[reset(4)]
    pub credits: u32,
    pub stalled: bool,
}

fn issue(core: orr::ColsMut<'_, Core>) {
    let mut c = core.into_cols();
    if *c.credits > 0 {
        *c.credits -= 1;
        *c.issued += 1;
    }
}

#[orr::module]
pub struct Sm {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::columns]
    pub core: Core,
    pub idle: u64,
    pub seen: u64,
}

#[orr::module_impl]
impl Sm {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, mut core: &mut Core) -> Result<(), orr::SimError> {
        issue(core.reborrow());
        let mut c = core.cols();
        *c.stalled = *c.credits == 0;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn count(_ctx: &mut orr::TaskCtx, core: &'latest Core, idle: &mut u64) -> Result<(), orr::SimError> {
        if *core.cols().stalled {
            *idle += 1;
        }
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn watch(
        _ctx: &mut orr::TaskCtx,
        #[orr::bind(core.issued)] issued: &'latest u64,
        seen: &mut u64,
    ) -> Result<(), orr::SimError> {
        *seen = *issued;
        Ok(())
    }
}
```

After ten edges, `issued` is 4, `credits` 0, `idle` 7 and `seen` 4, on any
number of threads.

Each member is a column of its own: the plan orders tasks by the members
they write and read, and the host reads one by its name,
`sim.state(sm.core.issued)`. What a group does not allow:

- a member of a load-time size, a gate, an interface or an external
  port does not compile; a compile-time container such as
  `orr::ArrayVec<T, N>` is refused when the plan is built;
- a task that writes the group binds it by that one parameter: a second
  parameter for the same field, or for one of its members, does not
  compile;
- a member bound alone is bound whole (`core.issued`), not a part of it;
- a field of columns is not an array of groups, and is not reset as a
  whole.

## Within one task

Two parameters of one task may refer to the same location (through
references or interfaces, say) only if both are `&` or both are
`&'latest`. Anything else is an error when the plan is built, since one
task cannot run both before and after the writer.

## The task context

| Method | Gives |
|---|---|
| `ctx.module()` | the module's view: parameters as values (`*ctx.module().step`) |
| `ctx.now()` | the current `orr::Timepoint` (`.fs()`, `.ps()`, `.delta()`, displayed as `10ns` or `10ns+1`) |
| `ctx.triggered_triggers()` | a tuple, one slot per `triggers` entry: `Option<&P>` for a field, `&[Option<&P>]` for an array; a clock's payload is its `ClockInfo` |
| `ctx.emit_triggers()` | a tuple, one emitter per `emits` entry: an `orr::EventEmitter`, or an `orr::EventEmitters` for an array |
| `ctx.wakes()` | a tuple, one per `wakes` entry: an `orr::Waker`, or an `orr::Wakers` for an array ([Wakes](./time#wakes)) |

```rust
let (clock, inputs) = ctx.triggered_triggers();
for packet in inputs.iter().flatten() {
    // every packet that arrived on any input this timepoint
}
let outputs = &mut ctx.emit_triggers().0;
outputs[port].emit(packet, link_delay)?; // one element, by index
for (i, mut output) in outputs.iter().enumerate() {
    output.emit(i as u32, orr::Time::ns(i as u64 + 1))?; // every element
}
if let Some(mut output) = outputs.get(9) {
    output.emit(packet, link_delay)?; // `None`: there is no element 9
}
```

An `EventEmitters` gives element *i*'s emitter as `outputs[i]`, which
panics past the end, or as `outputs.get(i)`, which gives `None` there;
`iter()` gives each element's in order, and `len()` how many there are.

Emitters and payloads are borrowed for the call only. Sending events is
described in [Time](./time#events).

## Errors

A task that returns `Err(orr::SimError)` (or panics) ends the run with an
error naming the task, the instance and the timepoint: `Err(...)` from
`run_until`. Use `orr::SimError::new("message")`, or `?` on anything that
returns one. Everything that comes before the failing task in serial
order has run; what came after it is not specified. See
[Errors and debugging](./errors#which-error-is-reported).
