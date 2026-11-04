# 运行仿真

## 从 plan 到仿真

```rust
use orrery_lang as orr;

let plan = orr::SimPlan::<Top>::new(Some("params.toml".into()), None)?;
let mut sim = orr::Sim::new(plan, orr::SimConfig::default())?;
```

- `SimPlan::new(params, snapshot)` 展开模型并检查它的连接。两个参数都是可选的路径：没有参数文件时，每个参数保持默认值；给了快照时，仿真从快照开始（[快照](./snapshots)）。
- `Sim::new(plan, config)` 加载仿真。它推出模型里各个 task 的先后，顺序有歧义的模型会被拒绝（[报错](./errors#when-the-simulation-loads)）。
  `Sim::with(plan, config, event_log, |top, ports| ..)` 多两个参数：[事件日志](#event-log)的 sink（没有就给 `None`），和一个闭包，为带[外部端口](./external-ports)的模型绑定端口。

## 配置 {#configuration}

`orr::SimConfig`：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `workers_per_group` | 1 | 执行 task 的 worker 数（[worker](#workers)）；0 会让加载失败 |
| `groups` | 1 | worker 的组数；目前只有 1 能加载 |
| `checks` | `false` | 记下 task 访问状态的先后，供 `sim.check_trace()` 检查；慢，用于测试；只能用一个 worker |
| `delta_limit` | 30 | 同一个物理时间上 delta 周期数的上限，最大 62（[delta 周期](./time#delta-cycles)） |
| `event_capacity` | 2²⁰ | 一个实例的一个事件源最多能有多少个在途事件，即已经发出、还没到达的 |
| `fail_on_drop` | `false` | 门控关闭时到达触发端口的事件是错误，而不是被丢弃并计数（[时钟门控](./time#clock-gates)） |
| `snapshots` | `None` | 何时、在哪里拍[快照](./snapshots)；只能用一个 worker |
| `reports` | `Warnings` | 仿真往 stderr 打印什么：`Warnings` 打印加载时的警告，以及丢弃仿真时关着的门控丢掉了多少事件；`Off` 什么都不打印；`All` 另外打印加载的报告和每次运行的报告 |
| `cpus` | `None` | 多个 worker 时，仿真可以从哪些 CPU 里取核；`None` 表示进程能用的任何 CPU |
| `idle_spin` | 1 ms | 多个 worker 时，两次运行之间仿真的线程占着核空转多久才睡眠 |
| `slots` | 16 | 多个 worker 时的一个调优设置：2 到 1024 之间的 2 的幂 |

这些字段都不改变结果，只有 `delta_limit`、`event_capacity` 和 `fail_on_drop` 例外，它们以确定的方式改变结果：给定的值在任何机器上、不论模型用几个 worker 运行，都得到同样的结果。

以后还会增加字段。用 `..orr::SimConfig::default()` 写的配置会一直能编译：

```rust
let config = orr::SimConfig {
    checks: true,
    delta_limit: 50,
    ..orr::SimConfig::default()
};
```

## worker {#workers}

一个 worker 时（默认），仿真在调用 `run_until` 或 `step` 的线程上运行 task。两个或更多时，它在自己的这么多个线程上运行 task，并且比 worker 数多占一个物理核，全都在同一个 L3 缓存域里：

```rust
let config = orr::SimConfig {
    workers_per_group: 4,
    ..orr::SimConfig::default()
};
```

- *W* 个 worker 要占一个至少有 *W* + 1 个物理核的域，这些核全都在 `cpus` 允许的 CPU 里。域比这小的机器上加载会失败（`LoadError::Config`，字段 `workers_per_group`）；`cpus` 里没有一个核数足够的完整的域，加载也会失败（字段 `cpus`）。在每个域 8 个核的机器上，最多就是 7 个 worker。
- 仿真一直占着它的域，直到被丢弃。同一个进程里另一个同样用多个 worker 的仿真会占另一个域，没有剩下的域就加载失败（字段 `cpus`）。

用多个 worker 时，模型给出的和一个 worker 时相同：同样的状态、结束方式和错误。不过目前多个 worker 只能运行由时钟、[门控](./time#clock-gates)、状态、`Fifo` 这样的组件和[外部端口](./external-ports)组成的模型。下面这些在两个或更多 worker 上会让加载失败（`LoadError::Config`，字段 `parallel`），要用一个 worker 运行：

- 发出事件、或者唤醒 task 的 task（[唤醒](./time#wakes)）；
- `checks`；
- 拍快照，或者从快照恢复；
- 事件日志。

## 运行

`sim.run_until(time)` 运行到 `time` 为止（含 `time`）的每个时刻点，包括它们的全部 delta 周期，并返回运行是怎样结束的：

```rust
match sim.run_until(orr::Time::us(10))? {
    orr::RunEnd::Reached(time) => println!("reached {time}"),
    orr::RunEnd::Finished(finished) => println!("stopped at {}", finished.at),
}
```

运行可以分多次：先 `run_until(1 µs)`，再 `run_until(2 µs)`，第二次从第一次停下的地方继续，结果与一次跑到 2 µs 相同。一次运行最多走到约 288 s 的仿真时间：`run_until` 给的时间超过它，或者不是物理时间，就返回 `RunError::Request`，什么也不运行，仿真还可以继续（[时间](./time#delta-cycles)）。

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

`state` 读一个值；`state_mut` 写一个值，宿主也用它访问 store 和外部端口的源对象。[时钟门控](./time#clock-gates)用 `sim.gate_mut` 打开和关闭。宿主写入的值从下一次运行的第一个时刻点起被看到。

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

*L* 是模型的**免费延迟**，由 Orrery 在加载模型时算出。它取决于模型，从不取决于配置，`sim.stop_delay()` 会报告它。这里它是 0；[GPGPU-Sim 模型](../examples/gpgpu-sim)里它是 425 ps。Orrery 算不出免费延迟的模型，它也是 0。它不能设置。

有 stop 延迟时，一次运行结束和仿真结束可能不是同一次调用：

- `run_until` 跑到终点 *T* + *L* 时返回 `Finished`；在那之前的 `run_until` 照常返回 `Reached`。从此以后，任何合法的 `run_until`（不论给的是范围之内的哪个时间）和任何 `step` 都返回同一个 `Finished`；不成立的请求，例如给的是 `Time::DELTA`，仍然返回 `Request`（[报错](./errors#while-it-runs)）。
- `step` 从不越过它运行的那个时刻点。*L* = 0 时，运行 *T* 的那次 `step` 返回 `Finished`。*L* > 0 时，`step` 一个一个地运行终点之前的时刻点，然后返回 `Idle`，`next_timepoint()` 是 `None`；之后越过终点的 `run_until` 返回 `Finished`。

`Stop::Error(e)` 和来自 `SimError` 的 `Err(e)` 则让这次运行以错误结束；`?` 会做转换。终点之前的错误会被报告，在它之前返回的 stop 仍然留在 `finished()` 里；只会发生在终点之后的事情不会被报告（[报错](./errors#while-it-runs)）。

## 事件日志 {#event-log}

仿真可以把模型发出的每个事件按到达的时间排好，在运行中交给一个 **sink**。sink 在仿真加载时通过 `Sim::with` 给出。`orr::EventLogFile` 是把日志写进文件的 sink：

```rust
let log = orr::EventLogFile::create("events.log")?;
let mut sim = orr::Sim::with(plan, orr::SimConfig::default(), Some(Box::new(log)), |_, _| Ok(()))?;
sim.run_until(orr::Time::ns(100))?;
sim.close_event_log()?;
```

- 目前带事件日志的仿真用一个 worker 运行。
- 每次运行把到停下的地方为止的事件交给 sink，并让它 flush。`close_event_log` 交出剩下的，即已经发出、到达得更晚的事件，然后结束日志；此后再运行会失败。丢弃仿真也会结束日志，但错误就丢了。
- sink 返回的错误让这次运行失败，以后的运行也都失败。
- 同一个时刻点上的事件有固定的先后，所以一个模型每次运行写出的日志文件都相同，两个日志文件可以逐字节比较。

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

自己的 sink 实现 `orr::EventSink`：`event` 接收每个事件，`flush`、`end` 按上面说的时机调用，`begin` 在仿真加载时调用一次。它必须是 `Send`。`orr::EventLogFile::new(writer)` 把同样的文件格式写进任意 writer，例如一个压缩的 writer。

[从快照恢复](./snapshots#restoring)的仿真也这样给 sink，从快照处起记下事件，和原来那次运行记下的一样，只是快照之前发往一个不触发任何 task 的事件源的事件不在里面：

```rust
let plan = orr::SimPlan::<Top>::new(params, Some(latest))?;
let log = orr::EventLogFile::create("restored.log")?;
let mut sim = orr::Sim::with(plan, config, Some(Box::new(log)), |_, _| Ok(()))?;
```

sink 在调用 `run_until` 或 `step` 的线程上调用。

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
- `--windows <span>`（`50ns`、`1us`、`1500fs`）每个有事件的时间窗口打印一行，`<start> <events> <hash>`，用来比较大得存不下的日志：两份日志的行不同的地方，事件也不同。再加 `--by column`，每个窗口的行按字段分开（`%Producer.done`）。
- `--fingerprint <log>` 打印日志所属模型的指纹；给了第二份日志时，两份属于不同的模型就失败。

退出码：成功是 0；读不了文件或指纹不同是 1；参数不对是 2，即没给日志、多给了一个，或者 `--windows` 的跨度不是一个大于 0 的时间。不认识的选项会被当成文件名。它读这一版 Orrery 写的日志，拒绝更早版本的。

## 运行之后

- 一旦有 task 停止，`sim.finished()` 就返回这个 stop。
- 开启 `checks` 时，`sim.check_trace()` 验证访问同一个状态、其中至少一个是写的每两次访问都按串行顺序运行，否则返回违反之处。没开 `checks` 时，它返回一个说明这一点的错误。
- `sim.load_report().warnings` 里是加载时的警告，一条一行，和 stderr 上 `warning: ` 后面的一样（[报错](./errors#warnings)）。
