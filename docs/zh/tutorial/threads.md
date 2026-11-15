# 10. 多开线程更快

这一章把前面几章合起来讲：模型什么时候随线程数变快，跑之前和跑一次以后看哪些数，不变快时怎么办。事实在指南[多开线程什么时候会快](../guide/running#when-threads)里；这里把它们写成按次序查的规则，每条配本教程里的一个例子。

## 按这个次序查

**1. 2 线程不比 1 线程快。** *N* 线程的运行里，一个线程派活，*N* − 1 个跑 task。2 线程时只有一个线程跑 task，和 1 线程一样，还要多付派活的钱。本教程的例子在 2 线程上和 1 线程一样快，或者略慢。从 4 线程起才并行。同时最多跑 *N* − 1 个执行单元，所以 8 线程最多约是 1 线程的 7 倍快。

**2. 干活的实例不能挨着。** Orrery 把同一种模块的实例按编号连着分进执行单元。一拍里只有几个实例干活、又是挨着的，它们就全落在一个执行单元里，开多少线程都一个接一个地跑。这是要最先查的一条：它会让别的数都看着没问题，8 线程却一点也不快。

**3. 每拍的活要够这么多执行单元分。** 只要这一拍有东西在跑，每个执行单元都要花钱，门开不开都一样。用下面的比值 *R* 估一估。

**4. task 的种类越少，执行单元越少。** 有很多实例的一种 task，约分成 min(实例数, *N* − 1) 个执行单元。所以每多一种 task，就多出最多 *N* − 1 个执行单元，每拍都要付，门全关着也要付。在一个小模型上，8 线程时每种约 0.3 µs 一拍。
总在同样的拍上一起跑的 task，可以考虑合并（第 6 章，待写）。只喂另一个单元的小单元也可以并，两者不在同样的拍上跑也一样：往一个输入缓冲里放东西的源，可以写进有这个缓冲的那个单元里，就少一种 task。在一个开关网络的模型里，源单独写的用时，1 线程是把源写进第一级的 1.1–1.2 倍，8 线程最多 1.9 倍；每个输入、每个线程数都是单独写的慢。

**5. 带延迟的线并排跑**（[第 5 章](./lines)）。一串不带延迟的线，就是一串执行单元，一个等一个。

## 规则 2：让干活的实例散开

64 条通道，只有 4 或 8 条干活，其余的永远关着门。干活的通道要么挨着（第 0、1、2……条），要么散开（每隔 16 或 8 条一条）：

```rust
#[orr::module]
pub struct Lane {
    #[orr::trigger(gate = gate)]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub working: bool,
    #[orr::constant]
    pub work: u64,
    gate: orr::Gate,
    pub state: u64,
}

#[orr::module_impl]
impl Lane {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, working: &'constant bool, work: &'constant u64, gate: &mut orr::Gate, state: &mut u64) -> Result<(), orr::SimError> {
        if !*working {
            gate.close(); // for good
            return Ok(());
        }
        *state = mix(*state ^ 1, *work);
        Ok(())
    }
}

#[orr::module]
pub struct Lanes {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("working lanes", default = 4usize)]
    working: usize,
    #[orr::param("how far apart the working lanes are: 1 for neighbours", default = 1usize)]
    stride: usize,
    #[orr::param("mixer rounds a working lane does in a cycle", default = 1000u64)]
    work: u64,
    #[orr::trigger]
    clock: orr::Clock,
    #[orr::submodule]
    lanes: [Lane; orr::DynShape(n)],
}

#[orr::module_impl]
impl Lanes {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        let (working, stride) = (*module.working, *module.stride);
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, lane) in module.lanes.iter().enumerate() {
            ctx.connect(module.clock, lane.clock)?;
            ctx.set_const(lane.working, i % stride == 0 && i / stride < working)?;
            ctx.set_const(lane.work, *module.work)?;
        }
        Ok(())
    }
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    let end = orr::Time::ps(PERIOD_PS * 20_000 - 1);
    for working in [4usize, 8] {
        let mut cases = Vec::new();
        for (stride, cut, how) in [
            (1usize, orr::CutPolicy::Runs, "neighbours, Runs"),
            (1, orr::CutPolicy::Dealt, "neighbours, Dealt"),
            (64 / working, orr::CutPolicy::Runs, "spread out, Runs"),
        ] {
            let file = params(&format!("ch10-w{working}-s{stride}"), &format!("[top]\nworking = {working}\nstride = {stride}\n"))?;
            cases.push(Case::new(
                format!("{working} of 64 working, {how}"),
                orr::SimConfig { cut, ..orr::SimConfig::default() },
                move |config| Ok(orr::Sim::new(orr::SimPlan::<Lanes>::new(Some(file.clone()), None)?, config)?),
                move |sim| {
                    sim.run_until(end)?;
                    let mut sum = 0;
                    for lane in sim.top().lanes.iter() {
                        sum = mix(sum ^ *sim.state(lane.state)?, 1);
                    }
                    Ok(sum)
                },
            ));
        }
        compare(&threads, 3, &cases)?;
    }
    Ok(())
}
```

运行的秒数（粗量）。执行单元数是 1、1、3、7：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 |
|---|---|---|---|---|---|
| 4 条挨着，`Runs` | 0.154 | 0.157 | 0.159 | 0.159 | 0.96× |
| 4 条挨着，`Dealt` | 0.153 | 0.157 | 0.082 | 0.045 | 3.38× |
| 4 条散开，`Runs` | 0.154 | 0.157 | 0.083 | 0.045 | 3.42× |
| 8 条挨着，`Runs` | 0.305 | 0.309 | 0.310 | 0.311 | 0.98× |
| 8 条挨着，`Dealt` | 0.305 | 0.309 | 0.121 | 0.084 | 3.63× |
| 8 条散开，`Runs` | 0.305 | 0.309 | 0.121 | 0.084 | 3.64× |

- **挨着的，多开线程什么也得不到。** 用缺省的切法 `Runs`，8 线程和 1 线程一样长。
- **`cut: orr::CutPolicy::Dealt` 把实例一个一个发牌式地分**，挨着的就分进了不同的执行单元。8 线程于是快到 3.4–3.6 倍，和手工把通道散开差不多。
- **事先从模型的派活方式看出干活的实例是不是挨着。** 一个按轮转派活、每次从上次派到的下一个起的派活方，活少的时候派出来的就是挨着的。今天没有报告列出每个执行单元里有哪些实例，要从模型自己推。
- **`Dealt` 有代价**：每个实例每拍都有一点活的模型上，它可能慢，慢多少看模型，百分之几到两成都有。两种都量一下。
  - 随 Orrery 附带的 GPU 模型里，命令处理器把线程块轮流发给下一个核，所以线程块少的程序，忙着的核挤在一起。
  - Orrery 开发方在 11 个测试程序上、8 线程、按整个进程计时：一拍里只有少数几个核有活的程序，发牌式只用缺省的 0.71–0.89 的时间；每个核每拍都有活的程序，多花 2–7%。

## 规则 3：每拍的活够不够

拿一段典型的输入在 1 线程上跑一次。输入差得多的（比如一种很忙、一种大多数拍没活），结论可能正相反：各定各的。然后：

1. 运行时间（只算运行）÷ 拍数，就是每拍的时间。大多数拍什么事都没有的，除以有活的拍数。怎么数：同一段输入用 `sim.step()` 再跑一次，数它返回了几个 `StepEnd::Executed`（[运行](../guide/running#how-many-threads)）。
2. 用 8 线程建一次仿真，读 `sim.load_report().units`。
3. 算 *R* = 每拍的时间 ÷（执行单元数 × 0.1 µs）。
   - *R* ≥ 2.5：只要干活的实例不挨着（规则 2），8 线程就快。
   - 不到 2.5：可能快也可能慢，要量。

拿本教程的例子来核这个估法（粗量）：

| 例子 | 1 线程每拍 | 8 线程执行单元 | *R* | 8 线程比 1 线程 |
|---|---|---|---|---|
| 通道，活重（[第 1 章](./measure)） | 121 µs | 7 | 170 | 6.0× |
| 通道，活轻（第 1 章） | 0.67 µs | 7 | 1.0 | 1.6× |
| 一个循环，活轻（第 1 章） | 0.59 µs | 1 | 5.9 | 0.55× |
| 带延迟的级，活轻（[第 5 章](./lines)） | 3.7 µs | 14 | 2.7 | 2.4× |
| 不带延迟的级，活轻（第 5 章） | 4.5 µs | 56 | 0.8 | 0.97× |
| 4 条挨着的通道，`Runs`（上面） | 7.7 µs | 7 | 11 | 0.96× |

- 活分得开的地方，*R* 管用。
- 活分不开的地方，*R* 什么也说明不了：
  - 循环只有一个执行单元；
  - 四条挨着的通道在一个执行单元里。
  
  两者 *R* 都高，却都不快。所以先查规则 1 和 2。
- 很多一模一样、每个几乎没活的单元，*R* 低也可能快，比如活轻的通道。0.1 µs 是粗量，在这些小模型上每个执行单元的实际开销更低（约 0.06 µs），而且随模型、随机器变。你自己的地板，是 8 线程每拍时间 ÷ 执行单元数量到的最低值。

## 配置

- `workers_per_group`：线程数。单线程和最多（一个缓存域里空着的物理核数）各跑一次，留快的；中间的只在要给别的程序留核时试。2 不会比 1 快。
- `cut`：`Runs`（缺省）或 `Dealt`。一次只有少数实例干活时，两种都试。
- `cpus`：运行可以从哪些 CPU 里拿核。给它没有别的活的 CPU，不然别的活会让它慢得不均匀。
- `host_spin` 和 `idle_spin`：进程能用的 CPU 只有线程数那么多时，两个都设成 0：这时调 `run_until` 的线程没有自己的 CPU。

见指南：[配置](../guide/running#configuration)、[线程](../guide/running#threads)。

## 不变快时

从外面看得到的东西走一遍：

1. **装载报告**：每个线程数的执行单元数。是不是比一拍里干活的实例多得多？那就找可以合并的 task 种类，以及不带延迟的线串。
2. **自己的计数**：典型的一拍里，有几个实例干活、是哪几个？挨着吗？那就试 `Dealt`。
3. **1 线程每拍的时间**，和执行单元数比（*R*）。*R* 低，就是每拍的活不够分给这么多执行单元：要么减少执行单元（规则 4、5），要么少开线程。
4. **停止报告**（`reports: orr::Reports::All`）给出运行的墙钟时间，多线程时还有活是怎么分的。

**运行到一半换线程数**，今天能做，要经快照：在换的时刻存一份快照，再用另一个线程数从它建一个新的仿真。花的钱约等于重建一次仿真。在 GPU 模型的 11 个测试程序上，没有哪个程序有长长的一段，换一个线程数能把这笔钱赚回来。所以一次运行挑一个线程数。

## 核对单

- 典型的一拍里有几个实例干活？挨着吗？试过 `Dealt` 吗？
- 8 线程的 *R* 是多少？1、2、4、8 线程各有几个执行单元？
- 有没有一种 task 每拍都跑（或每拍都关着门），却几乎不干活？（多线程时两样都要付。）
- 硬件上带延迟的线，都声明了延迟吗？
- 单线程和最多各跑了一次、只比运行的时间了吗？跳过 2 线程了吗？
