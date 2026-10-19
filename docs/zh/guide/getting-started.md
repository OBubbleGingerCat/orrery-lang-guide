# 快速开始

这一页搭出最小的有用模型，即事件流水线：一个时钟，加上若干个 worker；每个
worker 里有一个随时钟计数的 producer，和一个把 producer 发来的值累加起来的 sink。

```text
Top
└── workers: [Worker; worker_count]
    ├── producer: Producer   on every clock edge: value += step; send value
    └── sink: Sink           on every value received: total += value
```

## 准备 {#setting-up}

Orrery 没有发布到 crates.io。从你能访问的一份检出依赖它：

```toml
[dependencies]
orrery-lang = { path = "../orrery-lang/crates/orrery-lang" }

# 推荐：task 的函数体、宏生成的代码和 Orrery 本身
# 分在不同的 crate 里，跨 crate 内联让模型更快。
[profile.release]
lto = "fat"
codegen-units = 1
```

Orrery 需要较新的稳定版 Rust。本站的示例把它叫作 `orr`，每个文件开头写
`use orrery_lang as orr;`。

## producer

模块是标记了 `#[orr::module]` 的结构体。它的字段是参数（`#[orr::param]`）、触发源（`#[orr::trigger]`）、子模块（`#[orr::submodule]`），以及不带属性的状态。

```rust
use orrery_lang as orr;

#[orr::module]
pub struct Producer {
    #[orr::param("increment per tick", default = 1)]
    step: u64,

    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::trigger]
    pub done: orr::EventSource<u64>,

    value: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,), emits = (done,))]
    fn produce(ctx: &mut orr::TaskCtx, value: &mut u64) -> Result<(), orr::SimError> {
        *value += *ctx.module().step;
        ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;
        Ok(())
    }
}
```

- `clock` 是一个**中继**：时钟来自上一层的模块。
- `done` 是一个**事件源**，载荷是 `u64`。
- 这个 task 在每个时钟沿运行。`value: &mut u64` 以写的方式绑定同名的状态字段。
  `emit(.., orr::Time::DELTA)` 在一个 delta 周期之后、同一时刻发出这个值。

## sink

```rust
#[orr::module]
pub struct Sink {
    #[orr::trigger]
    pub done: orr::EventRelay<u64>,

    pub total: u64,
}

#[orr::module_impl]
impl Sink {
    #[orr::task(triggers = (done,))]
    fn consume(ctx: &mut orr::TaskCtx, total: &mut u64) -> Result<(), orr::SimError> {
        if let Some(value) = ctx.triggered_triggers().0 {
            *total += *value;
        }
        Ok(())
    }
}
```

`triggered_triggers()` 为每个触发源给出一个槽位，触发了的那些是
`Some(payload)`。

## 连接

连接在模块的 `#[orr::init]` 里用显式的调用建立；没有任何东西按名字自动连接。

```rust
#[orr::module]
pub struct Worker {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::submodule]
    producer: Producer,

    #[orr::submodule]
    sink: Sink,
}

#[orr::module_impl]
impl Worker {
    #[orr::init]
    fn init_worker(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.connect(module.clock, module.producer.clock)?;
        ctx.connect(module.producer.done, module.sink.done)?;
        Ok(())
    }
}

#[orr::module]
pub struct Top {
    #[orr::param("number of workers", default = 2)]
    worker_count: usize,

    #[orr::trigger]
    pub clock: orr::Clock,

    #[orr::submodule]
    workers: [Worker; orr::DynShape(worker_count)],
}

#[orr::module_impl]
impl Top {
    #[orr::init]
    fn init_top(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
        for worker in module.workers.iter() {
            ctx.connect(module.clock, worker.clock)?;
        }
        Ok(())
    }
}
```

- `orr::Clock` 是一个独立的时钟；`init_clock` 给它 10 ns 的周期和 0 相位。
- `[Worker; orr::DynShape(worker_count)]` 是一个数组，长度是参数
  `worker_count`。

## 运行

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let params = std::env::args_os().nth(1).map(std::path::PathBuf::from);
    let plan = orr::SimPlan::<Top>::new(params, None)?;
    let mut sim = plan.into_sim(orr::SimConfig::default())?;
    match sim.run_until(orr::Time::ns(100))? {
        orr::RunEnd::Reached(time) => println!("ran until {time}"),
        orr::RunEnd::Finished(finished) => println!("finished at {}", finished.at),
    }
    let totals: Vec<_> = (sim.top().workers.iter())
        .map(|worker| worker.sink.total)
        .collect();
    for (index, total) in totals.into_iter().enumerate() {
        println!("workers[{index}].sink.total = {}", sim.state(total)?);
    }
    Ok(())
}
```

- `SimPlan::new` 读取参数（给的是 `None` 时用默认值），建立实例，运行各个
  `init`，并检查模型。
- `into_sim` 构建仿真，这里用的是默认配置（[运行仿真](./running#configuration)）。
- `run_until` 运行 100 ns 及之前的每个时刻点。时钟沿落在 0、10、…、100 ns，所以每个 sink 最后是 1 + 2 + … + 11 = 66。
- `sim.top()` 给出模型状态的句柄，用 `sim.state` 读取。

## 参数

参数来自一个 TOML 文件，文件里的表按实例树组织。三个 worker，第二个每次加十：

```toml
[top]
worker_count = 3

[[top.workers]]
__select_index__ = 1
[top.workers.producer]
step = 10
```

```text
ran until 100ns
workers[0].sink.total = 66
workers[1].sink.total = 660
workers[2].sink.total = 66
```

## 下一步

- [模块与参数](./modules)和[状态与 task](./state-and-tasks) 完整地解释了刚才写下的东西。
- [时间](./time)介绍时钟、事件和延迟。
- [运行仿真](./running)介绍配置、停止和读取结果。
