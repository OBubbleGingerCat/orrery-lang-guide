# 事件基准

一个由事件驱动的合成模型，这是 GPGPU-Sim 模型（只有时钟）从来用不到的。它有两种形态：每一级都互相依赖的多条 lane，以及一个更接近真实模型的片上 mesh 网络。

## Lane

```text
Top (clocks)
└── lanes: [Lane; lane_count]
    ├── source: Source        every `every` clock edges: emit a token
    ├── stages: [Stage; stage_count]
    │     on a token from its own lane or the next one:
    │     state = mix(state, token, `work` rounds); pass its own lane's token on
    └── sink: Sink            total += token
```

lane *i* 的第 *k* 个 stage 接收自己 lane 和 lane *i* + 1 上第 *k* 个位置的 token，所以各条 lane 互相依赖。stage 之间的每一跳要么是一个 delta 周期，要么是声明的
1 ns 最小延迟（[声明的延迟](../guide/time#declared-delays)）。各条 lane 可以跑在相位错开的时钟上，使每个时刻点只有少数几条 lane 活跃；另有一个变体，每条 lane 末尾的几个 stage
也由时钟触发。

## Mesh

一个 `width` × `height` 的路由器 mesh。每个路由器的时钟注入数据包，其中一部分发往一个热点，使负载不均匀；数据包先走 X 再走 Y，每个输出端口有自己的队列，每周期发出一个，经过的链路是带物理延迟（默认 2 ns）的事件。有一个变体加了一个监视器，它在时钟上汇总每个路由器的计数器，一旦送达的数据包达到目标数量就停止仿真，这在事件驱动模型中检验了[停止仿真](../guide/running#stopping)。

## 受门控的单元

挂在受门控时钟上的单元（[时钟门控](../guide/time#clock-gates)）：每次运行时把状态混合一次，运行 `busy` 次之后关闭门控 1 到 `idle` 个沿，具体多少由状态决定。一个配置每次运行都写门控（运行一次，空闲一个沿）；另一个让单元忙 20 次，再空闲最多 80 个沿。程序输出运行次数、被门控的沿和校验和。

## 它说明了什么

每种配置都打印一个校验和，所有线程数都必须得到同一个校验和。

## 构建和运行 {#build-and-run}

在 Orrery 仓库里，用 `bench/models/event_bench/params/` 下的一个参数文件：

```sh
cargo build --release -p event-bench-model --bin event-bench
target/release/event-bench --params bench/models/event_bench/params/light.toml --workers 4
target/release/event-bench --model mesh --hop timed --params bench/models/event_bench/params/mesh_light.toml
target/release/event-bench --model gated --params bench/models/event_bench/params/gated_turns.toml
```

```text
tokens 160016 checksum a834b91aff890990
10000 cycles in 0.192s (19.2 µs per cycle, 0.071s to load)
```

- `--model plain|mixed|mesh|mesh_goal|gated` 选形状（缺省是 lane；`mixed` 加上受时钟触发的级），`--hop delta|timed` 选级与级之间怎么传，`--cycles <n>` 选长度（缺省 10,000），`--workers <n>` 选线程数（[线程](../guide/running#threads)）。
- 配 lane 以外的 `--model` 的参数文件，开头几行写明了是哪个。
- 第一行是结果，用几个线程都一样；第二行是用时，构建仿真的时间单独列出。
