---
layout: home

hero:
  name: Orrery
  text: 用 Rust 给整个系统建模
  tagline: 在一个 Rust 模型里结合时钟驱动的时序与离散事件，并行仿真，结果与怎么运行无关。
  image:
    light: /logo.svg
    dark: /logo-dark.svg
    alt: Orrery
  actions:
    - theme: brand
      text: 快速开始
      link: /zh/guide/getting-started
    - theme: alt
      text: GPGPU-Sim 模型
      link: /zh/examples/gpgpu-sim

features:
  - title: 并行而确定
    icon:
      light: /icons/deterministic.svg
      dark: /icons/deterministic-dark.svg
      width: 44
      height: 44
    details: 不论用几个 worker，状态、事件、结束方式和报出的错误都完全相同。Orrery 在加载时推出模型的顺序，拒绝有歧义的写法，运行之后还可以对照检查。
  - title: 用普通的 Rust 写模型
    icon:
      light: /icons/rust.svg
      dark: /icons/rust-dark.svg
      width: 44
      height: 44
    details: 模块是 Rust 结构体，状态是 Rust 值，行为是 Rust 函数。其余的交给宏，参数来自 TOML 文件。
  - title: 时钟与事件同在
    icon:
      light: /icons/clocks.svg
      dark: /icons/clocks-dark.svg
      width: 44
      height: 44
    details: 任意多个以整数皮秒计的时钟域，加上带物理延迟或 delta 周期的事件，放在同一个模型里，从流水线的一级到 GPU 之间的网络。
  - title: 为速度而设计
    icon:
      light: /icons/speed.svg
      dark: /icons/speed-dark.svg
      width: 44
      height: 44
    details: 状态原地存放，task 的函数体是编译好的 Rust，模型里互不依赖的部分在多个核上并行运行。
  - title: 快照不停机
    icon:
      light: /icons/snapshots.svg
      dark: /icons/snapshots-dark.svg
      width: 44
      height: 44
    details: 快照在仿真运行的同时增量地拍下，可以从快照恢复，不必从头再跑就能复现问题。
  - title: 边界开放
    icon:
      light: /icons/open.svg
      dark: /icons/open-dark.svg
      width: 44
      height: 44
    details: 外部端口接入 trace、功能模型或 RTL 联合仿真；组件和接口都可以由用户自己定义。
---
