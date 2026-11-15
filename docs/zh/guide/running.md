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

`orr::SimConfig` 里用户要设的字段：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `workers_per_group` | 1 | 运行仿真的线程数：1 是在调用 `run_until` 的线程上运行；*N* ≥ 2 是一个分派工作的线程，加 *N* − 1 个执行 task 的 worker，所以 2 不会比 1 快（[线程](#threads)）；0 会让加载失败 |
| `groups` | 1 | 线程的组数；目前只有 1 能加载 |
| `checks` | `false` | 记下 task 访问状态的先后，供 `sim.check_trace()` 检查；用于测试：它记下的东西随运行长度增长，占的内存和时间也随之增长 |
| `delta_limit` | 30 | 同一个物理时间上 delta 周期数的上限，最大 62；超过就是错误（[delta 周期](./time#delta-cycles)） |
| `snapshots` | `None` | 何时、在哪里拍[快照](./snapshots) |
| `event_capacity` | 2²⁰ | 一个实例的一个事件源最多能有多少个在途事件，即已经发出、还没到达的；超过就是错误 |
| `fail_on_drop` | `false` | 门控关闭时到达触发端口的事件让运行失败，而不是被丢弃并计数（[时钟门控](./time#clock-gates)） |
| `reports` | `orr::Reports::Warnings` | 仿真往 stderr 打印什么：`Warnings` 打印加载时的[警告](./errors#warnings)，以及丢弃仿真时关着的门控丢掉了多少事件；`Off` 什么都不打印；`All` 再加上装载报告和每次停下时的报告 |
| `cut` | `orr::CutPolicy::Runs` | 多线程运行怎么把同一种模块的实例分进执行单元：`Runs` 按编号连着分，`Dealt` 发牌式地一个一个分（[什么时候发牌式地分](#dealt)） |
| `cpus` | `None` | 多线程运行可以从哪些 CPU 里挑核；`None` 表示进程能用的任何 CPU（[线程](#threads)） |
| `host_spin` | 1 ms | 调 `run_until` 的线程等一次多线程运行跑完时，先空转多久再睡；`0` 立刻睡。进程能用的 CPU 只有线程数那么多时，它和 `idle_spin` 都设 `0`（[线程](#threads)） |
| `idle_spin` | 1 ms | 多线程运行自己的线程（分派工作的线程、worker、写事件日志的线程）在两次调用之间先空转多久再睡；`0` 立刻睡。和 `host_spin` 一起设 `0`（[线程](#threads)） |
| `slots` | 16 | 保持默认值 |

只要运行不超过 `delta_limit` 和 `event_capacity`，这些字段都不改变结果；`fail_on_drop` 例外：开了它，本来会丢弃事件的运行就在那里失败。这三个字段改变结果的方式，在任何机器上、任何线程数下都一样。`SimConfig` 还有两个字段，`timings` 和 `pause`，保持默认值即可。

以后还会增加字段。用 `..orr::SimConfig::default()` 写的配置会一直能编译：

```rust
let config = orr::SimConfig {
    workers_per_group: 4,
    delta_limit: 50,
    ..orr::SimConfig::default()
};
```

## 线程 {#threads}

`workers_per_group: N` 让仿真跑在 *N* 个线程上。*N* = 1 时，调用 `run_until` 的线程运行所有 task。*N* ≥ 2 时，一个线程分派工作，*N* − 1 个 worker 运行 task，调用 `run_until` 的线程只发起运行并等它结束。所以 2 个线程时运行 task 的只有一个 worker，和 1 个线程一样，还要付多线程运行的代价：从 3 个线程起 task 才并行运行。开了[事件日志](#event-log)的多线程运行另有一个写日志的线程。线程数从不改变结果。

**核。** *N* ≥ 2 的运行把每个线程固定在一个单独的物理核上，这些核都在同一个缓存域里（共享同一块 L3 缓存的那组核）；worker 等活时忙等，不睡眠。给它独占的核：同一些核上还有别的程序时，它会明显变慢，时间也不稳定。`cpus: Some(vec![..])` 限定它可以从哪些 CPU 里挑核；同一个物理核的两个硬件线程算一个核。单线程运行不固定在核上，也不读 `cpus`。允许的 CPU 里没有哪个缓存域凑得出 *N* 个空闲的物理核时，构建仿真失败，并说明是哪一种情况。

**调 `run_until` 的线程**不算在 *N* 个里面。多线程运行跑着的时候它在等：先空转 `host_spin`（缺省 1 ms），再睡；空转要占一个 CPU。两次调用之间正好反过来：运行自己的线程先空转 `idle_spin`（也是 1 ms）再睡，这时这个线程在做程序在两次调用之间要做的事。进程能用的 CPU 只有线程数那么多时（比如 8 个线程，进程只能用 8 个 CPU、每核一个；或者 2 个线程 2 个 CPU、4 个线程 4 个），这个线程没有自己的 CPU：哪一边空转，就占了另一边要用的 CPU，运行会变慢。每 1000 拍调一次 `run_until`，最多慢到四倍；活一小阵一小阵来的模型，慢到二十倍上下，整次运行只调一次也一样。这时两个都设成零，不管每次调用跑多长：

```rust
let config = orr::SimConfig {
    workers_per_group: 8,
    host_spin: std::time::Duration::ZERO,
    idle_spin: std::time::Duration::ZERO,
    ..orr::SimConfig::default()
};
```

这样设了以后，运行都回到 CPU 有富余时的用时，差几个百分点，活一小阵一小阵来的那个模型差一成；每几拍调一次的，仍然最多慢两成。只把 `host_spin` 设 0，这种最多慢到 1.4 倍；两个都设，没有哪次比只设一个慢出多跑几遍的抖动。数的是进程能用的 CPU：某个核的第二个硬件线程空着、可不在进程能用的里面，不算。只用 `cpus` 限了核、进程别的 CPU 还能用的，不用改。CPU 有富余，就两个都别动：那时立刻睡什么也省不下，睡了的线程还得叫醒。

**一个仿真在被丢掉之前一直占着它的核**，不只是在运行的时候。同一个进程里一个接一个地跑几个多线程的仿真时，先让前一个出作用域（或 `drop(sim)`），再建下一个；否则前一个的核还被占着，建下一个会像上面那样失败。把下一个仿真绑定到同一个名字上，并不会丢掉前一个。每次运行放进自己的块里：

```rust
let config = orr::SimConfig { workers_per_group: 4, ..orr::SimConfig::default() };
{
    let plan = orr::SimPlan::<Top>::new(Some("small.toml".into()), None)?;
    let mut sim = orr::Sim::new(plan, config.clone())?;
    sim.run_until(orr::Time::us(10))?;
} // 第一个仿真在这里被丢掉，它的核又空出来了
{
    let plan = orr::SimPlan::<Top>::new(Some("large.toml".into()), None)?;
    let mut sim = orr::Sim::new(plan, config)?;
    sim.run_until(orr::Time::us(10))?;
}
```

**几个进程。**别的进程里的仿真看不到这个仿真占着的核，可能占到同一些。要同时跑几个、每个都用多个线程，就给每个各自的 `cpus`，彼此没有相同的 CPU（`lscpu -e` 列出每个 CPU 的 L3 缓存，在 `L1d:L1i:L2:L3` 一列的最后）：

```rust
// 假设 CPU 0-7 和 8-15 是两个 L3 域：一个进程里写
let config = orr::SimConfig {
    workers_per_group: 4,
    cpus: Some((0..8).collect()),
    ..orr::SimConfig::default()
};
// 同时跑的另一个进程里写 cpus: Some((8..16).collect())
```

**多线程运行目前还不收的。**模型里有下面两样之一，在 2 个或更多线程上加载会失败（`LoadError::Config`，字段 `parallel`），单线程能加载：一个 task [唤醒](./time#wakes)的[时钟门控](./time#clock-gates)，又在它所属的执行单元以外的单元里被读；两个 task 可能在同一个时刻点写同一个状态位置，而模型没有给这两者定先后。

**有多少能并行。** 构建仿真时，Orrery 把模型分成若干个**执行单元**（execution unit）：总在一起、同一时刻只在一个 worker 上运行的一组调用。task 越多、同一种模块的实例越多（至少和 worker 一样多，才能让每个 worker 都有活），分出的执行单元就越多。以下几种调用会进同一个执行单元：只有一个实例的模块上的 task；没有延迟的事件或唤醒在它们之间绕成一圈；几个发送者发到同一个事件位置；在同一个时刻点里互相有先后、绕成了环。

`sim.load_report().units` 是执行单元的个数。比 worker 少时，构建仿真时会警告：

```text
warning: group 0: 2 execution units for 3 workers
```

这时至少有一个 worker 一直没活干，但仍然占着它的核；结果不受影响。`1 execution unit` 就是串行运行。目前还不能列出每个执行单元里有哪些 task。

### 多开线程什么时候会快 {#when-threads}

多开线程，只有在有活的那些拍里活够多、又分在足够多的执行单元里，能让 worker 都忙起来时，运行才会快。活以外还要付两样：

- **2 个线程时运行 task 的只有一个 worker。**`workers_per_group: 2` 是一个线程分派工作、一个 worker 把所有 task 一个接一个地跑完，和 1 个线程一样，分派是多出来的。2 个线程不会比 1 个快。从 3 个线程起，两个 worker，task 才并排跑。
- **每个执行单元每一拍都有一点固定开销**，在它的 task 自己的活以外，不管它的门开着还是关着：至少百分之几微秒（粗量，随模型和机器变）。只有整个模型一个 task 都不跑的那些拍，才会成段跳过去。task 做的事少时，这点开销比并排跑省下的还多，运行反而比单线程慢。两种情况结果都一样。

模型的三件事决定快不快：

- **一拍里通常有几个实例有活。**数干活的，不数建出来的：64 个 bank 的缓存，一拍里只有 4 个忙，那只够 4 个 worker 有活干，不是 64 个。只有一个实例在干活时，第一个以外的线程都只是开销。
- **这几个实例是不是挨着的。**同一种模块的实例按编号连着分进执行单元：64 个实例、7 个 worker，大约前 9 个一个单元，接下来 9 个一个单元，依此类推。有活的那几个实例要是挨着，它们就在同一个执行单元里，在一个 worker 上一个接一个地跑，开多少线程都一样（[发牌式地分](#dealt)能治）。
- **干活的实例一拍做多少事**，和一个执行单元的开销比。task 的种类越多，没有[声明延迟](./time#declared-delays)的事件连成的链越长，执行单元就越多。单线程时关着的门不花钱。

### 什么时候发牌式地分 {#dealt}

一种模块的实例一拍里只有少数几个有活、而且编号挨着时，试一下 `cut: orr::CutPolicy::Dealt`。按轮转发活、每次从上次发到的下一个开始的分派器，活少的时候就是这样：它让忙起来的，是一个挨一个的那几个。`Dealt` 发牌式地分：第 *i* 个实例进 *k* 个单元里的第 (*i* mod *k*) 个，挨着的实例就分进了不同的单元，能并排跑。它不改结果。

每个实例每拍都有一点活的模型上，发牌式可能比缺省的 `Runs` 慢：百分之几到四成，每个实例做的事越少，慢得越多。两种都量一下。在一个 GPU 模型的 11 个测试程序上（8 个线程，按整个进程计时），一拍里只有少数几个核有活的程序，发牌式只用缺省的 0.71–0.89 的时间；每个核每拍都有活的程序，慢 2–7%。

有两样目前还看不到。哪个报告都不列每个执行单元里有哪些实例，所以忙的那几个是不是挤在一个单元里，要从模型自己看，Orrery 不告诉你。还有，`Dealt` 分出的执行单元可能比 `Runs` 少，为什么少也没有报告说：比两种的运行以前，两种各读一次 `sim.load_report().units`。

### 开几个线程 {#how-many-threads}

试跑来选。拿一段有代表性的输入，用 1 个线程跑一次，用最多的线程跑一次，留快的那个：

- 最多的线程数是一个缓存域里空着的物理核数（[线程](#threads)）。这些要是进程能用的全部 CPU，先把 `host_spin` 和 `idle_spin` 设成 0。一次只有少数几个挨着的实例有活时，最多线程那次再加上 `cut: orr::CutPolicy::Dealt` 跑一次（[见上](#dealt)）。
- 只比运行，即 `run_until` 那几次调用，不算建仿真：多线程建仿真要多花些时间，运行只有零点几秒时，它能左右整个进程哪个快。
- 量过的模型里，中间的线程数从来不是最快的；只有要给别的程序留核时才试少开。

模型要跑的每一种输入都试一下。同一个模型换一种输入，结论可能正相反：一个模型在每拍活很多的输入上 8 线程比 1 线程快，在活很少的输入上反而慢。所以模型多大、执行单元多少，都不能事先说哪个快。

一般说来，小的模型、每拍活很少的模型，用 1 个线程，也就是默认值。多开线程要在大的运行上才划算：实例很多，每个每拍都有不少活。

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

*L* 是模型的**免费延迟**，由 Orrery 在加载模型时算出。它取决于模型，从不取决于配置，`sim.stop_delay()` 会报告它。这里没有任何东西依赖计数器，所以它是 0；[GPGPU-Sim 模型](../examples/gpgpu-sim)里它是 425 ps。stop 延迟不能设。Orrery 算不出免费延迟的模型，stop 延迟按 0，装载报告里会写明（`reports: orr::Reports::All`）。

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

- 每次运行把到停下的地方为止的事件交给 sink，并让它 flush。`close_event_log` 交出剩下的，即已经发出、到达得更晚的事件，然后结束日志；此后再运行会失败。丢弃仿真也会结束日志，但错误就丢了。
- sink 返回的错误让这次运行失败，以后的运行也都失败。
- 同一个时刻点上的事件有固定的先后，所以一个模型在任何线程数下写出的日志文件都相同，两份日志可以逐字节比较。

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

### 写日志的线程 {#log-thread}

单线程时，sink 在调用 `run_until` 的线程上调用。多线程时，仿真另起一个线程来调用 sink（[线程](#threads)）：8 个线程加 sink 共 9 个，调用 `run_until` 的线程不算在内。也给写日志的线程一个单独的核。核不够时，仿真照样能结束，结果也相同，但可能变慢。

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
- `sim.stop_report()` 报告上一次 `run_until` 或 `step`（第一次之前是 `None`），`sim.load_report()` 报告仿真的构建；`reports: orr::Reports::All` 把两者都打印出来。
- 开启 `checks` 时，`sim.check_trace()` 验证访问同一个状态、其中至少一个是写的每两次访问都按串行顺序运行（多线程时，验证运行的同步给它们定了先后），否则返回违反之处。没开 `checks` 时，它返回一个说明这一点的错误。
- `sim.load_report().warnings` 里是加载时的警告，一条一行，和 stderr 上 `warning: ` 后面的一样（[报错](./errors#warnings)）。
