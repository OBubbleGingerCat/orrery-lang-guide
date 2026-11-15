# 8. 跨模块的仲裁

交叉开关、总线、调度器：请求方很多，资源很少，每拍每个资源给一个请求方。仲裁要等这一拍的请求都到齐了才能定，所以它要读很多单元的状态；它怎么拆成 task，既决定它的快慢，也决定加速比。

**规则。** 一个仲裁域（交叉开关的一个子网、一个调度器）**一个 task**。在这个 task 里：

1. 把请求方**走一遍**，记下每个资源有谁要。
2. **只给有人要的资源**挑赢家，各按自己的规则（从指针起轮转、最老的先……）。
3. 放行赢家：弹出它们的队头，推给资源。

不要写成：

- **每个资源一个 task、每个请求方一个 task。** 资源的 task 挑赢家；请求方的 task 再去读每个资源选了谁，才知道自己赢没赢。这就要在 task 之间传 `winner`，一拍里要守一个先后，执行单元也多得多。
- **一个 task 一个资源一个资源地过、每次扫所有请求方。** 那样每拍要资源数 × 请求方数次检查；多线程时，别的都在等这一个 task。

**怎么核对。** 也写一个笨办法的版本，一个资源一个资源地过。两者必须在同样的拍上送出同样的值。笨办法的那一版留作测试。

## 例子：80 × 64 的交叉开关，三种写法

80 个源时不时向 64 个汇中的一个发一个值，经一个 FIFO 进交叉开关。每个输出口从自己的指针起轮转，授予第一个队头要去它那里的输入；被授予的输入弹出队头，口把它推给自己的汇。源和汇每拍还可以干一些自己的活（`work`），代表模型的其余部分。

源和汇：

```rust
const INPUTS: usize = 80;
const PORTS: usize = 64;
const NONE: u8 = u8::MAX;

/// Whether source `s` sends at its `k`-th edge, and where to.
fn send(s: usize, k: u64) -> Option<u32> {
    let h = (k.wrapping_mul(0x9E37_79B9_7F4A_7C15) ^ (s as u64).wrapping_mul(0xC2B2_AE3D_27D4_EB4F)) >> 32;
    (h % 4 == 0).then_some(((h >> 8) % PORTS as u64) as u32)
}

fn fold(sum: u64, value: u32, cycle: u64) -> u64 {
    sum.wrapping_mul(1_000_003).wrapping_add(u64::from(value) << 24 | cycle)
}

/// `rounds` rounds of a xorshift-multiply mix, each depending on the last:
/// work a component does that has nothing to do with the crossbar.
fn busy(state: u64, rounds: u32) -> u64 {
    let mut x = state | 1;
    for _ in 0..std::hint::black_box(rounds) {
        x ^= x >> 29;
        x = x.wrapping_mul(0xBF58_476D_1CE4_E5B9);
    }
    x
}

#[orr::module]
pub struct Source {
    #[orr::param("busy rounds a cycle", default = 0u32)]
    work: u32,
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub id: u8,
    #[orr::interface]
    pub out: orr::FifoTx<u32>,
    k: u64,
    noise: u64,
}

#[orr::module_impl]
impl Source {
    #[orr::task(triggers = (clock,))]
    fn push(
        ctx: &mut orr::TaskCtx,
        id: &'constant u8,
        out: &'latest mut orr::FifoTx<u32>,
        k: &mut u64,
        noise: &mut u64,
    ) -> Result<(), orr::SimError> {
        *noise = busy(*noise, *ctx.module().work);
        *k += 1;
        if let Some(port) = send(usize::from(*id), *k) {
            // The payload: the port, and which source and when.
            let _ = out.push(port | u32::from(*id) << 8 | ((*k as u32) & 0xffff) << 16);
        }
        Ok(())
    }
}

#[orr::module]
pub struct Sink {
    #[orr::param("busy rounds a cycle", default = 0u32)]
    work: u32,
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::interface]
    pub inbox: orr::FifoRx<u32>,
    pub sum: u64,
    noise: u64,
}

#[orr::module_impl]
impl Sink {
    #[orr::task(triggers = (clock,))]
    fn pop(ctx: &mut orr::TaskCtx, inbox: &mut orr::FifoRx<u32>, sum: &mut u64, noise: &mut u64) -> Result<(), orr::SimError> {
        *noise = busy(*noise, *ctx.module().work);
        let clock = ctx.triggered_triggers().0.expect("runs on its clock");
        if let Some(value) = inbox.pop() {
            *sum = fold(*sum, value, clock.cycle_of(ctx.now()));
        }
        Ok(())
    }
}
```

汇用 `clock.cycle_of(ctx.now())` 把拍号折进去；它把时钟的第一个沿算作 1。拍号要从 0 数（规格多半这样数）的，减一。

**每个口一个模块、每个输入一个模块。** 每个口挑出赢家并写下来；每个输入读每个口的赢家（`&'latest`），赢了就弹出队头。第三个模块装着它们，把每个口的赢家共享给每个输入：

```rust
/// The input after `next` whose head goes to `port`, round robin.
fn grant(port: usize, next: &mut u8, heads: impl Fn(usize) -> Option<u32>) -> u8 {
    let first = (usize::from(*next) + 1) % INPUTS;
    let won = (first..INPUTS).chain(0..first).find(|&i| heads(i).is_some_and(|v| (v & 0xff) as usize == port));
    match won {
        Some(i) => {
            *next = i as u8;
            i as u8
        }
        None => NONE,
    }
}

// --- split: a module per port and per input ---

#[orr::module]
pub struct Port {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub port: u8,
    #[orr::reset(INPUTS as u8 - 1)]
    next: u8,
    pub winner: u8,
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub out: orr::FifoTx<u32>,
}

#[orr::module_impl]
impl Port {
    #[orr::task(triggers = (clock,))]
    fn grant(
        _ctx: &mut orr::TaskCtx,
        port: &'constant u8,
        next: &mut u8,
        winner: &mut u8,
        ins: &orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        out: &'latest mut orr::FifoTx<u32>,
    ) -> Result<(), orr::SimError> {
        *winner = NONE;
        if !out.is_full() {
            *winner = grant(usize::from(*port), next, |i| ins.at(i).front().copied());
            if *winner != NONE {
                let value = *ins.at(usize::from(*winner)).front().expect("won with its head");
                out.push(value).expect("checked for room");
            }
        }
        Ok(())
    }
}

#[orr::module]
pub struct Input {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub input: u8,
    pub winners: [orr::StateRef<u8>; 64],
    #[orr::interface]
    pub rx: orr::FifoRx<u32>,
}

#[orr::module_impl]
impl Input {
    #[orr::task(triggers = (clock,))]
    fn pop(_ctx: &mut orr::TaskCtx, input: &'constant u8, winners: &'latest [u8], rx: &mut orr::FifoRx<u32>) -> Result<(), orr::SimError> {
        if let Some(&head) = rx.front() {
            if winners[(head & 0xff) as usize] == *input {
                rx.pop();
            }
        }
        Ok(())
    }
}

#[orr::module]
pub struct Split {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::submodule]
    ports: [Port; 64],
    #[orr::submodule]
    inputs: [Input; 80],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl Split {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        for (p, port) in module.ports.iter().enumerate() {
            ctx.connect(module.clock, port.clock)?;
            ctx.set_const(port.port, p as u8)?;
            for i in 0..INPUTS {
                ctx.share_interface(module.ins[i], port.ins[i])?;
            }
            ctx.share_interface(module.outs[p], port.out)?;
        }
        for (i, input) in module.inputs.iter().enumerate() {
            ctx.connect(module.clock, input.clock)?;
            ctx.set_const(input.input, i as u8)?;
            for p in 0..PORTS {
                ctx.share_state(module.ports[p].winner, input.winners[p])?;
            }
            ctx.share_interface(module.ins[i], input.rx)?;
        }
        Ok(())
    }
}
```

**一个 task，一个口一个口地过。** 对每个口，扫一遍输入：

```rust
// --- one: one task for the crossbar ---

#[orr::module]
pub struct One {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::reset(INPUTS as u8 - 1)]
    next: [u8; 64],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl One {
    #[orr::task(triggers = (clock,))]
    fn cycle(
        _ctx: &mut orr::TaskCtx,
        next: &mut [u8],
        ins: &mut orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        outs: &'latest mut orr::ArrayView<[orr::FifoTx<u32>; 64]>,
    ) -> Result<(), orr::SimError> {
        let mut winners = [NONE; PORTS];
        for (port, winner) in winners.iter_mut().enumerate() {
            if !outs.at(port).is_full() {
                *winner = grant(port, &mut next[port], |i| ins.at(i).front().copied());
                if *winner != NONE {
                    let value = *ins.at(usize::from(*winner)).front().expect("won with its head");
                    outs.at_mut(port).push(value).expect("checked for room");
                }
            }
        }
        for i in 0..INPUTS {
            let head = ins.at(i).front().copied();
            if let Some(head) = head {
                if winners[(head & 0xff) as usize] == i as u8 {
                    ins.at_mut(i).pop();
                }
            }
        }
        Ok(())
    }
}
```

**一个 task，把输入走一遍。** 先记下每个口有哪些输入要（一个位集），再只给有人要的口挑赢家，最后弹出赢家：

```rust
// --- fast: one task, which finds each port's requests once ---

#[orr::module]
pub struct Fast {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::reset(INPUTS as u8 - 1)]
    next: [u8; 64],
    #[orr::interface]
    pub ins: [orr::FifoRx<u32>; 80],
    #[orr::interface]
    pub outs: [orr::FifoTx<u32>; 64],
}

#[orr::module_impl]
impl Fast {
    #[orr::task(triggers = (clock,))]
    fn cycle(
        _ctx: &mut orr::TaskCtx,
        next: &mut [u8],
        ins: &mut orr::ArrayView<[orr::FifoRx<u32>; 80]>,
        outs: &'latest mut orr::ArrayView<[orr::FifoTx<u32>; 64]>,
    ) -> Result<(), orr::SimError> {
        // Which inputs' heads go to each port, as a bit set over inputs.
        let mut requests = [0u128; PORTS];
        for i in 0..INPUTS {
            if let Some(&head) = ins.at(i).front() {
                requests[(head & 0xff) as usize] |= 1 << i;
            }
        }
        let mut granted = 0u128;
        for (port, &asking) in requests.iter().enumerate() {
            if asking == 0 || outs.at(port).is_full() {
                continue;
            }
            // The first asking input after the pointer, round robin.
            let first = (usize::from(next[port]) + 1) % INPUTS;
            let after = asking >> first << first;
            let won = if after != 0 { after.trailing_zeros() } else { asking.trailing_zeros() } as usize;
            next[port] = won as u8;
            granted |= 1 << won;
            let value = *ins.at(won).front().expect("won with its head");
            outs.at_mut(port).push(value).expect("checked for room");
        }
        while granted != 0 {
            let i = granted.trailing_zeros() as usize;
            granted &= granted - 1;
            ins.at_mut(i).pop();
        }
        Ok(())
    }
}
```

顶层和驱动程序，它核对三种写法在同样的拍上送出了同样的值：

```rust
macro_rules! top {
    ($name:ident, $xbar:ident) => {
        #[orr::module]
        pub struct $name {
            #[orr::trigger]
            clock: orr::Clock,
            #[orr::submodule]
            sources: [Source; 80],
            #[orr::submodule]
            sinks: [Sink; 64],
            #[orr::submodule]
            ins: [orr::Fifo<u32>; 80],
            #[orr::submodule]
            outs: [orr::Fifo<u32>; 64],
            #[orr::submodule]
            xbar: $xbar,
        }

        #[orr::module_impl]
        impl $name {
            #[orr::init]
            fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
                let module = ctx.module();
                ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))?;
                ctx.connect(module.clock, module.xbar.clock)?;
                for (s, source) in module.sources.iter().enumerate() {
                    ctx.connect(module.clock, source.clock)?;
                    ctx.set_const(source.id, s as u8)?;
                    ctx.connect_interface(&module.ins[s], source.out)?;
                    ctx.connect_interface(&module.ins[s], module.xbar.ins[s])?;
                }
                for (p, sink) in module.sinks.iter().enumerate() {
                    ctx.connect(module.clock, sink.clock)?;
                    ctx.connect_interface(&module.outs[p], sink.inbox)?;
                    ctx.connect_interface(&module.outs[p], module.xbar.outs[p])?;
                }
                Ok(())
            }
        }
    };
}

top!(SplitTop, Split);
top!(OneTop, One);
top!(FastTop, Fast);

/// A checksum of what the sinks saw.
fn checksum<M: orr::Module>(sim: &orr::Sim<M>, sums: Vec<orr::StateHandle<u64>>) -> Res<u64> {
    let mut check = 0u64;
    for sum in sums {
        check = check.wrapping_mul(31).wrapping_add(*sim.state(sum)?);
    }
    Ok(check)
}

macro_rules! variant {
    ($top:ty, $name:expr, $file:expr, $end:expr) => {
        Case::new(
            $name,
            orr::SimConfig::default(),
            |config| Ok(orr::Sim::new(orr::SimPlan::<$top>::new(Some($file.clone()), None)?, config)?),
            |sim| {
                sim.run_until($end)?;
                let sums = sim.top().sinks.iter().map(|s| s.sum).collect();
                checksum(sim, sums)
            },
        )
    };
}

fn main() -> Res<()> {
    let threads = [1, 2, 4, 8];
    for (work, cycles) in [(0u32, 50_000u64), (300, 5_000)] {
        let file = params(&format!("ch8-work{work}"), &format!("[[top.sources]]\nwork = {work}\n\n[[top.sinks]]\nwork = {work}\n"))?;
        let end = orr::Time::ns(cycles - 1);
        let results = compare(
            &threads,
            3,
            &[
                variant!(SplitTop, format!("a module per port and per input, work {work}"), file, end),
                variant!(OneTop, format!("one task, port by port, work {work}"), file, end),
                variant!(FastTop, format!("one task, one pass over the inputs, work {work}"), file, end),
            ],
        )?;
        let (split, one, fast) = (results[0].1, results[1].1, results[2].1);
        assert!(split == one && one == fast, "the three crossbars delivered differently");
    }
    Ok(())
}
```

## 对三条指标的影响

运行的秒数（粗量），交叉开关周围没有活的（50 000 拍），以及每个源和汇每拍约 0.4 µs 的活的（5 000 拍）：

| | 1 | 2 | 4 | 8 | 8 线程比 1 线程 | 执行单元 |
|---|---|---|---|---|---|---|
| 每个口、每个输入一个模块，周围没活 | 0.429 | 0.459 | 0.243 | 0.173 | 2.48× | 4、4、12、28 |
| 一个 task，一个口一个口地过，周围没活 | 0.436 | 0.458 | 0.466 | 0.462 | 0.94× | 3、3、7、15 |
| 一个 task，走一遍，周围没活 | 0.057 | 0.079 | 0.090 | 0.087 | 0.66× | 3、3、7、15 |
| 每个口、每个输入一个模块，周围有活 | 0.320 | 0.324 | 0.119 | 0.059 | 5.47× | 4、4、12、28 |
| 一个 task，一个口一个口地过，周围有活 | 0.321 | 0.323 | 0.140 | 0.088 | 3.66× | 3、3、7、15 |
| 一个 task，走一遍，周围有活 | 0.283 | 0.285 | 0.102 | 0.051 | 5.59× | 3、3、7、15 |

- **一个 task、走一遍，比每个口、每个输入一个模块更好。**
  - 每个线程数都更快。
  - 周围没活：1 线程快到 7.5 倍，8 线程 2.0 倍。
  - 周围有活：快到 1.13–1.16 倍。
  - 8 线程上快。
- **周围没活时它的加速比低**（0.66），这无关紧要。它把交叉开关变得这么便宜，以至于模型里没有别的东西时，没剩下什么可以分给线程。周围有活时，它的加速比是三者里最高的（5.6）。照[前言](./#when-the-three-disagree)的取舍办法，比时间，不比加速比。
- **一个 task、一个口一个口地过，有快有慢。**
  - 1 线程上和每个口一个模块一样快。
  - 8 线程上：周围没活慢到 2.7 倍，周围有活 1.5 倍。
  - 它在一个 task 里每拍做 64 × 80 次检查，没有哪个线程能分担。
- **读。**
  - 走一遍的仲裁器是一个函数，从头读到尾，task 之间不传 `winner`，也没有先后要守。
  - 每个口、每个输入一个模块的写法要两种模块；
  - 它还要在两种模块之间共享赢家，并守一拍里的先后：输入用 `&'latest` 读赢家。

## GPU 模型里

随 Orrery 附带的 GPU 模型，交叉开关有两个子网：请求一个，回复一个。它以前就是每个口一个模块、每个输入一个模块，像上面那样在两者之间传赢家。现在每个子网一个 task，把输入走一遍。

Orrery 开发方在 11 个测试程序上计时，和每个口一个模块的那一版比（别的都不变）：

- 1 线程：合计 0.923 的时间，每个程序都快；
- 8 线程：合计 0.752 的时间，每个程序都快；
- 执行单元：从 106 个降到 80 个。

交叉开关的代码从约 320 行减到约 240 行，模块从四种减到两种，144 个 `winner` 字段没有了。

## 什么时候不该这样写

- **一个域大到走一遍本身就占了一拍里活的大头。** 照硬件分的地方分开：一个子网一个 task，或者一个调度器一个 task。不要按资源分。
- **几个从不共用请求方的域**，是各自独立的仲裁器：各给一个 task，让它们并排跑。

## 核对单

- 仲裁器看到的是这一拍所有的请求吗（请求方在它之前动，或者它读的是这一拍开始时的值）？
- 指针挪得和硬件一样吗？（和笨办法的版本逐拍比。）
- 每个请求方每拍只看一次吗？
- 每个仲裁域是一个 task 吗？在装载报告里数一下它占了几种 task。
