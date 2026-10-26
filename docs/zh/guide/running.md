# 运行仿真

## 从 plan 到仿真

```rust
use orrery_lang as orr;

let plan = orr::SimPlan::<Top>::new(Some("params.toml".into()), None)?;
let mut sim = plan.into_sim(orr::SimConfig::default())?;
```

- `SimPlan::new(params, snapshot)` 展开并检查模型。两个参数都是可选的路径：没有参数文件时，每个参数保持默认值；给了快照时，仿真从快照开始（[快照](./snapshots)）。
- `into_sim(config)` 构建仿真。`into_sim_with(config, event_log, |top, ports| ..)`
  多两个参数：[事件日志](#event-log)的 sink（没有就给 `None`），和一个闭包，为带[外部端口](./external-ports)的模型绑定端口。

## 配置 {#configuration}

`orr::SimConfig`：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `workers` | 1 | 执行 task 的线程数；放置策略可能用得更少；多于一个、又装了[事件日志](#event-log)时，另有一个线程写日志（[线程](#threads)） |
| `overlap` | `true` | 在模型允许的地方，前面的时刻点还在运行时就开始后面的；`false` 一次只运行一个时刻点，用于测试 |
| `checks` | `false` | 记下 task 访问状态的先后，供 `sim.check_trace()` 检查；慢，用于测试 |
| `delta_limit` | 30 | 同一个物理时间上 delta 周期数的上限，最大 65,534；设得越大，一次运行在时间上能走得越短（[delta 周期](./time#delta-cycles)） |
| `stop_delay` | `None` | stop 之后仿真还运行多久；`None` 表示模型的免费延迟（[停止](#stopping)） |
| `event_capacity` | 2²⁰ | 一个实例的一个 task 在一个事件源上最多能有多少个在途事件，即已经发出、还没到达的 |
| `fail_on_drop` | `false` | 门控关闭时到达触发端口的事件是错误，而不是被丢弃并计数（[时钟门控](./time#clock-gates)） |
| `snapshots` | `None` | 何时、在哪里拍[快照](./snapshots) |
| `skip_closed` | `Some(64)` | 连续至少这么多个时钟沿上所有受门控的时钟都关着、也没有别的事发生时，一次越过它们；`None` 逐个越过 |
| `lead_limit` | `Some(256)` | 模型的各部分最多能比它们发事件去的部分领先多远，由此限制在途事件的数量和它们占的内存；至少为 1，`None` 不设上限 |
| `wait` | `orr::exec::Adaptive` | 空闲的 worker 怎样等待工作：`Adaptive` 先自旋、再让出、最后睡眠；`AlwaysSpin` 适合仿真独占的机器，`AlwaysPark` 适合 worker 多于空闲核的情况 |
| `placement` | `orr::plan::Measured` | 模型怎样分布到各个 worker 上；`Measured` 测量运行来决定 |
| `pin` | `orr::exec::NoPin` | worker 在哪些 CPU 上运行：`NoPin` 交给操作系统；`CacheDomain` 把它们限制在和 worker 数一样多的物理核上，离调用线程最近的优先 |
| `topology` | `None` | `pin` 从中挑选的 CPU；`None` 读取本机的 |
| `steal` | `orr::exec::DomainSteal` | 空闲的 worker 从哪些别的 worker 那里取工作：`DomainSteal` 取与它共享缓存的，`FlatSteal` 取任意的，`NoSteal` 不取 |
| `job_log` | `None` | 每个 worker 把自己做的工作记录写进这个目录，用于性能分析 |
| `account` | `false` | 记下 worker 的 CPU 时间花在哪里，用 `sim.account()` 读取 |

这些字段都不改变结果，只有 `stop_delay`、`delta_limit`、`event_capacity` 和 `fail_on_drop` 例外，它们以确定的方式改变结果：给定的值在任何机器、任何 worker 数下都得到同样的结果。

```rust
let config = orr::SimConfig {
    checks: true,
    delta_limit: 100,
    ..orr::SimConfig::default()
};
```

策略（`wait`、`placement`、`pin`、`steal`）是共享的值：

```rust
use std::sync::Arc;

let config = orr::SimConfig {
    workers: 4,
    wait: Arc::new(orr::exec::AlwaysSpin),
    pin: Arc::new(orr::exec::CacheDomain),
    ..orr::SimConfig::default()
};
```

## 运行

`sim.run_until(time)` 运行到 `time` 为止（含 `time`）的每个时刻点，包括它们的全部 delta 周期，并返回运行是怎样结束的：

```rust
match sim.run_until(orr::Time::us(10))? {
    orr::RunEnd::Reached(time) => println!("reached {time}"),
    orr::RunEnd::Finished(finished) => println!("stopped at {}", finished.at),
}
```

运行可以分多次：先 `run_until(1 µs)`，再 `run_until(2 µs)`，第二次从第一次停下的地方继续，结果与一次跑到 2 µs 相同。

`sim.step()` 恰好运行下一个时刻点，返回 `StepEnd::Executed(timepoint)`、
`StepEnd::Finished(..)`，或者在没有东西可运行时返回 `StepEnd::Idle`。
`sim.next_timepoint()` 给出下一个会是哪个时刻点。

时刻点是有 task 运行、或者有事件在关着的[时钟门控](./time#clock-gates)上被丢弃的时刻。受门控的时钟全都关着的时钟沿不是时刻点，到达一个不触发任何 task 的事件源的事件也不构成时刻点：`step` 会跳过它们，`next_timepoint` 也不报告它们。

`sim.now()` 是最后运行的那个时刻点，第一个时刻点之前是 `None`。它不表示仿真跑到了哪里：`run_until(1 µs)` 之后，`now()` 是 1 µs 及之前最后一个有事发生的时刻点。

## 读写状态

两次运行之间，宿主通过 `sim.top()`（即实例树）里的句柄读取状态：

```rust
let total = sim.top().workers[0].sink.total;
println!("{}", sim.state(total)?);
*sim.state_mut(total)? = 0;
```

`state` 读一个值；`state_mut` 写一个值，宿主也用它访问 store、外部端口的源对象和[时钟门控](./time#clock-gates)。宿主写入的值从下一次运行的第一个时刻点起被看到。

## 停止 {#stopping}

可能结束仿真的 task 返回 `Result<(), orr::Stop>`：

```rust
#[orr::module]
pub struct Countdown {
    #[orr::param("ticks before stopping", default = 5u64)]
    limit: u64,

    #[orr::trigger]
    clock: orr::Clock,

    ticks: u64,
}

#[orr::module_impl]
impl Countdown {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))
    }

    #[orr::task(triggers = (clock,))]
    fn count(ctx: &mut orr::TaskCtx, ticks: &mut u64) -> Result<(), orr::Stop> {
        *ticks += 1;
        if *ticks == *ctx.module().limit {
            return Err(orr::Stop::finish(format!("counted to {ticks}")));
        }
        Ok(())
    }
}
```

```rust
match sim.run_until(orr::Time::ns(100))? {
    orr::RunEnd::Finished(finished) => {
        for finish in &finished.by {
            println!("{} at {}: {}", finish.path, finished.at, finish.message);
        }
        println!("ended at {}", finished.end);
    }
    orr::RunEnd::Reached(time) => println!("reached {time}"),
}
```

```text
top at 4ns: counted to 5
ended at 4ns
```

在 *T* 返回的 stop 让仿真在 *T* + *L* 结束，*L* 是 stop 延迟：到这个时间为止的一切都会运行，之后的都不运行。`finished` 记录 stop 从哪里来（`at`、`by`）、仿真在哪里结束（`end`），以及其间返回的其他 stop（`later`）。

默认的 *L* 是模型的**免费延迟**，由 Orrery 根据模型算出，让停止的代价尽量小。它取决于模型，从不取决于配置，`sim.stop_delay()` 会报告它。这里没有任何东西依赖计数器，所以它是 0。`stop_delay` 可以设成任何别的物理时间，比免费延迟短或长都行：更短的值，例如 `Some(orr::Time::ps(0))`，结果同样精确，但仿真可能变慢。少数模型的免费延迟 Orrery 算不出来，这时构建仿真会失败，要求给出 `stop_delay`。

有 stop 延迟时，一次运行结束和仿真结束可能不是同一次调用：

- `run_until` 跑到终点 *T* + *L* 时返回 `Finished`；在那之前的 `run_until` 照常返回 `Reached`。从此以后，任何 `run_until`（不论给的时间是多少）和任何 `step` 都返回同一个 `Finished`。
- `step` 从不越过它运行的那个时刻点。*L* = 0 时，运行 *T* 的那次 `step` 返回 `Finished`。*L* > 0 时，`step` 一个一个地运行终点之前的时刻点，然后返回 `Idle`，`next_timepoint()` 是 `None`；之后越过终点的 `run_until` 返回 `Finished`。

`Stop::Error(e)` 和来自 `SimError` 的 `Err(e)` 则让这次运行以错误结束；`?` 会做转换。终点之前的错误会被报告，在它之前返回的 stop 仍然留在 `finished()` 里；只会发生在终点之后的事情不会被报告（[报错](./errors#while-it-runs)）。

## 事件日志 {#event-log}

仿真可以把模型发出的每个事件按到达的时间排好，在运行中交给一个 **sink**。`orr::EventLogFile` 是把日志写进文件的 sink：

```rust
let mut sim = plan.into_sim(orr::SimConfig::default())?;
sim.log_events_to(orr::EventLogFile::create("events.log")?)?;
sim.run_until(orr::Time::ns(100))?;
sim.close_event_log()?;
```

- sink 在第一次运行以前装，只装一次。
- 每次运行把到停下的地方为止的事件交给 sink，并让它 flush。`close_event_log` 交出剩下的，即已经发出、到达得更晚的事件，然后结束日志；此后再运行会失败。丢弃仿真也会结束日志，但错误就丢了。
- sink 返回的错误让这次运行失败，以后的运行也都失败。
- 同一个时刻点上的事件有固定的先后，所以一个模型在任何 worker 数下写出的日志文件都相同，两份日志可以逐字节比较。

`orr::EventLogReader` 把日志文件逐个事件读回来：时间、位置和载荷的字节。

```rust
let mut log = orr::EventLogReader::open("events.log")?;
while let Some(event) = log.next() {
    let (at, loc, payload) = event?;
    println!("{at} {} {payload:02x?}", log.path(loc));
}
```

```text
0ps+1 top.workers[0].producer.done [01, 00, 00, 00, 00, 00, 00, 00]
0ps+1 top.workers[1].producer.done [01, 00, 00, 00, 00, 00, 00, 00]
10ns+1 top.workers[0].producer.done [02, 00, 00, 00, 00, 00, 00, 00]
```

没有结束的日志，或者被截断的日志，先给出截断处之前的事件，然后是一个 `UnexpectedEof` 类的错误。

自己的 sink 实现 `orr::EventSink`：`event` 接收每个事件，`flush`、`end` 按上面说的时机调用，`begin` 在装上时调用一次。它必须是 `Send`。`orr::EventLogFile::new(writer)` 把同样的文件格式写进任意 writer，例如一个压缩的 writer。

[从快照恢复](./snapshots#restoring)的仿真在构建时通过 `into_sim_with` 装 sink，从快照处起记下事件，和原来那次运行记下的一样；对它调用 `log_events_to` 会失败：

```rust
let plan = orr::SimPlan::<Top>::new(params, Some(latest))?;
let log = orr::EventLogFile::create("restored.log")?;
let mut sim = plan.into_sim_with(config, Some(Box::new(log)), |_, _| Ok(()))?;
```

### 线程 {#threads}

一个 worker 时，sink 在调用 `run_until` 的线程上调用。多个 worker 时，仿真另起一个线程来调用 sink：8 个 worker 加 sink 共 9 个线程。给这个线程一个单独的核，和每个 worker 一样。核不够时（写日志的线程和某个 worker 共用一个核，或者这些核上还有别的负载），仿真照样能结束，结果也相同，但可能变慢。

### orr-events

`orr-events` 命令把日志文件打印成文本，一个事件一行，载荷用十六进制。在 Orrery 的 checkout 里运行：

```sh
cargo run --release -p orrery-lang --bin orr-events -- --paths events.log
```

```text
0ps+1 top.workers[0].producer.done 0100000000000000
0ps+1 top.workers[1].producer.done 0100000000000000
10ns+1 top.workers[0].producer.done 0200000000000000
```

- 不加 `--paths` 时，位置用一个更短的名字打印：`%Producer.done[0]`。
- `--windows <span>`（`50ns`、`1us`）每个有事件的时间窗口打印一行，`<start> <events> <hash>`，用来比较大得存不下的日志：两份日志的行不同的地方，事件也不同。再加 `--by column`，每个窗口的行按字段分开（`%Producer.done`）。
- `--fingerprint <log>` 打印日志所属模型的指纹；给了第二份日志时，两份属于不同的模型就失败。

退出码：成功是 0；读不了文件或指纹不同是 1；参数不对是 2。

## 运行之后

- 一旦有 task 停止，`sim.finished()` 就返回这个 stop。
- 开启 `checks` 时，`sim.check_trace()` 验证访问同一份状态、其中至少一个是写的每两次访问都按串行顺序运行，否则返回违反之处。
