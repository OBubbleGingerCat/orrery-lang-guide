# 组件与接口

## plain 类型 {#plain-types}

作为状态存放或作为事件载荷发出的一切，都是 **plain** 类型：大小固定，按字节复制，没有要 drop 的东西，没有指针。

- 内置的：整数、浮点数、`bool`、`()`、`orr::Time`、
  `orr::ClockInfo`、`Option<T>`，以及由 plain 类型组成的数组和元组。
- 自己的：在字段都是 plain 类型的 struct 或 enum 上加
  `#[derive(orr::Plain)]`。

```rust
use orrery_lang as orr;

#[derive(Debug, Default, Clone, Copy, PartialEq, orr::Plain)]
pub enum Op {
    #[default]
    Idle,
    Read(u64),
    Write { addr: u64, data: [u8; 4] },
}
```

`Vec`、`String`、`Box` 和引用都不是 plain 类型；请用容器。参数不是状态，只需要 `orr::ParamValue`。

## 容器

容器存放有限个 plain 元素，容量在编译时固定（`Queue<T, 8>`），或者在模型加载时按实例确定（`Queue<T, orr::DynShape(depth)>`，大小由参数给出）：

| 容器 | 是什么 |
|---|---|
| `orr::Array<T, N>` | N 个元素，作为**一个**位置存放，总是整体访问 |
| `orr::ArrayVec<T, N>` | 有界向量：满了时 `push` 把值原样返回 |
| `orr::Queue<T, N>` | 环形 FIFO：`push`、`pop`、`front`、`iter`、… |
| `orr::Map<K, V, N>` | 线性查找的小 map |
| `orr::BitSet![N]` | 位集合，带常见的集合操作 |
| `orr::Store<S>` | 由 Orrery 管理的稀疏存储，例如 `orr::SparseMem<orr::CacheLine<64>>`，单元在写入时才分配 |

在 task 里，加载时确定容量的容器以它的 `Dyn` 形式出现（`orr::DynQueue<T>`），辅助函数接收的也是这种形式。

## 组件

**组件**是一个可复用的模块，别的模块通过**接口**使用它。Orrery 的标准组件是
`orr::Fifo<T>`，即两个模块之间的有界队列：生产者一侧是 `orr::FifoTx<T>`，消费者一侧是 `orr::FifoRx<T>`。

```rust
#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub out: orr::FifoTx<u64>,

    next: u64,
    pub stalls: u64,
}

#[orr::module_impl]
impl Producer {
    #[orr::task(triggers = (clock,))]
    fn send(
        _ctx: &mut orr::TaskCtx,
        out: &'latest mut orr::FifoTx<u64>,
        next: &mut u64,
        stalls: &mut u64,
    ) -> Result<(), orr::SimError> {
        match out.push(*next) {
            Ok(()) => *next += 1,
            Err(_) => *stalls += 1,
        }
        Ok(())
    }
}

#[orr::module]
pub struct Consumer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,

    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    #[orr::task(triggers = (clock,))]
    fn receive(
        _ctx: &mut orr::TaskCtx,
        inbox: &mut orr::FifoRx<u64>,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        Ok(())
    }
}
```

拥有 `Fifo` 的模块把两侧都连到它上面：

```rust
#[orr::module]
pub struct Top {
    #[orr::trigger]
    clock: orr::Clock,

    #[orr::submodule]
    fifo: orr::Fifo<u64>,

    #[orr::submodule]
    producer: Producer,

    #[orr::submodule]
    consumer: Consumer,
}

#[orr::module_impl]
impl Top {
    #[orr::init]
    fn init(ctx: &mut orr::InitCtx) -> Result<(), orr::SimError> {
        let module = ctx.module();
        ctx.init_clock(module.clock, orr::Time::ns(1), orr::Time::ps(0))?;
        ctx.connect(module.clock, module.producer.clock)?;
        ctx.connect(module.clock, module.consumer.clock)?;
        ctx.connect_interface(&module.fifo, module.producer.out)?;
        ctx.connect_interface(&module.fifo, module.consumer.inbox)
    }
}
```

队列的深度是 `Fifo` 实例的一个参数：

```toml
[top.fifo]
depth = 2
```

`FifoTx` 有 `push`、`is_full`、`free`、`len`、`capacity`；`FifoRx` 有
`pop`、`front`、`get(i)`、`is_empty`、`len`、`capacity`。

## 谁先动作 {#who-acts-first}

FIFO 的两侧在同一个周期运行时，必须有一侧先动作，结果取决于是哪一侧：本周期
push 的值能不能在本周期 pop 出来？本周期腾出的槽能不能再用？接口参数的写法说明了每一侧的先后，而且**恰好有一侧必须是 `&'latest mut`**：

| 生产者 | 消费者 | 含义 |
|---|---|---|
| `&'latest mut FifoTx` | `&mut FifoRx` | 消费者先动作：第 *k* 周期 push 的值从 *k* + 1 起才能 pop；pop 腾出的槽可以在同一周期再填 |
| `&mut FifoTx` | `&'latest mut FifoRx` | 生产者先动作：push 的值可以在同一周期 pop；腾出的槽从下一周期起才能再用 |

两侧都是 `&mut`，或者都是 `&'latest mut`，就构成一个顺序环，在仿真加载时报错，错误信息里给出两个 task 和这个 FIFO 的状态（[报错](./errors#when-the-simulation-loads)）。顺序属于 task，不属于组件，而且只在两者都运行的时刻点才有意义（比如两个时钟相遇的地方）。

一个 task 同时持有同一个 FIFO 的两侧，只能是为了看两侧：`&FifoTx<T>` 和 `&FifoRx<T>`，看到的是周期开始时的整个 FIFO。只要有一侧是 `&mut` 或 `&'latest mut`，或者一侧是 `&`、另一侧是 `&'latest`（同一个计数要在周期里两个不同的时候去读），就在构建 plan 时报错：

```rust
// 可以：两侧都是 `&`。
#[orr::task(triggers = (clock,))]
fn watch(
    _ctx: &mut orr::TaskCtx,
    out: &orr::FifoTx<u64>,
    inbox: &orr::FifoRx<u64>,
    free: &mut u64,
) -> Result<(), orr::SimError> {
    *free = (out.capacity() - inbox.len()) as u64;
    Ok(())
}
```

两个 task 也不能都用 `&mut` 或 `&'latest mut` 持有同一侧：在它们一起运行的地方，这是仿真加载时报出的写冲突。别的 task 仍然可以看这一侧（[只看不取](#looking-without-popping)）。

## 只看不取 {#looking-without-popping}

有时好几个 task 都要看 FIFO 里等着的东西，而只有其中一个 pop：一个仲裁器要看它每个输入队列的队头，一个统计 task 要数缓冲有多满。这样的 task 用共享引用取这一端，`&FifoRx<T>` 或 `&'latest FifoRx<T>`，只调只看不改的方法：`front`、`get(i)`、`len`、`is_empty`、`capacity`。除了从这一端 pop 的那一个 task，可以有任意多个 task 这样看它；它们改不了它，经 `&FifoRx` 调 `pop` 无法编译。

看的 task 对本周期能看到什么：

| 写法 | 另一侧的改动 | 自己这一侧的改动 |
|---|---|---|
| `&FifoRx` | 看不到 | 看不到 |
| `&'latest FifoRx` | 看得到 | 看不到 |

在消费者一端，`&FifoRx` 看到的是周期开始时的 FIFO：本周期的 push 之前，也在本周期的 pop 之前。`&'latest FifoRx` 看得到本周期的 push，但仍然看不到本周期的 pop。生产者一端一样，`&FifoTx` 和 `&'latest FifoTx`，两侧对调：看的 task 永远看不到自己这一侧本周期的改动。

`&'latest FifoRx` 要排在生产者之后、pop 之前，所以要生产者先动作（`&mut FifoTx` 和 `&'latest mut FifoRx`）。消费者先动作时没有这样的位置：这是一个顺序环，在仿真加载时报错（[报错](./errors#when-the-simulation-loads)）。`&FifoRx` 两种先后都行。数组也一样：`&[orr::FifoRx<T>]` 和 `&'latest [orr::FifoRx<T>]`。

一个完整的例子：生产者先动作，每个周期推一个值；消费者每个周期 pop 一个；消费者另有两个 task 看这个 FIFO。

```rust
use orrery_lang as orr;

#[orr::module]
pub struct Producer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub out: orr::FifoTx<u64>,

    next: u64,
}

#[orr::module_impl]
impl Producer {
    // 先动作：有空位就推 1、2、3……
    #[orr::task(triggers = (clock,))]
    fn send(
        _ctx: &mut orr::TaskCtx,
        out: &mut orr::FifoTx<u64>,
        next: &mut u64,
    ) -> Result<(), orr::SimError> {
        if out.push(*next + 1).is_ok() {
            *next += 1;
        }
        Ok(())
    }
}

#[orr::module]
pub struct Consumer {
    #[orr::trigger]
    pub clock: orr::ClockRelay,

    #[orr::interface]
    pub inbox: orr::FifoRx<u64>,

    pub waiting: u64,
    pub newest: u64,
    pub sum: u64,
}

#[orr::module_impl]
impl Consumer {
    // 在生产者之后动作：每个周期取一个值。
    #[orr::task(triggers = (clock,))]
    fn receive(
        _ctx: &mut orr::TaskCtx,
        inbox: &'latest mut orr::FifoRx<u64>,
        sum: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(value) = inbox.pop() {
            *sum += value;
        }
        Ok(())
    }

    // 在本周期的 push 和 pop 之前看。
    #[orr::task(triggers = (clock,))]
    fn peek(
        _ctx: &mut orr::TaskCtx,
        inbox: &orr::FifoRx<u64>,
        waiting: &mut u64,
    ) -> Result<(), orr::SimError> {
        *waiting = inbox.len() as u64;
        Ok(())
    }

    // 在本周期的 push 之后、pop 之前看。
    #[orr::task(triggers = (clock,))]
    fn look(
        _ctx: &mut orr::TaskCtx,
        inbox: &'latest orr::FifoRx<u64>,
        newest: &mut u64,
    ) -> Result<(), orr::SimError> {
        if let Some(last) = inbox.len().checked_sub(1) {
            *newest = *inbox.get(last).unwrap();
        }
        Ok(())
    }
}
```

`Top` 照第一个例子连接它们。每个周期 `peek` 看到 FIFO 是空的，`look` 看到刚推进去的值，然后 `receive` 把它 pop 出来：跑十个周期（0 到 9 ns）以后，`waiting` 是 0，`newest` 是 10，`sum` 是 55。

## 数组与转发

接口可以是数组：

```rust
#[orr::interface]
pub outs: [orr::FifoTx<u64>; orr::DynShape(lanes)],
```

拥有者的 `init` 逐个元素连接它们：

```rust
ctx.connect_interface(&module.fifos[lane], module.splitter.outs[lane])?;
```

如果一个模块不是组件的直接父模块，它声明同样的接口，再由它的父模块转发：
`ctx.share_interface(module.inbox, module.consumer.inbox)?`。

## 自己的接口

接口是一个由若干部分组成的 struct，每个部分都是组件的一个 `pub` 字段，标为自己拥有的（由持有者写），或者另一侧的（只读）。`orr::Fifo` 和它的生产者一侧就是这样写的：

```rust
#[orr::module]
pub struct Fifo<T: orr::Plain + Default> {
    #[orr::param("capacity in elements", default = 2usize)]
    depth: usize,

    pub slots: orr::Array<T, orr::DynShape(depth)>,
    pub tail: u64,
    pub head: u64,
}

#[orr::module_impl]
impl<T: orr::Plain + Default> Fifo<T> {}

#[orr::interface(component = Fifo<T>)]
pub struct FifoTx<T: orr::Plain + Default> {
    slots: orr::Own<orr::DynArray<T>>,
    tail: orr::Own<u64>,
    head: orr::Other<u64>,
}

impl<T: orr::Plain + Default> FifoTx<T> {
    pub fn push(&mut self, value: T) -> Result<(), T> {
        let capacity = self.slots.len() as u64;
        if *self.tail - *self.head == capacity {
            return Err(value); // 满了
        }
        self.slots[(*self.tail % capacity) as usize] = value;
        *self.tail += 1;
        Ok(())
    }
}
```

每个部分的名字必须是组件里一个类型兼容的 `pub` 字段；这在编译时检查。在
`&mut I` 下，task 写自己拥有的部分，读到的其他部分是原来的值（`curr`）；在
`&'latest mut I` 下，它读的是另一侧动作之后的值（`latest`）。只看的 task 用 `&I` 或 `&'latest I`（数组用 `&[I]`、`&'latest [I]`）：它读自己这一侧拥有的东西时，读的是本周期改动之前的值，其余的照 `&mut I` 或 `&'latest mut I` 那样读，只能调接受 `&self` 的方法。接口如果有 `impl Drop`，它的 `drop` 不会为这样取它的 task 运行。任何模块都可以是组件，包括自己也有 task 的模块：接口是给模块一个有类型、有顺序的端口的通用办法，很像 SystemVerilog 的 interface 和它的 modport。
