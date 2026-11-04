# 时间：时钟与事件

## 时间与时刻点

`orr::Time` 是以整数飞秒计的物理时间，或者是 `Time::DELTA`，即一个 delta 周期：

```rust
use orrery_lang as orr;

let period = orr::Time::ns(10); // 也有 fs、ps、us、ms、s
let link = orr::Time::from_ps(425);
let next = orr::Time::DELTA;
```

它本身没有算术运算，也不能比较大小；比较和相加要通过 `as_fs()`，对 `DELTA`
它返回 `None`。`from_ps` 和 `as_ps()` 以整皮秒计：`as_ps()` 会舍掉不足一皮秒的部分。时间按能精确表示它的最大单位显示：`10ns`、`1500ps`、`1500fs`。在参数文件里，时间写成字符串：`"10ns"`、`"250 ps"`、`"1500fs"`、`"delta"`。

**时刻点**是一个物理时间加上一个 delta 周期，先按时间、再按 delta 排序，显示为
`10ns` 或 `10ns+1`。`Timepoint::at(ps)` 和 `Timepoint::at_fs(fs)` 给出 delta 0 上的时刻点。时钟在 delta 0 触发；用 `DELTA` 发出的事件在同一时间的下一个
delta 到达；用物理延迟发出的事件在之后那个时间的 delta 0 到达。时刻点严格按顺序运行：先是 `10ns` 的全部，然后是 `10ns+1` 的全部，依此类推。

## 时钟 {#clocks}

独立的时钟是一个 `orr::Clock` 字段，在 `init` 里配置一次，给出周期和相位（第一个时钟沿的时间，小于周期），或者给出以赫兹计的频率和相位：

```rust
ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
ctx.init_clock_hz(module.core, 1_132_000_000, orr::Time::ps(0))?;
```

按频率给出的时钟，周期是 10¹⁵ / *hz* fs，不必是整数。每个时钟沿落在它的精确时间四舍五入到最近的飞秒上，所以时钟沿永远不会漂移：1132 MHz 时，沿在 0、883392 fs、1766784 fs、2650177 fs，依此类推。

接收时钟的模块声明一个 `orr::ClockRelay`，并被连到时钟上：
`ctx.connect(module.clock, module.child.clock)?`。由时钟触发的 task 以时钟的
`orr::ClockInfo` 作为触发载荷：

- `period()` 是写成分数的周期，`(num, den)` 飞秒；`interval()` 是两个沿之间最小的间隔；`phase()` 是相位。
- `cycle_of(now)` 是一个时刻点落在第几个沿上，相位处的沿算第 1 个；`edge(cycle)` 是某个沿的时刻点；`is_edge(now)` 和 `edges_before(now)` 给出一个时刻点是不是沿、它之前有几个沿。

`init` 用 `ctx.clock_info(clock)` 读时钟的 `ClockInfo`，宿主用 `plan.clock_config(handle)`。

**多个时钟域**就是多个时钟。它们的周期和相位都是精确的，所以应该重合的时钟沿永远重合。GPGPU-Sim 模型的核心跑在 1132 MHz，DRAM 跑在 850 MHz，分别表示为
425 和 566 皮秒的周期，两者之间的比例因此保持精确。

时钟的个数，以及它们的周期怎样组合，都没有限制，只有一点例外：两个时钟的沿相距不到 1 fs 而又不重合时，例如一个 3 GHz、一个周期 333333 fs，加载会失败（`LoadError::Init`，`stage` 为 `clocks`）。时钟的周期和相位在 `init` 设定之后就固定了；运行中频率会变的单元，可以改用事件触发自己，每次按它在状态里保存的周期发出。

## 事件 {#events}

事件源有一个载荷类型，必须是 plain 类型：

```rust
#[orr::trigger]
pub done: orr::EventSource<u64>,
```

可能往它上面发送的 task 在 `emits` 里列出它，并带一个延迟发送：

```rust
ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;   // 下一个 delta 周期
ctx.emit_triggers().0.emit(*value, orr::Time::ns(2))?;   // 2 ns 之后
```

延迟是 `DELTA` 或一个正的物理时间；物理延迟为零是错误。接收方声明一个
`orr::EventRelay<T>`，连到源上，或者连到另一个中继上；中继链不增加延迟，一个中继恰好有一个源。由中继触发的 task 在自己的触发槽位里拿到载荷。

一个源在每个时刻点最多接收一个事件：同一位置、同一时刻点的两个事件，不论来自两个 task 还是一个，都是**多驱动**错误。要同时发送几个值，就把它们放进一个载荷，或者用一个源数组。

## 声明的延迟 {#declared-delays}

事件源可以声明它的事件可能用的延迟：

```rust
#[orr::param("link delay", default = orr::Time::ns(2))]
link: orr::Time,

#[orr::trigger(min_delay = link)]
pub out: [orr::EventSource<Packet>; 4],

#[orr::trigger(max_delay = orr::Time::DELTA)]
pub hint: orr::EventSource<()>,
```

界限是一个常量 `orr::Time`，或者本模块一个 `Time` 参数的名字。每次发出都会对照它检查：超出界限的发出让这次运行失败。

请为每条链路和每个端口声明真实的最小延迟。声明的最小延迟从不改变结果。如果一个可能结束仿真的 task 用到的源声明了最小延迟，这个延迟也决定了 Stop 之后仿真还要继续运行多久（[停止](./running#stopping)）：一个会停止、又往 `min_delay` 为 5 ns 的源上发送的 task，给出 5 ns 的 stop 延迟。

`max_delay = Time::DELTA` 表示这个事件总是只晚一个 delta 周期到达，所以永远不会碰上时钟沿：它触发的 task 永远不会和时钟触发的 task 在同一个时刻点运行。两者之间的冲突于是不可能发生，既不会作为可能的冲突报告，也不会在仿真运行时检查。

## delta 周期 {#delta-cycles}

delta 周期给同一个物理时间之内的事情定序：一串事件，每个都用 `DELTA` 发出，就作为同一时间的一系列时刻点运行。同一时间超过 `delta_limit` 个 delta 周期（默认 30）是错误，这能抓住永远稳定不下来的模型。需要更长链条的模型，可以设置 `SimConfig::delta_limit`，最大 62。

不论上限是多少，一次运行最多走到约 288 秒的仿真时间。`run_until` 超过它就返回 `RunError::Request`，什么也不运行；仿真还可以用一个够得着的时间继续。

## 时钟门控 {#clock-gates}

一个没有事做的单元可以关掉自己的时钟。它把门控声明为一个状态字段，并在受这个门控控制的触发端口上指明它：

```rust
#[orr::module]
pub struct Sm {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::trigger(gate = gate)]
    pub request: orr::EventRelay<Request>,
    #[orr::trigger]
    pub wake: orr::EventRelay<()>,
    pub gate: orr::Gate,
    pub next_work: u64,
}
```

受门控的触发端口只在门控打开时触发。门控关闭时的时钟沿不运行 task；如果某个沿上所有受门控的时钟都关着，那里也没有别的事发生，这个沿就根本不是时刻点，`sim.step()` 会跳过它。门控关闭期间到达受门控触发端口的事件被丢弃并计数。没有 `gate =` 的触发端口不受影响，所以一个不受门控的事件可以在门控关闭时运行 task（这时受门控的槽位是 `None`），由它打开门控。

task 以 `&mut orr::Gate` 取得门控：

```rust
#[orr::task(triggers = (clock, request, wake))]
fn step(
    ctx: &mut orr::TaskCtx,
    gate: &mut orr::Gate,
    next_work: &mut u64,
) -> Result<(), orr::SimError> {
    if ctx.triggered_triggers().2.is_some() {
        gate.open(); // 被唤醒：从下一个沿起再运行
        return Ok(());
    }
    // ……一个周期的工作，其中设置 `next_work` ……
    match *next_work {
        0 => {}                                                    // 忙：保持打开
        u64::MAX => gate.close(),                                  // 直到被唤醒
        at => gate.close_until(orr::Timepoint::at(u128::from(at))), // 已知的截止时间
    }
    Ok(())
}
```

- 写入从下一个时刻点起生效。在当前时刻点内，门控保持触发端口看到的值，`is_open()` 报告的也是这个值。
- `close_until(t)` 在 `t` 打开门控：在 `t` 或之后到达的事件可以通过，时钟从 `t` 或之后的第一个沿起再运行。`t` 不必是时钟沿。
- 门控开始时是打开的，或者按复位值：
  `#[orr::reset(orr::Gate::closed())]`、
  `#[orr::reset(orr::Gate::closed_until(orr::Timepoint::at(4_000)))]`。
- `gate.opens()` 是按最后一次写入、门控打开的时刻点（打开时为 `Timepoint::ZERO`，关闭时为 `Timepoint::MAX`）。
- `gate.gated_edges()` 计门控关闭时的沿，直到当前时刻（含）：受门控的触发端口指定的每个时钟一个计数，按声明顺序。功耗和利用率的统计需要的就是它。

门控、它控制的触发端口和写它的 task 是一体的：

- 门控是模块自己的一个标量状态字段，类型为 `orr::Gate`；
- 一个门控只控制一个 task 的触发端口，一个 task 受门控的触发端口共用一个门控；
- 只有这个 task 写门控；其他 task 可以读它。

宏和 `SimPlan::new` 会检查这些规则。task 取得的门控留在这次调用里：对它用 `std::mem::take` 或 `std::mem::replace` 编译不过。

两次运行之间，宿主通过 `sim.gate_mut(handle)` 取得门控：`open()`、`close()`、`close_until(t)`、`is_open()` 和 `opens()`。它写入的值在下一次运行开始时生效。`sim.state` 和 `sim.state_mut` 不接受门控。`sim.gated_edges(handle)` 读取计数。`sim.dropped_events()` 按实例、task 和触发端口列出丢弃的事件，如果有丢弃，仿真被丢弃时会打印一条警告；设了 `SimConfig::fail_on_drop`，第一个被丢弃的事件就是错误，它所在的时刻点一个 task 也不运行。快照带上门控、丢弃事件的计数和被门控的沿，所以恢复后的运行接着计数。

## 唤醒 {#wakes}

一个 task 可以不发事件，而用**唤醒**打开另一个模块的门控。发送方的模块声明一个 `orr::Wake` 字段，并在 task 的 `wakes` 里列出它；接收方的模块声明一个 `orr::WakeRelay`，指明它打开哪个门控；由某个 `init` 用 `ctx.connect` 把两者连起来：

```rust
#[orr::module]
pub struct Unit {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::wake(gate = gate)]
    pub wake: orr::WakeRelay,
    gate: orr::Gate,
    pub runs: u64,
}

#[orr::module_impl]
impl Unit {
    #[orr::task(triggers = (clock,))]
    fn step(_ctx: &mut orr::TaskCtx, gate: &mut orr::Gate, runs: &mut u64) -> Result<(), orr::SimError> {
        *runs += 1;
        gate.close(); // 没事可做，直到被唤醒
        Ok(())
    }
}

#[orr::module]
pub struct Feeder {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    pub units: [orr::Wake; 2],
}

#[orr::module_impl]
impl Feeder {
    #[orr::task(triggers = (clock,), wakes = (units,))]
    fn feed(ctx: &mut orr::TaskCtx) -> Result<(), orr::SimError> {
        let clock = ctx.triggered_triggers().0.expect("runs on its clock");
        let wakes = &ctx.wakes().0;
        if clock.cycle_of(ctx.now()) % 4 == 0 {
            wakes.wake(1); // 从下一个时刻点起
            wakes.wake_after(0, clock.interval())?; // 从下一个沿起
        }
        Ok(())
    }
}
```

```rust
ctx.connect(module.feeder.units[i], unit.wake)?;
```

- `wake()` 从下一个时刻点起打开门控：这个单元在它受门控的时钟此后的第一个沿上再次运行。`wake_after(delay)` 从 now + `delay` 或之后的第一个沿起打开门控，`delay` 是 `DELTA` 或一个正的物理时间。
- 同一个时刻点上对一个门控的几次唤醒算一次；同一个时刻点上，唤醒胜过门控自己的 task 写的 `close()`。
- `ctx.wakes()` 里，`wakes = (..)` 的每一项对应一个条目：一个唤醒；对数组则是 `wake(i)`、`wake_after(i, delay)` 和 `len()`。
- 标了 `#[orr::wake(forward)]` 的 `Wake` 或 `WakeRelay` 把唤醒传出子模块或者传进子模块，就像中继传递触发端口一样。

目前，有 task 发出唤醒的模型用一个 worker 运行（[worker](./running#workers)）。
