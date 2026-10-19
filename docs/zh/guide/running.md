# 运行仿真

## 从 plan 到仿真

```rust
use orrery_lang as orr;

let plan = orr::SimPlan::<Top>::new(Some("params.toml".into()), None)?;
let mut sim = plan.into_sim(orr::SimConfig::default())?;
```

- `SimPlan::new(params, snapshot)` 展开并检查模型。两个参数都是可选的路径：没有参数文件时，每个参数保持默认值；给了快照时，仿真从快照开始（[快照](./snapshots)）。
- `into_sim(config)` 构建仿真。带[外部端口](./external-ports)的模型用
  `into_sim_with(config, |top, ports| ..)` 绑定端口。

## 配置 {#configuration}

`orr::SimConfig`：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `checks` | `false` | 记下 task 访问状态的先后，供 `sim.check_trace()` 检查；慢，用于测试 |
| `delta_limit` | 30 | 同一个物理时间上 delta 周期数的上限，最大 65,534；设得越大，一次运行在时间上能走得越短（[delta 周期](./time#delta-cycles)） |
| `stop_delay` | `None` | stop 之后仿真还运行多久；`None` 表示模型的免费延迟（[停止](#stopping)） |
| `event_capacity` | 2²⁰ | 一个实例的一个 task 在一个事件源上最多能有多少个在途事件，即已经发出、还没到达的 |
| `fail_on_drop` | `false` | 门控关闭时到达触发端口的事件是错误，而不是被丢弃并计数（[时钟门控](./time#clock-gates)） |
| `snapshots` | `None` | 何时、在哪里拍快照 |

这些字段都不改变结果，只有 `stop_delay`、`delta_limit`、`event_capacity` 和 `fail_on_drop` 例外，它们以确定的方式改变结果：给定的值在任何机器、任何 worker 数下都得到同样的结果。

```rust
let config = orr::SimConfig {
    checks: true,
    delta_limit: 100,
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

## 运行之后

- 一旦有 task 停止，`sim.finished()` 就返回这个 stop。
- 开启 `checks` 时，`sim.check_trace()` 验证访问同一份状态、其中至少一个是写的每两次访问都按串行顺序运行，否则返回违反之处。
