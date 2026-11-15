# 事件流水线

最小的多模块模型，也就是[快速开始](../guide/getting-started)一步步搭起来的那个。

```text
Top                        a 10 ns clock
└── workers: [Worker; worker_count]
    ├── producer: Producer on every clock edge: value += step, send value
    └── sink: Sink         on every value received: total += value
```

它用大约 80 行代码展示了：

- 模块参数，以及一个大小由参数决定的子模块数组（`[Worker; orr::DynShape(worker_count)]`）；
- 嵌套的子模块，在各个模块的 `init` 里连接；
- 经中继穿过一层层级的时钟（`Clock` → `ClockRelay` →
  `ClockRelay`）；
- 连到中继上的事件源（`EventSource<u64>` →
  `EventRelay<u64>`）；
- 状态，以及最小的一对有用的 task：一个由时钟触发，发出事件；一个由事件触发。

## 运行

用默认参数（两个 worker，步长 1）时，时钟沿落在 0、10、…、100 ns，所以
`run_until(100 ns)` 之后每个 sink 的值都是 1 + 2 + … + 11 = 66。用一个参数文件把
worker 1 的步长设为 10，它的 sink 的值就是 660：

```toml
[top]
worker_count = 3

[[top.workers]]
__select_index__ = 1
[top.workers.producer]
step = 10
```

```text
workers[0].sink.total = 66
workers[1].sink.total = 660
workers[2].sink.total = 66
```

不论用几个线程，得到的数都一样。

## 构建和运行 {#build-and-run}

模型是 Orrery 仓库里 `bench/models/event_pipeline` 下的包 `event-pipeline-model`。它是一个库，没有自己的程序：要运行它，在你自己的 crate 里依赖它，

```toml
[dependencies]
orrery-lang = { path = "<orrery>/crates/orrery-lang" }
event-pipeline-model = { path = "<orrery>/bench/models/event_pipeline" }
```

再照[快速开始](../guide/getting-started#running-it)写一个 `main`，顶层模块用 `event_pipeline::top::Top`。然后 `cargo run --release -- params.toml` 打印出上面那些总数。
