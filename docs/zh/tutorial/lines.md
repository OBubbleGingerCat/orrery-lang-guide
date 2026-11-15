# 5. 连线和延迟

单元之间经连线传东西：事件、FIFO、共享的状态。一条线怎么写，决定两件事：一拍里两端谁先跑；以及两端能不能在不同线程上同时跑。

**规则。**

1. **硬件上带延迟的线，在模型里也带上这个延迟，并声明出来。** 一根导线、一条链路、两级之间的流水寄存器，至少要一拍。写成带 `#[orr::trigger(min_delay = ..)]` 的事件，或者 `orr::Due` 值的 FIFO（[第 3 章](./due)）。
   - 这样两端在同一拍里不会互为因果，可以并排跑。
   - 不带延迟的线，接收端在同一拍、晚一层 delta 上，在发送端之后跑。这样的线连成一串，就是一串 delta 层，一层等一层。
2. **一拍里的先后用访问方式说，不用 delta 层。**
   - 用 `&` 读状态的 task，看到的是这一拍开始时的值，在任何 task 写它之前。`&'latest` 看到的是这一拍的写者写过之后的值。
   - 访问方式写在参数上，读的人当场看得见。delta 层（`clock + delta(k)`）是一个全模型都要统一的数。只在访问方式说不清的地方用 delta 层，比如[第 4 章](./wakes)的第三种配对。
3. **两个互相送东西的单元，每个要两个 task，或者一条带延迟的线。** 两个方向各一个 FIFO、每个单元一个 task，先后就成了环，建仿真时报错。给每个单元一个收的 task、一个送的 task；或者链路在硬件上本来就有延迟，就用带 `min_delay` 的事件：它们在一拍里没有先后，每个单元一个 task 就够了（[两个互相送东西的单元](../guide/components#who-acts-first)）。

每种线的细节见指南：[声明的延迟](../guide/time#declared-delays)、[delta 周期](../guide/time#delta-cycles)、[谁先动作](../guide/components#who-acts-first)。

## 例子：一排排的级，带延迟和不带延迟

8 条通道，每条 8 级。每条通道的头在时钟上跑；后面每一级，在前一级把状态传过来时跑，把它混进自己的状态，再传下去。同一段代码用一个宏建两遍：

- 一遍每条线延迟一拍，并声明（`min_delay`）；
- 一遍不带延迟（`max_delay = DELTA`，每个事件晚一个 delta 周期送出）。

```rust
macro_rules! lanes {
    ($link:expr; $($bounds:tt)*) => {
        #[orr::module]
        pub struct Head {
            #[orr::trigger]
            pub clock: orr::ClockRelay,
            #[orr::trigger($($bounds)*)]
            pub out: orr::EventSource<u64>,
            #[orr::constant]
            pub work: u64,
            pub state: u64,
        }

        #[orr::module_impl]
        impl Head {
            #[orr::task(triggers = (clock,), emits = (out,))]
            fn cycle(ctx: &mut orr::TaskCtx, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
                *state = $crate::mix(*state ^ 1, *work);
                ctx.emit_triggers().0.emit(*state, $link)
            }
        }

        #[orr::module]
        pub struct Stage {
            #[orr::trigger]
            pub input: orr::EventRelay<u64>,
            #[orr::trigger($($bounds)*)]
            pub out: orr::EventSource<u64>,
            #[orr::constant]
            pub work: u64,
            pub state: u64,
        }

        #[orr::module_impl]
        impl Stage {
            #[orr::task(triggers = (input,), emits = (out,))]
            fn cycle(ctx: &mut orr::TaskCtx, work: &'constant u64, state: &mut u64) -> Result<(), orr::SimError> {
                let input = ctx.triggered_triggers().0.copied().unwrap_or(0);
                *state = $crate::mix(*state ^ input, *work);
                ctx.emit_triggers().0.emit(*state, $link)
            }
        }

        /// `lanes` lanes of a head and `stages - 1` later stages.
        #[orr::module]
        pub struct Lanes {
            #[orr::param("lanes", default = 8usize)]
            lanes: usize,
            #[orr::param("later stages, lanes x (stages a lane - 1)", default = 56usize)]
            rest: usize,
            #[orr::param("mixer rounds a stage does in a cycle", default = 100u64)]
            work: u64,
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            heads: [Head; orr::DynShape(lanes)],
            #[orr::submodule]
            stages: [Stage; orr::DynShape(rest)],
        }

        #[orr::module_impl]
        impl Lanes {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                let per_lane = *module.rest / *module.lanes;
                ctx.init_clock(module.clock, orr::Time::ps($crate::PERIOD_PS), orr::Time::ps(0))?;
                for (l, head) in module.heads.iter().enumerate() {
                    ctx.connect(module.clock, head.clock)?;
                    ctx.set_const(head.work, *module.work)?;
                    let mut previous = head.out;
                    for stage in &module.stages[l * per_lane..(l + 1) * per_lane] {
                        ctx.connect(previous, stage.input)?;
                        ctx.set_const(stage.work, *module.work)?;
                        previous = stage.out;
                    }
                }
                Ok(())
            }
        }

        /// A checksum of every stage's state.
        pub fn checksum(sim: &orr::Sim<Lanes>) -> $crate::Res<u64> {
            let mut sum = 0;
            for head in sim.top().heads.iter() {
                sum = $crate::mix(sum ^ *sim.state(head.state)?, 1);
            }
            for stage in sim.top().stages.iter() {
                sum = $crate::mix(sum ^ *sim.state(stage.state)?, 1);
            }
            Ok(sum)
        }
    };
}

/// Lines with a delay of one cycle, declared as their minimum.
pub mod timed {
    use orrery_lang as orr;
    lanes!(orr::Time::ps(super::PERIOD_PS); min_delay = orr::Time::ps(super::PERIOD_PS));
}

/// Lines without a delay: the next stage runs in the same cycle.
pub mod untimed {
    use orrery_lang as orr;
    lanes!(orr::Time::DELTA; max_delay = orr::Time::DELTA);
}
```

驱动程序把两种都量一遍，活轻的（每级 10 轮，约 10 ns）和活重一些的（100 轮，约 0.2 µs）各一次：

```rust
fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(10u64, 50_000u64), (100, 10_000)] {
        let file = params(&format!("ch5-work{work}"), &format!("[top]\nwork = {work}\n"))?;
        let end = orr::Time::ps(PERIOD_PS * cycles - 1);
        // Two different models: their checksums differ, and that is right.
        let timed = Case::new(
            format!("8 lanes x 8 stages, a delay of one cycle, work {work}"),
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<timed::Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                timed::checksum(sim)
            },
        );
        let untimed = Case::new(
            format!("8 lanes x 8 stages, no delay, work {work}"),
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<untimed::Lanes>::new(Some(file.clone()), None)?, config)?),
            |sim| {
                sim.run_until(end)?;
                untimed::checksum(sim)
            },
        );
        compare(&threads, 3, &[timed, untimed])?;
    }
    Ok(())
}
```

这两个是不同的模型：带延迟的，一个值从一级到下一级要一拍。它们的校验和不同，这是对的。两者相同的是活：64 级，每级每拍混合一次。

## 对三条指标的影响

运行的秒数（粗量），以及 1、2、4、8 线程的执行单元数：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 | 执行单元 |
|---|---|---|---|---|---|---|
| 延迟一拍，活轻 | 0.186 | 0.202 | 0.110 | 0.078 | 2.37× | 2、2、6、14 |
| 不带延迟，活轻 | 0.227 | 0.239 | 0.241 | 0.233 | 0.97× | 2、2、24、56 |
| 延迟一拍，活重一些 | 0.147 | 0.149 | 0.059 | 0.033 | 4.47× | 2、2、6、14 |
| 不带延迟，活重一些 | 0.155 | 0.158 | 0.084 | 0.060 | 2.58× | 2、2、24、56 |

- **声明了延迟，各级就并排跑。**
  - 8 线程、活轻：是不带延迟的 3.0 倍快。
  - 活重一些：1.8 倍快。
  - 不带延迟、活轻时，多开线程也不快：8 线程和 1 线程一样长。
- **执行单元数说明了原因。**
  - 不带延迟时，一级必须在喂它的那一级之后、同一拍里跑。所以每一层自成一步：先是各条的头，再是各条的第二级，依此类推。八层每一层都分到 7 个 worker 上：8 线程时 8 × 7 = 56 个执行单元。
  - 带延迟时只有两种 task，头和后面的级：2 × 7 = 14 个。
  - 每个执行单元每拍都要花钱（[第 10 章](./threads)）。
- **1 线程上**，活轻时带延迟的也快五分之一；活重一些时两者一样。
- **读**：声明的延迟在线上说出了硬件做的事："这要一拍"。不带延迟的线，要读的人自己推两端的先后。

## 什么时候不该这样写

- **不要加硬件上没有的延迟。** 延迟改变事情发生的时间，也就改变了结果。
- **也不要为了对上参考模型去加。** 要和参考模型逐拍对上的模型，一拍里的先后照参考模型的来，常常是"这一拍写的值"或"这一拍空出来的位子"。在那里加一拍，就和参考模型对不上了。
- 所以这条是写模型时用的：硬件上的线有延迟，就声明出来。它不是给已经对上参考的模型提速的办法。
- **声明真实的最小值。** 声明得比硬件大是错的；声明得小是允许的，但放弃了一部分并行。

## 核对单

- 硬件上要花时间的每条线，都声明了延迟吗（`min_delay`，或 `Due` 值的 FIFO）？
- 一拍里的每个先后，都用访问方式（`&`、`&'latest`）说了吗？每个 delta 层，都能写出它为什么在那里吗？
- 两个互相送东西的单元，各有一个收的 task 和一个送的 task，或者一条带延迟的线吗？
- 改前、改后比过执行单元数吗？执行单元数是不是随不带延迟的线串变多？
