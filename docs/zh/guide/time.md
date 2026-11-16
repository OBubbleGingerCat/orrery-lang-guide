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

`init` 用 `ctx.clock_info(clock)` 读时钟的 `ClockInfo`，前提是这个时钟已经初始化、中继已经连接好（[`init`](./modules#init)）；宿主用 `plan.clock_config(handle)` 读。

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

请为每条链路和每个端口声明真实的最小延迟。它们可能影响模型的 stop 延迟，即 stop 之后仿真还要继续运行多久（[停止](./running#stopping)）；实际的值用 `sim.stop_delay()` 读取。

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
        gate.open(); // 被唤醒：从下一个时刻点起打开
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

单元自己知道下一次什么时候有活的时候（固定的延迟、一个计时器、要等满的时序参数），就把门控关到那时，而不是关上以后等别人叫醒：这样不用别的单元来发唤醒，也不用推算该提前几拍发。

- 写入从下一个时刻点起生效。在当前时刻点内，门控保持触发端口看到的值，`is_open()` 报告的也是这个值（`&'latest orr::Gate` 的读者例外，见下面）。
- `close_until(t)` 在 `t` 打开门控：在 `t` 或之后到达的事件可以通过，时钟从 `t` 或之后的第一个沿起再运行。`t` 不必是时钟沿。`orr::Timepoint::at(ps)` 是以皮秒计的一个时间的第 0 个 delta（以飞秒计用 `Timepoint::at_fs`）。
- 门控开始时是打开的，或者按复位值：
  `#[orr::reset(orr::Gate::closed())]`、
  `#[orr::reset(orr::Gate::closed_until(orr::Timepoint::at(4_000)))]`。
- `gate.opens()` 是按最后一次写入、门控打开的时刻点（打开时为 `Timepoint::ZERO`，关闭时为 `Timepoint::MAX`）。
- `gate.gated_edges()` 计门控关闭时的沿，直到当前时刻（含）：受门控的触发端口指定的每个时钟一个计数，按声明顺序。门控管几个 task 时，先按 task 名的字典序（宏给一个模块的 task 排的次序，也就是程序里 task 的次序），每个 task 里按触发端口的声明顺序；一个时钟在几个级上被指定，就有几项，各按各的级计。功耗和利用率的统计需要的就是它。

门控属于模块的实例，不属于某个触发端口，也不属于某个 task：

- 门控是模块自己的一个标量状态字段，类型为 `orr::Gate`；
- 一个 task 受门控的触发端口共用一个门控；一个门控可以管这个实例上几个 task 的触发端口：几个 task 的 `triggers` 用同一个带 `gate = g` 的字段，或几个字段都写 `gate = g`；
- 至多一个 task 写门控（取 `&mut orr::Gate`），可以是受它管的 task 之一，也可以是同一模块的另一个 task；
- 其他 task 可以读它；有 task 唤醒的门控（见[唤醒](#wakes)）例外，只有它管的 task 和它的写者能读。

宏和 `SimPlan::new` 会检查这些规则。门控关着时，它管的 task 都不在受门控的触发上跑。几个 task 要各自睡、各自醒时，才各用各的门控：几个 `ClockRelay` 字段，各写自己的 `gate = ..`，都接到同一个时钟。互不相干、要并行跑的 task 也各用各的门控：一个门控管的 task 和它的写者在同一个执行单元里，一个接一个地跑。

一个门控管几个 task 时：

- 同一个时钟的不同级是不同的时刻点（[delta 周期](#delta-cycles)）。写者在沿 e 的 δ0 关门，同一个沿 δ1 起它管的那几级就不跑了；写者在它管的最后一级关门，从下一个沿起才都不跑。每个沿只做一次决定，就让写者在最后一级跑。
- `close_until(t)`：每个受门控的时钟从它在 `t` 或之后的第一个点起再跑，各按各的级；事件从 `t` 起照收。
- 在 task 体里读门控：`&orr::Gate` 和写者的 `&mut orr::Gate` 取得的是时刻点开头的值，和受门控的触发看到的一样；`&'latest orr::Gate` 取得的是写者在这个时刻点写过以后的值，可能不同：写者在这个时刻点 `close()` 了，触发照样跑了，`&'latest` 读到的 `is_open()` 却是 `false`。受门控的 task 要知道这个时刻点门开着没有，用 `&orr::Gate`。`gated_edges()` 对写者和每个读者都一样。
- 写者在沿 e 的最后一级关门、只在这一级写时，各级的计数相等。一个不受门控的事件在沿 e 的 δ0 让写者开门时，门从 δ1 起开：δ0 那一项多计这个沿。
- `sim.dropped_events()` 照旧按实例、task 和触发端口列：两个受门控的 task 订同一个事件时，门关着来的一个事件在两个端口上各丢一次。
- 写者自己跑在一个不受门控的时钟上时，它每拍都跑，门关着也省不掉这个单元的拍。写者最好是受门控的 task 之一，或者由事件触发。

task 取得的 `&mut orr::Gate` 在这次调用期间就是门控本身。仿真之外造不出门控，所以 task 不能把它拿着的门控移出这次调用：`std::mem::take(gate)` 和 `std::mem::replace(gate, ..)` 都编不过，`orr::Gate::default()` 也编不过。同一次调用拿着的两个门控可以用 `std::mem::swap` 交换；通过其中任何一个写，写到的是它交换后拿着的那个门控。

两次运行之间，宿主通过 `sim.gate_mut(handle)` 取得门控：`open()`、`close()`、`close_until(t)`、`is_open()` 和 `opens()`。它写入的值在下一次运行开始时生效。`sim.state` 和 `sim.state_mut` 不接受门控。`sim.gated_edges(handle)` 读取计数。`sim.dropped_events()` 按实例、task 和触发端口列出丢弃的事件，如果有丢弃，仿真被丢弃时会打印一条警告；设了 `SimConfig::fail_on_drop`，第一个被丢弃的事件就是错误，它所在的时刻点一个 task 也不运行。快照带上门控、丢弃事件的计数和被门控的沿，所以恢复后的运行接着计数。

## 唤醒 {#wakes}

一个关了门控、等有活再开的单元，由带活来的那个单元通过**唤醒**（wake）打开。唤醒只打开门控，别的什么也不做：不运行 task，也不带值。

- 发的一方的模块有一个 `orr::Wake` 类型的字段（或者数组 `[orr::Wake; N]`），发唤醒的 task 在 `wakes = (..)` 里写上它。`ctx.wakes()` 里，`wakes = (..)` 的每一项对应一个条目：一个唤醒用 `ctx.wakes().0.wake()` 发；数组用 `wake(i)` 叫醒第 `i` 个元素，另有 `wake_after(i, delay)` 和 `len()`。
- 收的一方的模块有一个 `orr::WakeRelay` 类型的字段，标上它打开哪个门控：`#[orr::wake(gate = gate)]`。指明门控的中继是单个字段，不能是数组。
- 父模块在 init 里把两者连起来：`ctx.connect(module.producer.consumer, module.consumer.wake)?`。

一个 FIFO 空了就关门控的消费者，和每推一个就叫醒它的生产者：

```rust
#[orr::module]
pub struct Consumer {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::wake(gate = gate)]
    pub wake: orr::WakeRelay,
    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,
    pub gate: orr::Gate,
    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    #[orr::task(triggers = (clock,))]
    fn take(
        _ctx: &mut orr::TaskCtx,
        inbox: &mut orr::FifoRx<u64>,
        gate: &mut orr::Gate,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        if inbox.is_empty() {
            gate.close(); // 等生产者叫醒
        }
        Ok(())
    }
}

#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub out: orr::FifoTx<u64>,
    pub consumer: orr::Wake,
    next: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,), wakes = (consumer,))]
    fn send(
        ctx: &mut orr::TaskCtx,
        out: &'latest mut orr::FifoTx<u64>,
        next: &mut u64,
    ) -> Result<(), orr::SimError> {
        *next += 1;
        if *next % 4 == 0 && out.push(*next).is_ok() {
            ctx.wakes().0.wake();
        }
        Ok(())
    }
}
```

**唤醒什么时候生效。**

- `wake()` 从下一个时刻点起打开门控：关着的单元在它受门控的时钟的、现在之后的第一个沿再运行。同一个时刻点里对同一个门控的几次唤醒算一次。
- 唤醒赢过门控的写者在同一个时刻点写的 `close()`。上面的消费者可能就在生产者推入并叫醒它的那一拍发现 FIFO 空了、写了关；门控仍然开着。
- 门控管几个 task 时，唤醒叫醒它们全部：每个受门控的触发从它在门开以后的第一个点起再跑，各按各的级。
- `wake_after(delay)` 从现在 + `delay`（`orr::Time::DELTA` 或一个正的时间）或之后的第一个沿起打开门控。它在那之前一刻生效：赢过在那之前写的所有 `close()`，从那时起写的 `close()` 赢过它。在那个时间或之后到达受门控触发端口的事件，也能通过。对 *n* 个沿以后到期的值，`orr::wake_delay(now, clock, n)?` 就是让门控正好在值被看见的那个沿打开的延迟（[延迟](./components#delays)）。

一个每次运行都关上自己门控的单元，和一个跑在同一个 1 ns 时钟上、唤醒其中两个的 feeder（`units: [orr::Wake; 2]`，写在它的 task 的 `wakes` 里）：

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
        gate.close(); // nothing to do until woken
        Ok(())
    }
}

// In the feeder's task:
let wakes = &ctx.wakes().0;
match ctx.now().ps() {
    0 => {
        wakes.wake_after(0, orr::Time::ns(2))?; // unit 0 runs at 0 and 2 ns
        wakes.wake_after(1, orr::Time::ns(2))?; // unit 1 runs at 0, 2 and 3 ns
    }
    2_000 => wakes.wake(1), // wins over unit 1's close at 2 ns
    _ => {}
}
```

两个门控都在 2 ns 之前一刻打开，两个单元在那里运行时又都把它关上。单元 0 的门控就此关着；单元 1 的门控因为 2 ns 那次 `wake()`，从下一个时刻点起又打开了。

唤醒和写者的 `close()` 的先后，门控管一个 task 还是几个都一样：

| 情形 | 结果 |
|---|---|
| 写者在时刻点 τ `close()`，同一个时刻点有 `wake()` | 门控从 τ 的下一个时刻点起开着（唤醒赢） |
| `wake()` 在 τ，写者在 τ 的下一个时刻点 τ′ `close()` | 门控在 τ′ 开着，它管的 task 在 τ′ 跑；从 τ′ 的下一个时刻点起关（`close()` 赢） |
| `wake()` 在沿 e 的 δ0，受门控的 task 在同一个沿的 δ1…δ5 | 从 δ1 起开：同一个沿后面几级就跑 |
| `wake_after(d)` 在 τ，写者在 τ + d 以前 `close()` | 唤醒赢：门控从 τ + d 或之后的第一个点起开 |
| `wake_after(d)` 在 τ，写者在 τ + d 或以后 `close()` | `close()` 赢 |
| 几个 task 在同一个时刻点唤醒同一个门控 | 算一次 |

**谁先动作决定唤醒赶不赶得上。** 唤醒不能让被叫醒的 task 在发出唤醒的这个时刻点里运行。这要不要紧，取决于两个 task 在一拍里的先后（[谁先动作](./components#who-acts-first)）：

| 生产者 | 消费者 | 消费者关着时推进去的值，什么时候被弹出 |
|---|---|---|
| `&'latest mut FifoTx`（后动） | `&mut FifoRx`（先动） | 下一拍，和门控一直开着一样：赶得上 |
| `&mut FifoTx`（先动） | `&'latest mut FifoRx`，由 `clock + delta(1)` 触发 | 同一拍，和门控一直开着一样：赶得上 |
| `&mut FifoTx`（先动） | `&'latest mut FifoRx`，同一层 | **晚一拍**：门控一直开着时同一拍就会弹出 |

第三种排法，要么把消费者挪到晚一层的 delta，要么在可能有值到来时不关它的门控。检查一个关门控的单元：把同一个模型改成门控从不关再跑一遍，结果要一样，而且在同样的拍上。

**穿过子模块。** `#[orr::wake(forward)]` 让唤醒穿过模块的边界：

- 标在 `orr::Wake` 字段上，把子模块的唤醒传出去：模块的 init 把子模块的唤醒连到它，父模块像连 task 的唤醒那样连它。task 不通过它发唤醒。
- 标在 `orr::WakeRelay` 字段上（可以是数组），把唤醒传进来：模块的 init 把它连到子模块的中继。

转发的唤醒上没有连任何唤醒，或者转发的中继最终没有通到门控，建仿真时报错。

**用唤醒还是用不受门控的事件。** [时钟门控](#clock-gates)那个例子里的 `wake` 触发端口是一个不受门控的事件：它到达时运行 task，由 task 打开门控。被叫醒的单元在自己的状态或 FIFO 里就能找到要做的事，用唤醒；叫醒时要带一个值，或者单元必须在它到达的那一刻就动作，用不受门控的事件。
