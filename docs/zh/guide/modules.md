# 模块与参数

模块是标记了 `#[orr::module]` 的 Rust 结构体，并且有且只有一个
`#[orr::module_impl]` 块，放它的 `init` 和 task，即使这个块是空的：

```rust
use orrery_lang as orr;

#[orr::module]
pub struct Leaf {
    value: u64,
}

#[orr::module_impl]
impl Leaf {}
```

模块不用手工构造：`SimPlan::new` 按参数创建每个实例（[展开](#elaboration)），宿主通过 `plan.top` 访问它们。

## 字段

| 字段 | 是什么 |
|---|---|
| `#[orr::param("doc", default = expr)] name: T` | 一个参数，在仿真开始前就固定下来 |
| `#[orr::trigger] clock: orr::Clock` | 一个独立的时钟源，在 `init` 里配置 |
| `#[orr::trigger] clock: orr::ClockRelay` | 来自别处的时钟 |
| `#[orr::trigger] out: orr::EventSource<T>` | 载荷为 `T` 的事件源 |
| `#[orr::trigger] input: orr::EventRelay<T>` | 来自别处的事件 |
| `wake: orr::Wake` | 打开其他模块的门控（[唤醒](./time#wakes)） |
| `#[orr::wake(gate = gate)] wake: orr::WakeRelay` | 让连过来的唤醒打开本模块的 `gate` |
| `#[orr::constant] depth: T` | 一个常量，在一次运行里固定不变（[常量](#constants)） |
| `#[orr::submodule] child: Child` | 一个子实例 |
| `#[orr::interface] out: orr::FifoTx<T>` | 通往一个组件的接口（[组件](./components)） |
| `name: T`（不带属性） | 状态，`T` 是 [plain 类型](./components#plain-types)、容器、`Store` 或 `External` 端口 |
| `name: orr::StateRef<T>` | 对另一个模块拥有的状态的引用 |

不带 Orrery 属性的字段是状态；除了文档注释，带其他任何属性都是错误。

**可见性。** 参数和子模块必须是私有的。父模块恰好能看到直接子模块里声明为 `pub`
的那些字段（状态、触发源、接口、常量），更深的一层都看不到：要访问孙模块，子模块得通过自己的一个 `pub` 中继、引用或接口把它转发出来。

## 数组 {#arrays}

子模块、触发源、状态、引用和接口都可以是数组：

```rust
#[orr::param("number of workers", default = 2)]
worker_count: usize,

#[orr::submodule]
workers: [Worker; orr::DynShape(worker_count)],
```

- `[X; 4]`：固定长度，一个非零的字面量；
- `[X; N]`：一个常量，可以是模块的 const 泛型参数、定义模块的地方能用到的 `const`（`CORES`、`cfg::CORES`、`<T as Trait>::N`），或者由它们算出的表达式（`2 * N`、`{ N + 1 }`）。不能用 `Self`，也不能用模块的字段。值为零是错误；
- `[X; orr::DynShape(p)]`：同一个模块的参数 `p` 的值，每个实例各自取值。参数要直接写名字，不允许写表达式。值为零是错误。

**长度是个算式时。** `DynShape` 的长度只能写一个参数的名字，所以由别的参数算出来的长度（比如缓存的 `sets` × `ways` 行）要单独做成一个参数。驱动程序把它算好，和别的参数一起写进参数文件；再在 `init` 里核一遍：文件里给错了，`SimPlan::new` 就带着 init 的那句话失败，不会建出一个不对的模型。

```rust
#[orr::module]
pub struct Cache {
    #[orr::param("sets", default = 64usize)]
    sets: usize,
    #[orr::param("ways", default = 4usize)]
    ways: usize,
    #[orr::param("lines, sets x ways", default = 256usize)]
    lines: usize,
    #[orr::trigger]
    clock: orr::Clock,
    #[orr::reset(u64::MAX)]
    tags: [u64; orr::DynShape(lines)],
    next: u64,
}

#[orr::module_impl]
impl Cache {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        if *module.sets * *module.ways != *module.lines {
            return Err(orr::SimError::new("lines must be sets x ways"));
        }
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))
    }

    #[orr::task(triggers = (clock,))]
    fn fill(ctx: &mut orr::TaskCtx, tags: &mut [u64], next: &mut u64) -> Result<(), orr::SimError> {
        let (sets, ways) = (*ctx.module().sets, *ctx.module().ways);
        let set = *next as usize % sets;
        let way = *next as usize / sets % ways;
        tags[set * ways + way] = *next;
        *next += 1;
        Ok(())
    }
}
```

驱动程序从它自己的输入里读到 `sets` 和 `ways` 以后：

```rust
let params = out_dir.join("params.toml");
std::fs::write(&params, format!("[top]\nsets = {sets}\nways = {ways}\nlines = {}\n", sets * ways))?;
let plan = orr::SimPlan::<Cache>::new(Some(params), None)?;
```

plan 只能从文件拿参数（[运行仿真](./running)），所以驱动程序要写一个；写在这次运行的输出旁边，也顺带记下了这次运行用的是什么。

状态数组的每个元素都是一个独立的位置，与其他元素分开定序。应当作为一个位置的值（总是整体取用的缓冲区）要改用 `orr::Array<T, N>`。

task 把接口数组整个当视图取，`orr::ArrayView`（[数组与转发](./components#arrays-and-forwarding)、[对数组写函数](./components#functions-over-arrays)）；状态数组、`StateRef` 数组也可以当视图取（[绑定其他字段](./state-and-tasks#binding-other-fields)）。

## 复位值

状态的初值是 `Default::default()`，或者 `#[orr::reset(..)]` 给出的值：

```rust
#[orr::param("credits at reset", default = 4u32)]
credits_init: u32,

#[orr::reset(credits_init)]     // 本模块的一个参数
pub credits: u32,

#[orr::reset(u64::MAX)]         // 一个常量，每个元素都用它
tags: [u64; orr::DynShape(depth)],
```

`#[orr::reset(with = f)]` 对默认值调用 `fn f(value: &mut T)`，
`#[orr::reset(with = f, args = [p, q])]` 还会把参数也传给它。复位值在模型展开时计算。

没有 `#[orr::reset(..)]` 的字段，初值只用 `Default::default()` 做一次，不是每个实例做一次：每个实例、状态数组的每个元素，都从它的一份拷贝开始。大小由参数定的类型，如 `orr::Array<T, orr::DynShape(depth)>`，实例里有几种大小就做几次。所以每次调用给出不同值的 `Default`（比如从一个计数器取号），仍让每个实例从同一个值开始。要让各个实例的初值不同，用 `#[orr::reset(..)]`，像上面的 `credits` 那样从参数取。

## 参数 {#parameters}

参数有一个 `default`（必需），还可以有 `validate = f`，即一个函数
`fn(&T) -> Result<(), orr::ParamError>`：

```rust
fn positive(value: &u32) -> Result<(), orr::ParamError> {
    match *value {
        0 => Err(orr::ParamError::new("must be positive")),
        _ => Ok(()),
    }
}

#[derive(Debug, Clone, Copy, PartialEq, orr::ParamValue)]
pub enum Policy {
    RoundRobin,
    Fixed { first: u8 },
}

#[orr::module]
pub struct Port {
    #[orr::param("credits at reset", default = 4u32, validate = positive)]
    credits_init: u32,

    #[orr::param("arbitration", default = Policy::RoundRobin)]
    policy: Policy,

    #[orr::param("link delay", default = orr::Time::ns(2))]
    link: orr::Time,

    #[orr::trigger(min_delay = link)]
    pub out: orr::EventSource<u64>,

    #[orr::reset(credits_init)]
    pub credits: u32,
}
```

参数的类型：整数、浮点数、`bool`、`char`、`String`、`orr::Time`（写作
`"10ns"`、`"250 ps"`、`"1500fs"` 或 `"delta"`）、由这些类型组成的数组和元组，以及任何带
`#[derive(orr::ParamValue)]` 的结构体或枚举。值的转换是严格的：浮点参数在文件里写成整数是错误。

init 和 task 通过 `ctx.module()` 读取参数：`*ctx.module().credits_init`。参数永远不变。

## 参数文件

表按实例路径组织，从 `top` 开始：

```toml
[top]
ports = 3

[[top.port]]            # 子模块数组：用数组表
credits_init = 8        # 没有选择器：所有元素

[[top.port]]
__select_index__ = 2    # 只有元素 2（也可以是列表、"a..b"、"a..=b"）
policy = { Fixed = { first = 1 } }
link = "500ps"
```

- 单个子模块是一个表，`[top.child]`；数组的元素是数组表，`[[top.children]]`，可以带一个 `__select_index__`。数组表后面的表作用于它选中的那些元素。
- 有几条规则设置同一个参数时，更具体的那条优先，其次是后写的那条。
- `__include__ = "other.toml"` 加载一个文件，把它的 `[top]` 表放到写这一行的那个表的位置上。
- 表必须对应一个存在的实例，键必须是这个实例的一个参数；其他情况都是错误。没有设置的参数保持默认值。

派生类型写成 TOML 值：结构体写成内联表，枚举写成 `"Variant"` 或
`{ Variant = ... }`。

## `init`

模块可以有一个 `#[orr::init]`，每个实例运行一次，在子模块的 init 之前运行，负责把东西连起来。它不能读写状态的值，也不能发送事件。

```rust
#[orr::init]
fn init_top(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
    let module = ctx.module();
    ctx.init_clock(module.clock, orr::Time::ns(10), orr::Time::ps(0))?;
    for worker in module.workers.iter() {
        ctx.connect(module.clock, worker.clock)?;
    }
    Ok(())
}
```

| 方法 | 作用 |
|---|---|
| `ctx.module()` | 模块自己的视图：参数是值，其他都是句柄，子模块是它们的 `pub` 视图（数组是 `Vec`） |
| `ctx.path()` | 实例路径，比如 `top.workers[1]` |
| `ctx.init_clock(clock, period, phase)` | 配置一个独立的时钟，恰好一次 |
| `ctx.init_clock_hz(clock, hz, phase)` | 同上，按以赫兹计的频率（[时钟](./time#clocks)） |
| `ctx.clock_info(clock)` | 一个时钟的 `ClockInfo`，或者一个中继所连时钟的 |
| `ctx.connect(source, target)` | 时钟 → 时钟中继，事件源 → 事件中继，唤醒 → 唤醒中继，中继 → 中继 |
| `ctx.set_const(handle, value)` / `ctx.get_const(handle)` | 设一个[常量](#constants)，恰好一次 / 读祖先或本 init 已经设好的常量 |
| `ctx.share_state(state, reference)` | 让一个 `StateRef` 指向一个状态，或者指向另一个引用；store 和外部端口也用它 |
| `ctx.connect_interface(&child_component, interface)` | 把一个接口绑定到一个直接子组件上 |
| `ctx.share_interface(interface, child_interface)` | 把一个接口转发给子模块 |

连接在所有 `init` 都运行完之后才解析，所以连接的先后无关紧要。`ctx.clock_info` 是例外：它只看到到目前为止已经初始化的时钟和已经连接的中继。`init` 从根往下运行，所以一个 `init` 能读它自己初始化过的时钟、祖先连接好的中继，以及它连接之后的子模块中继，但不能读子模块的 `init` 才配置的时钟。每个时钟中继、事件中继、引用和接口都必须恰好连接一次：没有连接、有两个驱动、连到一个源上，以及引用成环，都是错误。唤醒中继例外：可以接 0 个到好几个唤醒，同一个唤醒接两次才是错误。一个都不接时，没有谁能经过它打开那扇门，也没有错误提示：忘了接唤醒，那个单元关了门以后就一直睡着。一个单元莫名其妙不动了，先查它的唤醒中继接没接上。

另一头什么都没有的中继（比如网格边上的路由器朝外的口）也得连上一个源。在父模块里放几个没有任何 task 发的源，把这样的中继连上去：

```rust
#[orr::module]
pub struct Line {
    #[orr::param("nodes", default = 4usize)]
    count: usize,
    #[orr::trigger]
    clock: orr::Clock,
    /// Never emitted: what the two ends' outward ports are connected to.
    #[orr::trigger]
    nothing: [orr::EventSource<u64>; 2],
    #[orr::submodule]
    nodes: [Node; orr::DynShape(count)],
}

// In Line's init, after each node is connected to its neighbours:
let (first, last) = (&module.nodes[0], &module.nodes[*module.count - 1]);
ctx.connect(module.nothing[WEST], first.inputs[WEST])?;
ctx.connect(module.nothing[EAST], last.inputs[EAST])?;
```

没有哪个 task 的 `emits` 里有 `nothing`，所以它上面从来不发东西：这两个中继永远不触发，运行的结果和没有它们时一样。把边绕到另一边去连也能跑，可那几条链路上有没有东西走，就要看节点的代码（没有节点往边外发），读的人得自己去查；上面这样写，连接本身就说明了。

这样的中继每个连一个自己的源，像上面那样放成数组。同一个 task 的两个触发不能连到同一个源上：建计划时会报"subscribes to trigger … more than once"。

## 常量 {#constants}

参数来自参数文件。**常量**是 `init` 在建模型时算出来、task 只读的值：一个单元在模型里的位置、父模块替它选的模式、一个它不在上面跑的时钟的周期。它是标了 `#[orr::constant]` 的字段，类型可以是任何 `Send + Sync + 'static` 的类型：

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Mode {
    Plain,
    Doubled,
}

#[orr::module]
pub struct Unit {
    #[orr::trigger]
    pub clock: orr::ClockRelay,
    #[orr::constant]
    pub id: u64,              // assigned by the parent
    #[orr::constant]
    pub mode: Mode,
    #[orr::constant]
    pub weights: [u64; 3],    // an array is one constant
    #[orr::constant]
    step: u64,                // assigned by the unit itself
    pub sum: u64,
}

#[orr::module_impl]
impl Unit {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        let id = *ctx.get_const(module.id)?;   // the parent's init ran first
        ctx.set_const(module.step, id + 1)
    }

    #[orr::task(triggers = (clock,))]
    fn tick(
        _ctx: &mut orr::TaskCtx,
        mode: &'constant Mode,
        weights: &'constant [u64; 3],
        step: &'constant u64,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        let k = if *mode == Mode::Doubled { 2 } else { 1 };
        *sum += k * (weights.iter().sum::<u64>() + *step);
        Ok(())
    }
}

// In the parent's init, for each child:
for (i, unit) in module.units.iter().enumerate() {
    ctx.connect(module.clock, unit.clock)?;
    ctx.set_const(unit.id, i as u64)?;
    ctx.set_const(unit.mode, if i % 2 == 0 { Mode::Plain } else { Mode::Doubled })?;
    ctx.set_const(unit.weights, [1, 2, 3])?;
}
```

**在哪设。** 只有 `init` 能设常量，用 `ctx.set_const(handle, value)`，而且恰好一次：

- 模块设自己的常量，`pub` 的和不 `pub` 的都行。父模块设子模块 `pub` 的常量；不 `pub` 的不在子模块的视图里，父模块叫不出它的名字（编译错误："no field"）。孙模块的常量一层一层往下传，每一层的 `init` 设自己子模块的。
- init 从根往下跑，所以模块自己的 `init` 能用 `ctx.get_const(handle)` 读到父模块设的值，再由它算出别的常量（上面的 `step`）。
- 常量没有默认值，在常量上写 `#[orr::reset]` 是编译错误。没有哪个 `init` 设的常量、设了两次的常量，都在建计划（`SimPlan::new`）时报错，报出实例和字段："constant `core` is not assigned"、"constant `core` is assigned twice"。

**task 怎么读。** task 把本模块的常量当作一个和字段同名的参数来读，写成 `&'constant T`，`T` 是字段的整个类型：

- `'constant` 必须写：`mode: &Mode` 是编译错误，报错会叫你写 `&'constant T`；`&'constant mut Mode` 也是。
- 数组是一个值：`weights: &'constant [u64; 3]` 是整个数组，不像状态数组那样每个元素是一个位置。
- 常量不是状态。读它不会让 task 排在谁后面；它也不在快照里：从快照恢复的运行会重新跑一遍 init，由 init 重新设常量。

**数组长度。** 常量不能给数组定长度：`[X; N]` 接受的是 Rust 的常量（[数组](#arrays)），`orr::DynShape(p)` 只接受同一个模块的参数，在那里写常量是编译错误（"DynShape must reference a local param"）。触发源的 `min_delay` 也不行。长度或延迟要由某个值决定时，把这个值做成参数。

**参数还是常量。**

| | 参数 | 常量 |
|---|---|---|
| 谁设 | 参数文件，或者它的默认值 | `init`，在代码里 |
| 类型 | [参数类型](#parameters) | 任何 `Send + Sync + 'static` 的类型 |
| 没设 | 用默认值 | 报错 |
| 数组长度、`min_delay` | 能 | 不能 |
| task 里怎么读 | `*ctx.module().name` | 参数 `name: &'constant T` |

用户应该能为一次运行改的值，做成参数。由实例在模型里的位置决定的值（它的编号、它在第几级），或者父模块由自己的参数算出来的值，做成父模块设的常量。父模块也可以这样把自己的一个参数传下去，变成每个子模块的常量。

对常量泛型的模块，也可以这样把那个数交给 task：由它自己的 `init` 设成一个常量：

```rust
#[orr::module]
pub struct Buffer<const N: usize> {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::constant]
    depth: usize,

    pub count: u64,
}

#[orr::module_impl]
impl<const N: usize> Buffer<N> {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.set_const(module.depth, N)
    }

    #[orr::task(triggers = (clock,))]
    fn fill(_ctx: &mut orr::TaskCtx, depth: &'constant usize, count: &mut u64) -> Result<(), orr::SimError> {
        if *count < *depth as u64 {
            *count += 1;
        }
        Ok(())
    }
}
```

## 展开 {#elaboration}

`orr::SimPlan::<Top>::new(params, snapshot)` 读取参数文件，建出每个实例，运行各个 `init`，解析连接，并检查每个 task 的绑定。各个 task 的先后在仿真加载时推出（[报错](./errors#when-the-simulation-loads)）。两者的报错写的都是实例路径，所以直接指向模型里出问题的地方。展开之后，实例树可以通过 `plan.top` 访问，所有字段都是公开的：参数是 `Param<T>`（`.get()`），状态和触发源是句柄。

泛型模块和在 Rust 里一样：`Fifo<T>` 和 `Cache<const WAYS: usize>` 的每种实例化都是不同的模块类型。
