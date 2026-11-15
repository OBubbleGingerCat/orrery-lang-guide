# 1. 先量

**规则。** 改一个模型的写法时，把旧的写法留着，新旧两种放在驱动程序的同一次运行里轮流量。量四样：

- 1、2、4、8 线程各自跑的时间；
- 建仿真的时间，和跑的时间分开；
- 加速比，即 1 线程时间 ÷ *N* 线程时间；
- 每个线程数下，仿真被分成几个执行单元。

再数改动对可读性的影响。后面每一章都照这个办法量。

## 一个计时帮手

把下面这段放进你的模型的包里（它就是本教程例子的库）。在一个小驱动程序里，每种写法写成一个 `Case`，一起交给一次 `compare`：

```rust
//! The measuring helper of the tutorial's first chapter. Every example
//! uses it: copy it into your own project as it is.
use orrery_lang as orr;
use std::error::Error;
use std::path::PathBuf;
use std::time::Instant;

pub type Res<T> = Result<T, Box<dyn Error>>;

/// Writes a parameter file `params/<name>.toml` and returns its path.
pub fn params(name: &str, text: &str) -> Res<PathBuf> {
    std::fs::create_dir_all("params")?;
    let path = PathBuf::from(format!("params/{name}.toml"));
    std::fs::write(&path, text)?;
    Ok(path)
}

/// One run: seconds to build and to run, execution units, and a checksum
/// of the result.
#[derive(Clone, Copy)]
pub struct Run {
    pub build: f64,
    pub run: f64,
    pub units: usize,
    pub check: u64,
}

/// Builds a simulation with `config` and runs it, timing the two apart.
pub fn run_once<M: orr::Module>(
    config: orr::SimConfig,
    build: &dyn Fn(orr::SimConfig) -> Res<orr::Sim<M>>,
    run: &dyn Fn(&mut orr::Sim<M>) -> Res<u64>,
) -> Res<Run> {
    let start = Instant::now();
    let mut sim = build(config)?;
    let built = start.elapsed().as_secs_f64();
    let units = sim.load_report().units;
    let start = Instant::now();
    let check = run(&mut sim)?;
    let ran = start.elapsed().as_secs_f64();
    Ok(Run { build: built, run: ran, units, check })
}

fn median(mut v: Vec<f64>) -> f64 {
    v.sort_by(f64::total_cmp);
    v[v.len() / 2]
}

/// One way of writing a model, for `compare`: its name, its configuration
/// (the thread count is set by `compare`), how to build it and how to run
/// it, returning a checksum of the result.
pub struct Case<'a> {
    name: String,
    once: Box<dyn Fn(u32) -> Res<Run> + 'a>,
}

impl<'a> Case<'a> {
    pub fn new<M: orr::Module>(
        name: impl Into<String>,
        config: orr::SimConfig,
        build: impl Fn(orr::SimConfig) -> Res<orr::Sim<M>> + 'a,
        run: impl Fn(&mut orr::Sim<M>) -> Res<u64> + 'a,
    ) -> Self {
        let once = move |threads| {
            let config = orr::SimConfig { workers_per_group: threads, ..config.clone() };
            run_once(config, &build, &run)
        };
        Case { name: name.into(), once: Box::new(once) }
    }
}

/// Runs every case on each of `threads`, `reps` times. In each round, on
/// each thread count, every case runs once: the cases and the thread counts
/// take turns, so a slow spell on the machine is shared out among them.
/// Prints, for each case and thread count, the median run and build time,
/// the speedup over the first thread count and the execution units. Fails
/// if a case's checksum changes from run to run: more threads must not
/// change the result. Returns each case's median run times and checksum.
pub fn compare(threads: &[u32], reps: usize, cases: &[Case]) -> Res<Vec<(Vec<f64>, u64)>> {
    let mut runs: Vec<Vec<Vec<Run>>> = vec![vec![Vec::new(); threads.len()]; cases.len()];
    for _ in 0..reps {
        for (t, &n) in threads.iter().enumerate() {
            for (c, case) in cases.iter().enumerate() {
                runs[c][t].push((case.once)(n)?);
            }
        }
    }
    let mut results = Vec::new();
    for (case, runs) in cases.iter().zip(&runs) {
        let check = runs[0][0].check;
        if let Some(r) = runs.iter().flatten().find(|r| r.check != check) {
            return Err(format!("{}: checksum {:016x}, not {check:016x}", case.name, r.check).into());
        }
        let medians: Vec<f64> = runs.iter().map(|r| median(r.iter().map(|x| x.run).collect())).collect();
        for (t, &n) in threads.iter().enumerate() {
            let build = median(runs[t].iter().map(|x| x.build).collect());
            println!(
                "{}: {n} thread(s): run {:.4} s, {:.2}x of {} thread(s), build {build:.4} s, {} execution unit(s), checksum {check:016x}",
                case.name,
                medians[t],
                medians[0] / medians[t],
                threads[0],
                runs[t][0].units,
            );
        }
        results.push((medians, check));
    }
    Ok(results)
}

/// A mixer: `rounds` rounds of MurmurHash3's finalizer, the stand-in for
/// a unit's work in a cycle.
pub fn mix(mut x: u64, rounds: u64) -> u64 {
    for _ in 0..rounds {
        x ^= x >> 33;
        x = x.wrapping_mul(0xff51_afd7_ed55_8ccd);
        x ^= x >> 29;
    }
    x
}
```

它做什么、为什么：

- **建仿真和跑分开计时。** 建仿真是一笔固定的开销，多开线程也不会变少；8 线程时通常还比 1 线程长。把它分开计，不然一段很短的多线程运行看起来会比实际差。只有建仿真占得很少时，才拿整个进程来比。
- **每种写法、每个线程数跑三次，取中位数。** 每一轮里各写法、各线程数轮流跑，机器上一阵慢会摊到每一格上。相差 10% 以内算抖动。
- **几种写法在同一次运行里比。** 不要拿分开的几次运行的数来比：换一个进程、换一个时候、也许换了几个核，快慢能差得比你要找的差别还大。在一个有 64 和 256 个互不相干的单元的模型上，分开跑的两次让一种写法在 8 线程上看着慢了四分之一到三分之一；放进同一个进程里轮流跑，两种一样快。
- **打出执行单元数**（`sim.load_report().units`）。*N* 线程的运行里，一个线程派活，*N* − 1 个跑 task，所以同时最多跑 *N* − 1 个执行单元。执行单元比 worker 少时，建仿真还会警告：`1 execution unit for 7 workers: the same as serial`。
- **核一个结果的校验和。** `run` 这个闭包返回一个数，概括模型最后的状态。多开线程不能改变结果；同一件事换一种更好的写法，也不能改变结果。校验和是和运行一起计时的，所以要便宜：把最后状态里的几个数折起来，不要把整份输出拼成文字。

挑一段在 1 线程上至少跑约 0.1 s 的输入。输入是给定的、又更短时，每种多跑几遍（`compare` 的 `reps`），小的差别照样当噪声看。运行要在整个模型满足某个条件时结束（包都到齐了）的，驱动程序可以每次 `run_until` 跑几千拍，两次之间查这个条件；要让一个 task 返回 `orr::Stop`，它得自己看得到整个条件（指南：[停止](../guide/running#stopping)）。在至少有 8 个空闲核的机器上计时：8 线程的运行要占 8 个核，这些核上有别的活，会让它慢得不均匀。

## 例子：每条通道一个模块，还是一个循环

64 条通道，每条每拍把自己的状态混合一遍。写成每条通道一个模块实例：

```rust
/// A lane: `work` mixer rounds on its own state every cycle.
#[orr::module]
pub struct Lane {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub id: u64,
    #[orr::constant]
    pub work: u64,
    pub state: u64,
}

#[orr::module_impl]
impl Lane {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, id: &'constant u64, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
        *state = mix(*state ^ *id, *work);
        Ok(())
    }
}

/// `n` lanes, one instance each.
#[orr::module]
pub struct Lanes {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("mixer rounds a lane does in a cycle", default = 10u64)]
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
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        for (i, lane) in module.lanes.iter().enumerate() {
            ctx.connect(module.clock, lane.clock)?;
            ctx.set_const(lane.id, i as u64)?;
            ctx.set_const(lane.work, *module.work)?;
        }
        Ok(())
    }
}
```

`id` 和 `work` 是**常量**：标了 `#[orr::constant]` 的字段。父模块在自己的 `init` 里用 `ctx.set_const` 给每个赋一次值，所以字段要 `pub`。task 里写成 `&'constant u64` 来读。和参数不同，每个实例可以有自己的值；和状态不同，读它不会让一个 task 排在另一个后面。规则见 API 文档的 `ConstHandle`。

写成一个模块，它唯一的 task 循环走遍所有通道：

```rust
/// The same `n` lanes in one module, one task looping over them.
#[orr::module]
pub struct Loop {
    #[orr::param("lanes", default = 64usize)]
    n: usize,
    #[orr::param("mixer rounds a lane does in a cycle", default = 10u64)]
    work: u64,
    #[orr::trigger]
    clock: orr::Clock,
    pub states: [u64; MAX_LANES],
}

#[orr::module_impl]
impl Loop {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        assert!(*module.n <= MAX_LANES);
        ctx.init_clock(module.clock, orr::Time::ps(PERIOD_PS), orr::Time::ps(0))?;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn cycle(ctx: &mut orr::TaskCtx, states: &mut [u64]) -> Result<(), orr::SimError> {
        let (n, work) = (*ctx.module().n, *ctx.module().work);
        for (i, state) in states[..n].iter_mut().enumerate() {
            *state = mix(*state ^ i as u64, work);
        }
        Ok(())
    }
}
```

驱动程序把两种放在一起比，活轻的（10 轮，每条通道每拍约 10 ns）和活重的（1000 轮，约 2 µs）各一次：

```rust
fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(10u64, 100_000u64), (1000, 2_000)] {
        let file = params(&format!("ch1-work{work}"), &format!("[top]\nn = 64\nwork = {work}\n"))?;
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        let config = orr::SimConfig::default();
        let lanes = Case::new(
            format!("lanes, work {work}"),
            config.clone(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                let mut sum = 0;
                for lane in sim.top().lanes.iter() {
                    sum = mix(sum ^ *sim.state(lane.state)?, 1);
                }
                Ok(sum)
            },
        );
        let one_loop = Case::new(
            format!("loop, work {work}"),
            config,
            |config| Ok(orr::Sim::new(orr::SimPlan::<Loop>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                let mut sum = 0;
                for &state in sim.top().states.iter().take(64) {
                    sum = mix(sum ^ *sim.state(state)?, 1);
                }
                Ok(sum)
            },
        );
        // The two ways take turns, so they meet the same machine.
        let results = compare(&threads, 3, &[lanes, one_loop])?;
        assert_eq!(results[0].1, results[1].1, "the two ways computed different states");
    }
    Ok(())
}
```

运行的秒数（粗量），以及 1、2、4、8 线程的执行单元数：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 | 执行单元 |
|---|---|---|---|---|---|---|
| 每条通道一个模块，活轻 | 0.067 | 0.078 | 0.054 | 0.041 | 1.62× | 1、1、3、7 |
| 一个循环，活轻 | 0.059 | 0.073 | 0.102 | 0.108 | 0.55× | 1、1、1、1 |
| 每条通道一个模块，活重 | 0.242 | 0.243 | 0.086 | 0.041 | 5.98× | 1、1、3、7 |
| 一个循环，活重 | 0.242 | 0.242 | 0.243 | 0.243 | 0.99× | 1、1、1、1 |

表里看得出：

- **1 线程上，循环一样快或略快。** 调一次 task 花得少，但不是不花。活重时两种一样；活轻时，一拍调 64 次比调一次、里面循环 64 次多花 13%。只量 1 线程，你会选循环。
- **8 线程上差得很远。** 每条通道一个模块：活轻时 8 线程是 1 线程的 1.6 倍快，活重时 6.0 倍。一个循环只有一个执行单元，多开线程也分不开；活轻时，多出来的线程只多付了派活的钱，8 线程的时间是 1 线程的 1.8 倍。
- **执行单元数事先就说了。** 循环在每个线程数上都只有一个执行单元，在 4、8 线程上建仿真时还会警告。
- **2 线程不比 1 线程快。** 两个线程里一个派活，只有另一个跑 task。从 4 线程起才有得快。

照[前言](./#when-the-three-disagree)的取舍办法：

- **活重时，每条通道一个模块更好。** 没有哪个线程数变慢，4、8 线程更快；8 线程上快。
- **活轻时，有快有慢。** 1 线程慢 13%；4、8 线程快到 1.9 倍和 2.6 倍。按你要跑的线程数选：跑 4 或 8 线程，就每条通道一个模块。

每条通道一个模块，读起来也像硬件：一个单元一个模块。

早一版的驱动程序把两种写法先后各量一遍，1 线程上两种一样快。轮流量，四次运行里循环每次都快。

## 数可读性

在改的那段代码上，改前、改后各数三样：

1. **行数。** 数模块和它的 task，不数驱动程序。
2. **读懂一个单元要同时记住几样东西。** 数这几样：
   - 这个单元的 task；
   - 它的门，以及每扇门在什么条件下关；
   - 谁叫醒它、从哪里叫；
   - 它的 task 和别的 task 在一拍里靠什么排先后（访问方式、delta 层）。
   
   每一样都是读的人要去找、要记住的。
3. **一个硬件单元的行为是不是在一处看得全**：在一个模块的一个 task 里，还是散在几个 task、几个文件里？

上面的通道只有一个 task，没有门、没有叫醒、没有要排的先后；它的行为就一行，在一处。

## 找出时间花在哪

模型比你预想的慢时：

- **数调用次数。** 给一个 task 的状态加一个计数，每调一次加一；再加一个，数真正干了活的调用。跑完用 `sim.state` 读出来。门那两章（[第 2 章](./gates)、[第 3 章](./due)）讲的就是那些什么也不干的调用。用 `Reports::All`（见下）时，每次停下来的报告给出每个门跳过的沿（`gated`）：只由时钟触发的 task，在其余的沿上都被调了，不用加计数。
- **算出每拍的时间。** 1 线程的运行时间 ÷ 拍数。再拿它和线程的开销比（[第 10 章](./threads)）。
- **给一个不用 Orrery 的直译版计时。** 照硬件一拍一拍写一个普通的 Rust 循环，不用 Orrery；它也能用来核模型的结果。它 1 线程每拍的时间，大致就是活本身的价钱；模型多出来的，是调用和门的价钱。
- **看报告。** `SimConfig { reports: orr::Reports::All, .. }` 会打出装载报告，以及每次停下来的报告。
- **看有没有 task 走了慢路。** 装载报告里 `counts.fits.generic` 列出走较慢的通用路径的 task。那里写 `CapDiffers`，是说这个 task 的各个实例碰到的 FIFO 或队列容量不一样：有一个 FIFO 比别的深就够了。把它们都设成一样的容量。考题的网络里，有一级 FIFO 深 4、别的深 2，1 线程就慢了 1.2 到 3.4 倍，结果一样，也没有警告。

## 核对单

- 旧的写法留着了吗？新旧两种是在同一次运行里轮流量的吗？
- 1、2、4、8 线程都量了吗？每个三次？
- 建仿真和跑分开计时了吗？
- 算了加速比吗？记下每个线程数的执行单元数了吗？
- 每个线程数的结果都一样吗（校验和）？
- 改前、改后都数了可读性的三个数吗？
- 给这个改动记下"更好"还是"有快有慢"，以及它在 8 线程上怎样了吗？
