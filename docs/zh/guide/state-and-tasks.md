# 状态与 task

task 是模块的 `#[orr::module_impl]` 块里的一个函数，由模块的触发源运行。它读写模块的状态；它怎样声明每一份状态，就说明了在同一个时刻点里，它与碰这份状态的其他 task 谁先谁后。

## 一个 task

```rust
use orrery_lang as orr;

#[orr::task(triggers = (clock,), emits = (done,))]
fn produce(ctx: &mut orr::TaskCtx, value: &mut u64) -> Result<(), orr::SimError> {
    *value += *ctx.module().step;
    ctx.emit_triggers().0.emit(*value, orr::Time::DELTA)?;
    Ok(())
}
```

- `triggers = (..)` 列出运行它的触发源（其中任何一个触发都会运行它，每个时刻点、每个实例最多一次）；`emits = (..)` 列出它可能发往的事件源，`wakes = (..)` 列出它可能用的[唤醒](./time#wakes)。每一项可以是一个字段、表示单个元素的 `field[i]`，或者一整个数组。
- 第一个参数是 `ctx: &mut orr::TaskCtx`；后面是状态参数，顺序随意。
- 它返回 `Result<(), orr::SimError>`；如果它可能结束仿真，则返回
  `Result<(), orr::Stop>`（[运行仿真](./running#stopping)）。

## 访问模式 {#access-modes}

状态参数绑定同名的状态字段，它的引用类型就是它的**访问模式**：

| 参数 | 模式 | 看到的值 |
|---|---|---|
| `x: &T` | `curr` | 本时刻点的写者写之前的值 |
| `x: &'latest T` | `latest` | 本时刻点的写者写之后的值 |
| `x: &mut T` | `write` | 它就是写者：读写这个值 |

task 还可以读本模块的[常量](./modules#constants)，写成 `name: &'constant T`。常量不是状态：它没有访问模式，也不决定谁先谁后。

每个位置在每个时刻点最多有一个写者。在一个时刻点之内，每个 `curr` 读者都在写者之前运行，每个 `latest` 读者都在写者之后运行，与 task 声明的先后无关。如果一个时刻点里没有谁写它，两种读者看到的值相同。

```rust
#[orr::module]
pub struct Stage {
    #[orr::trigger]
    clock: orr::Clock,

    reg: u64,
    pub seen_before: u64,
    pub seen_after: u64,
}

#[orr::module_impl]
impl Stage {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))
    }

    #[orr::task(triggers = (clock,))]
    fn advance(_ctx: &mut orr::TaskCtx, reg: &mut u64) -> Result<(), orr::SimError> {
        *reg += 1;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn before(_ctx: &mut orr::TaskCtx, reg: &u64, seen_before: &mut u64) -> Result<(), orr::SimError> {
        *seen_before = *reg;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn after(_ctx: &mut orr::TaskCtx, reg: &'latest u64, seen_after: &mut u64) -> Result<(), orr::SimError> {
        *seen_after = *reg;
        Ok(())
    }
}
```

经过五个时钟沿（0 到 4 ns）之后，`reg` 是 5，`seen_before` 是 4，`seen_after`
是 5，不论用几个线程。

寄存器和连线就是这样建模的：用 `&` 读流水线寄存器，得到上一个周期的值；用
`&'latest` 读组合信号，得到这个周期的值。

`&` 读到的不是上一个周期另存下来的一份，而是让读的 task 排在这个时刻点的写者之前。所以一串寄存器、每个由后一级来读，可以拆成每级一个 task：读的一方总能先动。两个寄存器互相读对方、又各由自己的 task 写，就不行了：两个 task 都得排在对方前面，`Sim::new` 报依赖环：

```text
dependency cycle (triggered by clock top.clock (1ns phase 0ps)): %Pair.step_a at top reads top.b (curr) before %Pair.step_b at top writes it; %Pair.step_b at top reads top.a (curr) before %Pair.step_a at top writes it
```

把两个写进一个 task，先读出两个旧值再写：

```rust
#[orr::task(triggers = (clock,))]
fn step(_ctx: &mut orr::TaskCtx, a: &mut u64, b: &mut u64) -> Result<(), orr::SimError> {
    let (old_a, old_b) = (*a, *b);
    *a = old_b + 1;
    *b = old_a * 10;
    Ok(())
}
```

或者把每个寄存器的下一个值放进它自己的字段，再由一个用 `&'latest` 读它的 task 写回去：

```rust
#[orr::task(triggers = (clock,))]
fn step_a(_ctx: &mut orr::TaskCtx, a_next: &mut u64, b: &u64) -> Result<(), orr::SimError> {
    *a_next = *b + 1;
    Ok(())
}

#[orr::task(triggers = (clock,))]
fn update_a(_ctx: &mut orr::TaskCtx, a: &mut u64, a_next: &'latest u64) -> Result<(), orr::SimError> {
    *a = *a_next;
    Ok(())
}

// `step_b`、`update_b` 照样写
```

两种写法在 0、1、2 ns 三个沿之后都是 `a` 为 11、`b` 为 10，和两个触发器一样，不论用几个线程。

## 绑定其他字段 {#binding-other-fields}

`#[orr::bind(field)]` 把参数绑定到另一个名字的字段上，`#[orr::bind(field[3])]`
则绑定到数组的一个元素上：

```rust
fn tick(_ctx: &mut orr::TaskCtx,
        #[orr::bind(values[1])] second: &mut u64) -> Result<(), orr::SimError>
```

整个数组是 `&[T]`、`&'latest [T]` 或 `&mut [T]`。数组的一部分用
`orr::PartialAccess`，下标或范围写成字面量；这里是第 0 和第 2 个元素：

```rust
#[orr::bind(values)] even: &mut [u64; orr::PartialAccess(0, 2)],
```

| 写法 | 绑到的元素 |
|---|---|
| `PartialAccess(3)` | 第 3 个 |
| `PartialAccess(0, 2)` | 第 0、第 2 个，不是前两个 |
| `PartialAccess(0..2)` | 第 0、1 个：半开区间 |
| `PartialAccess(3, 1..3)` | 依次是第 3、1、2 个，照写的次序 |

下标和区间的上下界都是整数字面量，不能是常量、参数或表达式（`N`、`2 * K`）。区间只能写成 `start..end`，而且不能是空的（`1..=3`、`..3`、`2..` 都是错误）；下标不能重复，区间不能重叠，空的 `PartialAccess()` 是错误。

plain 值的数组也可以当视图取，`&orr::ArrayView<[T]>`、`&'latest orr::ArrayView<[T]>` 或 `&mut orr::ArrayView<[T]>`，可以是定长的（`[T; 3]`），也可以是一个选择（`[T; orr::PartialAccess(2, 0)]`，元素按选择的次序），方法和[视图](./components#arrays-and-forwarding)一样。它的元素就是状态本身，原地读写。元素不是连成一段的值时这一点才要紧：`StateRef` 数组的元素是别的模块的状态，常常不连成一段；跳着选、换了次序的选择也不连成一段。这样的元素做成切片是拷贝，`&mut` 切片的改动只在 task 返回 `Ok` 时才拷回去；视图的写马上落下，task 之后失败也留着。一个统计四个端口授权的模块，每个 `StateRef` 由父模块用 `ctx.share_state` 指到一个端口的 `grant`：

```rust
#[orr::module]
pub struct Tally {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    pub grants: [orr::StateRef<u32>; 4],
    credits: [u32; 4],
    pub total: u64,
}

#[orr::module_impl]
impl Tally {
    #[orr::task(triggers = (clock,))]
    fn tally(
        _ctx: &mut orr::TaskCtx,
        grants: &orr::ArrayView<[u32]>,
        credits: &mut [u32],
        total: &mut u64,
    ) -> Result<(), orr::SimError> {
        for (credit, grant) in credits.iter_mut().zip(grants.iter()) {
            *credit += *grant;
            *total += u64::from(*grant);
        }
        Ok(())
    }
}
```

对视图写的函数（[对数组写函数](./components#functions-over-arrays)）收状态的视图，和收接口的视图一样：`u32` 是 `orr::ArrayElem`。

容器字段用它自己的类型，或者它加载时的形式：
`&mut orr::Queue<Req, orr::DynShape(depth)>` 在 task 里是
`&mut orr::DynQueue<Req>`。

加载时定容量的容器组成的数组，整个拿的时候是一个视图 `orr::ArrayView`，照字段原样写：
`buffers: &mut orr::ArrayView<[orr::Queue<Req, orr::DynShape(depth)>]>`。
`buffers.at(i)` 读第 *i* 个，`buffers.at_mut(i)` 写它，拿到的都是 `DynQueue`；`&`、`&'latest` 是只读的。只要其中一个元素，像上面那样用 `#[orr::bind(buffers[2])]`。

参数写的类型必须就是字段的类型；不匹配编译不过："a task parameter of type `u32` cannot bind a field of type `u64`"。

## 一组列 {#columns}

一个单元的状态被几个 task 一个字段一个字段地共用（一个核的计数、额度、标志）时，可以把这些字段放成**一组列**：task 用一个参数拿到整组，不用每个字段一个参数；对程序的其余部分，每个成员仍是一个单独的字段。

- 一组列是一个有名成员的结构体，每个成员是定长的 plain 类型的值，加 `#[derive(orr::Columns)]`。成员的初值是它类型的默认值，或者按成员上写的 `#[reset(value)]`、`#[reset(with = f)]`。
- 模块里有一个这一组的字段，标 `#[orr::columns]`。
- task 用一个参数绑定整个字段：`&mut Core` 写它，`&Core` 读时刻点开始时的值，`&'latest Core` 读这个时刻点写过以后的值。不能写 `&'latest mut Core`。
- task 里用 `.cols()` 拿到每个成员一个句柄，名字和成员一样，解引用就是它的值：`*c.credits -= 1`。`&mut` 的一组，`cols()` 借用参数，所以参数要写成 `mut core: &mut Core`；`core.reborrow()` 把它借给一个函数、自己还留着，`core.into_cols()` 把它拿走。
- task 调用的函数取 `orr::ColsMut<'_, Core>`（写）或 `orr::ColsRef<'_, Core>`（读）。
- 只用到几个成员的 task，可以每个成员单独绑定：`#[orr::bind(core.issued)] issued: &'latest u64`。

```rust
#[derive(orr::Columns)]
pub struct Core {
    pub issued: u64,
    #[reset(4)]
    pub credits: u32,
    pub stalled: bool,
}

fn issue(core: orr::ColsMut<'_, Core>) {
    let mut c = core.into_cols();
    if *c.credits > 0 {
        *c.credits -= 1;
        *c.issued += 1;
    }
}

#[orr::module]
pub struct Sm {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::columns]
    pub core: Core,
    pub idle: u64,
    pub seen: u64,
}

#[orr::module_impl]
impl Sm {
    #[orr::task(triggers = (clock,))]
    fn cycle(_ctx: &mut orr::TaskCtx, mut core: &mut Core) -> Result<(), orr::SimError> {
        issue(core.reborrow());
        let mut c = core.cols();
        *c.stalled = *c.credits == 0;
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn count(_ctx: &mut orr::TaskCtx, core: &'latest Core, idle: &mut u64) -> Result<(), orr::SimError> {
        if *core.cols().stalled {
            *idle += 1;
        }
        Ok(())
    }

    #[orr::task(triggers = (clock,))]
    fn watch(
        _ctx: &mut orr::TaskCtx,
        #[orr::bind(core.issued)] issued: &'latest u64,
        seen: &mut u64,
    ) -> Result<(), orr::SimError> {
        *seen = *issued;
        Ok(())
    }
}
```

十个沿以后，`issued` 是 4，`credits` 是 0，`idle` 是 7，`seen` 是 4，线程数多少都一样。

每个成员是单独的一列：计划按 task 写和读的成员给它们排先后，宿主按名字读一个成员：`sim.state(sm.core.issued)`。一组列不允许的：

- 装载时定大小的成员、门控、接口、外部端口编不过；编译时定容量的容器（如 `orr::ArrayVec<T, N>`）在建计划时被拒绝；
- 写这一组的 task 只能用那一个参数绑定它：同一个字段、或它的某个成员再用第二个参数绑定，编不过；
- 单独绑定的成员要整个绑定（`core.issued`），不能只绑它的一部分；
- 一组列的字段不能是数组，也不能整体复位。

## 同一个 task 之内

一个 task 的两个参数可以指向同一个位置（比如经由引用或接口），但只有两者都是
`&`，或者都是 `&'latest` 时才行。其他情况在构建 plan 时报错，因为一个 task
不能既在写者之前、又在写者之后运行。

用来改的视图一次借出一个元素：`at_mut(i)` 取出的元素还在用时，视图不再给别的元素，要别的就编译不过。要几个元素一起处理，用 `iter_mut()` 依次走过，每个给一次。元素里的接口不能和别的互换，也不能挪出来；要改 task 服务各输入的次序，把次序作为下标存在状态里，照着走（[数组与转发](./components#arrays-and-forwarding)）。

## task 上下文

| 方法 | 给出 |
|---|---|
| `ctx.module()` | 模块的视图：参数是值（`*ctx.module().step`） |
| `ctx.now()` | 当前的 `orr::Timepoint`（`.fs()`、`.ps()`、`.delta()`，显示为 `10ns` 或 `10ns+1`） |
| `ctx.triggered_triggers()` | 一个元组，每个 `triggers` 项一个槽位：字段是 `Option<&P>`，数组是 `&[Option<&P>]`；时钟的载荷是它的 `ClockInfo` |
| `ctx.emit_triggers()` | 一个元组，每个 `emits` 项一个发送器：一个 `orr::EventEmitter`，数组则是一个 `orr::EventEmitters` |
| `ctx.wakes()` | 一个元组，每个 `wakes` 项一个：一个 `orr::Waker`，数组则是一个 `orr::Wakers`（[唤醒](./time#wakes)） |

```rust
let (clock, inputs) = ctx.triggered_triggers();
for packet in inputs.iter().flatten() {
    // 这个时刻点在任一输入上到达的每个包
}
let outputs = &mut ctx.emit_triggers().0;
outputs[port].emit(packet, link_delay)?; // 按下标取一个元素
for (i, mut output) in outputs.iter().enumerate() {
    output.emit(i as u32, orr::Time::ns(i as u64 + 1))?; // 每个元素
}
if let Some(mut output) = outputs.get(9) {
    output.emit(packet, link_delay)?; // `None`：没有第 9 个元素
}
```

`EventEmitters` 用 `outputs[i]` 给出第 *i* 个元素的发送器，越界时 panic；用 `outputs.get(i)` 则越界时给出 `None`。`iter()` 按顺序给出每个元素的发送器，`len()` 给出元素个数。

发送器和载荷只在这次调用期间借用。发送事件见[时间](./time#events)。

## 错误

返回 `Err(orr::SimError)`（或者 panic）的 task 会以一个错误结束这次运行，错误里写明 task、实例和时刻点：`run_until` 返回 `Err(...)`。用
`orr::SimError::new("message")` 构造，或者对任何返回它的调用用 `?`。串行顺序里在出错的 task 之前的一切都已经运行；在它之后的，没有规定。见[报错与调试](./errors#which-error-is-reported)。
