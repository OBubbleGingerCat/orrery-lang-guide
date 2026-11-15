# 2. 闲着就关门

**规则。** 有些单元在东西送到之前无事可做：一份活、一个请求、FIFO 里的一个值。这样的单元把活做完了，就在 task 末尾关上自己的时钟门：`gate.close()`。下一份活来的时候把它打开。活有两种来法：一种是不在门后的事件，它会让 task 跑起来；另一种是叫醒（[第 4 章](./wakes)）。

有两处要注意：

- **门怎么写读着清楚就怎么写。** 门已经开着时再写一次 `open()`，几乎不花什么，不用先查 `is_open()`（下面的例子查了，不查也行）。
- **关掉门控来核对。** 给模型一个参数，让门一直开着。门关和不关，结果必须一样，而且在同样的拍上。

另见指南：[时钟门控](../guide/time#clock-gates)、[唤醒](../guide/time#wakes)。

## 例子：两份活之间睡觉的工人

一个派活的给 64 个工人派活。一份活要工人做 `len` 拍，每拍约 0.2 µs 的活。活以事件的形式送出，一拍后送到。这个事件不在门后，所以门关着时它也会让 task 跑起来：

```rust
#[orr::module]
pub struct Worker {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    /// Jobs: not behind the gate, so one can arrive while it is closed.
    #[orr::trigger]
    pub jobs: orr::EventRelay<u64>,
    #[orr::constant]
    pub gating: bool,
    #[orr::constant]
    pub work: u64,
    #[orr::constant]
    pub len: u64,
    gate: orr::Gate,
    /// Cycles of work left on the jobs taken so far.
    left: u64,
    pub state: u64,
}

#[orr::module_impl]
impl Worker {
    #[orr::task(triggers = (clock, jobs))]
    #[allow(clippy::too_many_arguments)]
    fn cycle(
        ctx: &mut orr::TaskCtx,
        gating: &'constant bool,
        work: &'constant u64,
        len: &'constant u64,
        gate: &mut orr::Gate,
        left: &mut u64,
        state: &mut u64,
    ) -> Result<(), orr::SimError> {
        let (clock, job) = ctx.triggered_triggers();
        // A cycle of work on the jobs taken before now.
        if clock.is_some() && *left > 0 {
            *state = mix(*state ^ *left, *work);
            *left -= 1;
        }
        // A job that arrives now is worked on from the next cycle.
        if let Some(job) = job {
            *state ^= *job;
            *left += *len;
        }
        if *gating {
            if *left == 0 {
                gate.close(); // until the next job
            } else if !gate.is_open() {
                gate.open(); // a job came while it was closed
            }
        }
        Ok(())
    }
}
```

task 里的先后，保证两种写法的结果一样：

- 这一拍的活，只做在此之前收下的那些；
- 这一拍送到的活，在做完这一拍的活之后才收下。

所以一份活送到一个关着门的工人那里，从下一个沿开始做；门从不关的工人，也是从那个沿开始做。

派活的和驱动程序：

```rust
#[orr::module]
pub struct Office {
    #[orr::param("workers", default = 64usize)]
    n: usize,
    #[orr::param("a worker gets a job every this many cycles", default = 100u64)]
    every: u64,
    #[orr::param("cycles a job takes", default = 10u64)]
    len: u64,
    #[orr::param("mixer rounds in a cycle of a job", default = 100u64)]
    work: u64,
    #[orr::param("whether a worker closes its gate when it has no job", default = true)]
    gating: bool,
    #[orr::trigger]
    clock: orr::Clock,
    #[orr::submodule]
    workers: [Worker; orr::DynShape(n)],
    #[orr::trigger(min_delay = orr::Time::ps(PERIOD_PS))]
    jobs: [orr::EventSource<u64>; orr::DynShape(n)],
}

#[orr::module_impl]
impl Office {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, worker) in module.workers.iter().enumerate() {
            ctx.connect(module.clock, worker.clock)?;
            ctx.connect(module.jobs[i], worker.jobs)?;
            ctx.set_const(worker.gating, *module.gating)?;
            ctx.set_const(worker.work, *module.work)?;
            ctx.set_const(worker.len, *module.len)?;
        }
        Ok(())
    }

    /// Hands worker `i` a job, arriving at the next edge, every `every`
    /// cycles, the workers taking turns.
    #[orr::task(triggers = (clock,), emits = (jobs,))]
    fn dispatch(ctx: &mut orr::TaskCtx) -> Result<(), orr::SimError> {
        let edge = (ctx.now().ps() / u128::from(PERIOD_PS)) as u64;
        let every = *ctx.module().every;
        for (i, mut job) in ctx.emit_triggers().0.iter().enumerate() {
            if (edge + 7 * i as u64) % every == 0 {
                job.emit(edge, orr::Time::ps(PERIOD_PS))?;
            }
        }
        Ok(())
    }
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    // Busy a tenth of the time, and all the time.
    for (len, cycles) in [(10u64, 100_000u64), (100, 20_000)] {
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        let mut cases = Vec::new();
        for gating in [false, true] {
            let name = format!("ch2-len{len}-gating{gating}");
            let file = params(&name, &format!("[top]\nlen = {len}\ngating = {gating}\n"))?;
            cases.push(Case::new(
                format!("busy {}%, gating {gating}", len),
                orr::SimConfig::default(),
                move |config| Ok(orr::Sim::new(orr::SimPlan::<Office>::new(Some(file.clone()), None)?, config)?),
                move |sim| {
                    sim.run_until(end)?;
                    let mut sum = 0;
                    for worker in sim.top().workers.iter() {
                        sum = mix(sum ^ *sim.state(worker.state)?, 1);
                    }
                    Ok(sum)
                },
            ));
        }
        let results = compare(&threads, 3, &cases)?;
        // Closing the gate must not change the result.
        assert_eq!(results[0].1, results[1].1, "gating changed the result");
    }
    Ok(())
}
```

驱动程序在两种负载下把两种写法都量一遍：

- 工人一成的时间在忙（每 100 拍一份 10 拍的活）；
- 工人一直在忙（每 100 拍一份 100 拍的活）。

关门改变了结果，它就报错。

## 对三条指标的影响

运行的秒数（粗量）。1、2、4、8 线程的执行单元数是 2、2、4、8（工人，加上派活的）：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 |
|---|---|---|---|---|---|
| 一成时间在忙，门从不关 | 0.263 | 0.293 | 0.139 | 0.089 | 2.96× |
| 一成时间在忙，闲了就关 | 0.179 | 0.207 | 0.115 | 0.079 | 2.26× |
| 一直在忙，门从不关 | 0.254 | 0.261 | 0.096 | 0.0486 | 5.22× |
| 一直在忙，闲了就关 | 0.254 | 0.260 | 0.095 | 0.0481 | 5.28× |

- **一成时间在忙**：关门**更好**：
  - 1 线程快到 1.47 倍；
  - 8 线程快到 1.12 倍；
  - 没有哪个线程数变慢。
  
  8 线程上快，但加速比从 3.0 降到 2.3。在这个例子里，关门在 1 线程上省了三分之一的时间，在 8 线程上省了九分之一。多线程时，只要这一拍有任何东西在跑，每个执行单元都要花时间，哪怕它里面工人的门全关着。所以门省下的，1 线程的时间多、8 线程的时间少。这是应该的：比时间，不比加速比。
- **一直在忙**：在这个例子里，关门不省也不费（每个线程数都在抖动以内）。这里工人接了第一份活以后，门就再没关过：上一份做完的那一拍，下一份正好到。常关常开的门是要花钱的，见[什么时候不该这样写](#when-not-to)。
- **读**：要多记住两样：
  - 门在什么条件下关（`left == 0`，一行）；
  - 什么把它打开（送活的事件，一行）。

  条件说的是硬件上的事（"没活了"）时，读起来就是单元行为的一部分。

## GPU 模型里

随 Orrery 附带的 GPU 模型，几乎每个单元都会关门。比如一个 L2 bank 有两个 task：

- `reply` 这一步把队列头上的回复送进回复网络；把队列送空了，就关门。
- `cache` 这一步往这个队列里推回复，每推一个也叫醒 `reply`。模型的做法是：把队列的发送端包进一个小类型，它的 `push` 顺带叫醒。

两步都不写 `open()`：关着的那一步只靠叫醒打开。

## 什么时候不该这样写 {#when-not-to}

- **几乎一直在忙的单元**，关门没得省，还可能赔。门省下的是它跳过的调用；关一次门、再开一次，约花一到两次调用的钱。本章的工人，每 *k* 拍来一份一拍的活（每关一次门省 *k* − 2 次调用），和门一直开着比，1 线程上：*k* = 2、3 时关门略赔，4 时略赚，6 起明显赚。一个路由器网格，缓冲空了就关门，在忙的流量上 1 线程比门一直开着慢 13–16%。
- **一直开着的门也不是白给的。** 同样这些工人，1 线程上门一直开着比完全没有门多花 4% 到 28% 的时间，越闲多得越多；那些路由器是 4–13%。和没有门比，关门到 *k* = 6 才打平；工人十拍里闲九拍时，也只省了不到一成。8 线程上三种写法多半相差不到一成。所以只有单元闲得多，门才值。有门、没门都量；不划算的地方就不要门：触发源上不写 `gate`。这样也少一个要读的条件。没有门不是一个参数能切换的：没有门的模块是另一个模块。要量它，就把模块抄一份，去掉触发源上的 `gate` 和门的代码，两份轮流跑（上面的量法对工人就是这么做的）。
- **知道下一份活什么时候来的单元**（固定的延迟、一个定时器）应当关到那时，而不是等人来叫（[第 3 章](./due)）。这样别的单元什么都不用送。
- **门关着时送到门后触发源的事件会被丢掉**（并计数）。送活来的那个触发源不能在门后，或者送的一方要先叫醒它。测试时用 `SimConfig { fail_on_drop: true, .. }`，丢了事件就让运行报错。

## 核对单

- 每扇门在什么条件下关，能用一句话说出来吗？
- 活能送到这个单元的每一条路，都会打开它的门吗？（不在门后的事件，或每个送的一方都叫醒它：见第 4 章的表。）
- 门从不关时，结果一样吗？在同样的拍上吗？测试开了 `fail_on_drop` 吗？
- 四个线程数都量了有门和完全没有门两种吗（不只是门一直开着）？
