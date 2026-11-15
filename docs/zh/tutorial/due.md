# 3. 关到下一个到期的沿

**规则。** 单元常要等一件它知道时间的事：固定的延迟、一个定时器、一个要等满的时序参数。这种等待不要一拍一拍地倒数：

- 每个在等的值，和它到期的沿存在一起，存成 `orr::Due<T>`。几个在等的值放进 `orr::DelayQueue`（模块状态里），或者 `orr::Fifo<orr::Due<T>>`（模块之间）。
- 没有到期的东西时，把门关到第一个值被看见的那个沿：`gate.close_until(clock.edge_after(orr::Timepoint::at_fs(first.at)))`。

没有人要来叫醒它，也就没有叫醒要写对。

**怎么核对。** 把这个单元再按笨办法写一遍，每拍倒数，两者对照：它们必须在同样的拍上做同样的事。笨办法的那一版留作测试。

细节见指南：[延迟](../guide/components#delays)、[时钟门控](../guide/time#clock-gates)。

## 例子：延迟固定的存储器

64 个存储器，每个在请求到达 100 拍之后回答它。每 20 拍来一个请求，所以任何时候大约有五个在等。

笨办法：每个在等的请求记着它还剩几拍，存储器每拍把它们全部倒数一遍：

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

这一章的办法：每个在等的请求记着它到期的沿，存储器睡到第一个到期为止：

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

时钟和延迟是常量：

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

到期的沿怎么算：

- `CLOCK.nth_edge_from(now, LATENCY)` 是把 `now` 上的沿算作第一个、数到的第 100 个沿。所以在第 *E* 个沿到达的请求，在第 *E* + 99 个沿到期。
- 值在严格晚于到期沿的时刻才被看见。所以 `pop_due(now)` 在第 *E* + 100 个沿交出这个请求；倒数到 0，也正是这个沿。
- 存储器用一个常量 `ClockInfo` 来算，而不用它的时钟触发源：门关着、由请求事件跑起来的 task，拿不到时钟那一格。

两个顶层，以及驱动程序：它把两种都量一遍，并核对两者在同样的拍上回答了同样的请求：

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

## 对三条指标的影响

运行的秒数（粗量）。执行单元数是 2、2、4、8：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 |
|---|---|---|---|---|---|
| 每拍倒数 | 0.139 | 0.158 | 0.084 | 0.057 | 2.43× |
| 关到第一个到期 | 0.068 | 0.083 | 0.057 | 0.047 | 1.43× |

- **快。** 关到到期**更好**：1 线程快到 2.1 倍，2 线程 1.9 倍，4 线程 1.5 倍；8 线程上也快，1.2 倍。
  - 倒数的写法每拍都要跑每个存储器，总共约 320 万次调用。
  - 关到到期的写法，每个请求只跑两次（到达时、到期时），调用约为前者的十分之一。
  - 剩下的是回答请求的活，两种一样。
- **加速比**从 2.4 降到 1.4：省下的主要是 1 线程的时间（原因见[第 2 章](./gates)）。比时间。
- **读。** 一个字段（`waiting`）换掉了两个数组和一个循环；读的人不用再去跟"还剩几拍"。也没有叫醒要核对：门是单元自己开的。

到期的沿必须准到沿。差一个沿，单元就早一拍或晚一拍回答，结果就变了；和倒数的写法对照，能查出来。常见的错是从错的沿开始数：在晚一层 delta 上跑的 task（`clock + delta(k)`）必须从它那个时刻的 delta 0 数起，`orr::Timepoint::at_fs(ctx.now().fs())`，不然就是从下一个沿数起（[延迟](../guide/components#delays)）。

## GPU 模型里

随 Orrery 附带的 GPU 模型，每个固定延迟都存成到期的沿：

- SM 的执行流水线里存的是 `orr::Due` 条目；
- L2 bank 的 ROP 延迟队列是一个 `orr::DelayQueue`；
- DRAM 通道的延迟队列里，请求存成 `orr::Due` 值。

L2 bank 的 cache 这一步、每个 DRAM 通道，都以同样的三种情形结尾：

- 忙：开着；
- 没事做：`gate.close()`，等人叫醒；
- 到某个已知的沿之前都没事：`gate.close_until(clock.edge_after(..))`。

## 什么时候不该这样写

- **等多久事先不知道**，比如要等另一个单元腾出地方。那就关到被叫醒（[第 2 章](./gates)），或者开着。
- **延迟不同的值共用一个队列。** `DelayQueue` 只看队头，值按进去的顺序出来。一种固定延迟用一个队列。
- **单元本来就几乎拍拍都忙。** 存到期的沿仍比倒数清楚，但不会更快。

## 核对单

- 有没有每拍减一的计数？换成到期的沿。
- 门关到的沿，是队头的值被看见的那个沿（`edge_after(at_fs(front.at))`）吗？
- task 在晚一层 delta 上跑时，是从它那个时刻的 delta 0 数起的吗？
- 笨办法（倒数）的那一版，在同样的拍上给出同样的结果吗？
