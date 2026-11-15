# 4. 叫醒要赶得上

**规则。** 关到有活为止的单元，由送活来的单元用叫醒打开（[第 2 章](./gates)）。叫醒从**下一个**时刻点起打开门：它不能让被叫醒的 task 在发出叫醒的那个时刻点里跑。活能不能按时取走，取决于一拍里两个 task 谁先动。

经 FIFO 传的值，在这张表里找你的这一对 task：

| 生产者 | 消费者 | 消费者关着时推进去的值，什么时候被取走 |
|---|---|---|
| `&'latest mut FifoTx`（后动） | `&mut FifoRx`（先动） | 下一拍，和门一直开着一样：赶得上 |
| `&mut FifoTx`（先动） | `&'latest mut FifoRx`，在 `clock + delta(1)` | 同一拍，和门一直开着一样：赶得上 |
| `&mut FifoTx`（先动） | `&'latest mut FifoRx`，在同一层 delta | **晚一拍**：门一直开着时同一拍就取走 |

第三种配对，二选一：

- 把消费者挪到晚一层 delta；
- 或者在可能有值送到的时候让它的门开着。

**等地方是反过来的。** 生产者手里有一个值、FIFO 满了，它等消费者弹出。表的第一行里消费者先动：它这一拍空出的位置，门开着的生产者这一拍就能放进去。消费者发的叫醒要到下一个时刻点才打开生产者的门：晚一拍。所以这样的生产者在等地方时门要开着。生产者先动时，空出的位置它本来就要下一拍才看得见，消费者的叫醒赶得上。

一拍里 task 的先后见指南[谁先动作](../guide/components#who-acts-first)，叫醒的调用见[唤醒](../guide/time#wakes)。

**怎么核对。** 给模型一个让门一直开着的参数，两种都跑一遍：结果必须一样，而且在同样的拍上。叫醒来晚了，模型不会停；它只会让被叫醒的单元时不时晚一拍，于是结果变了。

## 例子：第三种配对，和改法

生产者每三拍推一个值，并叫醒消费者。它先动（`&mut FifoTx`）。每个值就是它被推进去的那一拍：

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

有两个消费者。每个都用 `&'latest` 一拍取一个值，FIFO 空了就关门：

- `Same` 和生产者在同一层 delta 上跑；
- `Later` 晚一层。

每个都数它取到的值等了几拍：

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

顶层接上叫醒和 FIFO。驱动程序让每个消费者跑 1000 拍，门从不关、闲了就关各一次：

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

它打出：

```text
same level,    gating false: 334 values, waited 0.00 cycles on average
delta level 1, gating false: 334 values, waited 0.00 cycles on average
same level,    gating true: 333 values, waited 1.00 cycles on average
delta level 1, gating true: 334 values, waited 0.00 cycles on average
```

- 门一直开着时，两个消费者都在值推进去的那一拍取走它。
- 门会关时，`Same` 每个值都晚一拍取走，到结束时还少取一个。生产者的叫醒，正好在 `Same` 本该读的那个时刻点里发出：来晚了。
- `Later` 在叫醒之后晚一层才跑，叫醒赶得上；门会关时的结果和不关一样。

## 用哪个：叫醒，还是不在门后的事件

- **用叫醒**：被叫醒的单元在自己的状态或 FIFO 里找活时。叫醒什么也不带，也不让 task 跑；它只开门。
- **用不在门后的事件**：活本身就是这个事件时，比如一份活、一个请求。事件让 task 跑起来，task 收下活、自己开门（[第 2 章](./gates)的工人）。这样没有先后要弄对：事件在发送方选定的时刻到。
- **知道下一份活什么时候来的单元**，两样都不要：它自己关到那时（[第 3 章](./due)）。
- **一个 *n* 个沿后到期的值**，用 `wake_after(orr::wake_delay(now, clock, n)?)` 叫醒：门在值被看见的那个沿打开，而不是下一个沿。

## 对三条指标的影响

- 叫醒关乎对不对，不关乎快慢。写错了，模型时不时晚一拍；和门一直开着对照能查出来，看速度查不出来。
- **读**：每个叫醒都是一样要记住的东西：谁叫、从哪一层叫、赶不赶得上。每个叫醒，在注释里写明它是表里哪一种配对、所以赶得上。

## GPU 模型里

随 Orrery 附带的 GPU 模型里，L2 bank 的 `reply` 这一步把一个回复送进网络，并叫醒网络的那个口，那个口最早在下一拍把它送走。模型最上面的注释，对每个叫醒都推了一遍：哪一边先动、在哪一层。

## 核对单

- 每个叫醒：被叫醒的 task 在同一拍里，是在叫它的 task 之前还是之后动？是表里哪一行？
- 第三行的情形，把消费者挪到晚一层了吗？或者在可能有值送到时让门开着？
- 门从不关时，结果一样吗？在同样的拍上吗？
- 活本身就是事件的地方，用的是不在门后的事件，而不是叫醒吗？
