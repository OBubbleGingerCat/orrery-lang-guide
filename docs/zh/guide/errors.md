# 错误与调试

大多数建模错误在任何东西运行之前就会被发现：由编译器发现，或者在构建 plan 时、仿真加载时发现，并用模型里的实例路径指明。其余的在发生时被发现，并给出时刻点。每种错误都能用 `?` 转成 `orr::SimError`。

## 编译时 {#when-it-compiles}

编译器拒绝：

- 类型与字段不一致的 task 参数：
  ``a task parameter of type `u32` cannot bind a field of type `u64` ``；
- 在不能相连的种类之间的连接：时钟连到事件中继，或者连进一个时钟、一个事件源；
- 把门控移出调用的 task（[时钟门控](./time#clock-gates)）。

## 构建 plan 时 {#when-the-plan-is-built}

`SimPlan::new` 在以下情况失败，并给出消息：

**参数**
- 一个不对应任何实例的表，或者一个不是参数的键：
  `top.x is not a parameter`；
- 类型不对的值，或者 `validate` 失败：
  `top.limits: validation failed: ...`；
- `DynShape` 的长度为零：
  `top.n: dynamic shape must be greater than zero`。

**连接**
- 从未连接的中继、引用或接口：
  `x: unbound handle (referenced by y)`；
- 连接了两次的：`x: duplicate driver (from y)`；
- 由引用构成的环：`connection cycle: a -> b -> a`；
- 从未配置的时钟：`top.clock: Clock was not initialized`，或者配置了两次的时钟：`top.clock: clock initialized more than once`；
- 从未赋值、或者赋了两次值的[常量](./modules#constants)：
  ``top.child: constant `depth` is assigned twice``。

**task**
- 同一个 task 的两个参数以不同的模式访问同一个位置：
  ``state parameters `value` and `alias` conflict on ...``；
- 门控被另一个 task 写，而不是被它所控制的触发端口所属的那个 task 写。

## 仿真加载时 {#when-the-simulation-loads}

`orr::Sim::new` 和 `orr::Sim::with` 在以下情况失败，返回一个 `orr::LoadError`：

**顺序**（`LoadError::Hazards`），由 Orrery 根据访问模式推出（[状态与 task](./state-and-tasks#access-modes)）：

- 两个 task 在同一个时刻点必然写同一个位置：

  ```text
  write conflict: %WriteConflict.bump at top and %WriteConflict.clear at top
  both write top.value (both triggered by clock top.clock (1ns phase 0ps))
  ```

- 总是一起运行的 task 之间形成环：

  ```text
  dependency cycle (triggered by clock top.clock (1ns phase 0ps)):
  %Producer.send at top.producer writes top.fifo.slots before
  %BothFirst.receive at top.consumer reads it (latest); ...
  ```

  对 FIFO 来说，这就是两侧都声称自己先动作，或者都声称在后（[组件](./components#who-acts-first)）。

**配置**（`LoadError::Config`，带字段名）：`delta_limit` 超过 62，`groups` 不是 1，`workers_per_group` 为 0 或者超过一个 L3 域容得下的数目，以及多个 worker 目前还不能运行的东西（[worker](./running#workers)）。

**初始化**（`LoadError::Init`，带 `stage`）：

- `clocks`：两个时钟的沿相距不到 1 fs 而又不重合（[时钟](./time#clocks)）；
- `external`：外部端口没有绑定，`external port %Ports.log row 0 is not bound`，或者绑定了两次，`top.log: external port is bound twice`；
- `snapshots`：配置了快照时，task 写（`&mut`）的外部端口的源对象不能保存，或者快照目录建不起来。

**恢复**（`LoadError::Restore`，带文件名）：快照读不出来，或者属于另一个模型（[快照](./snapshots#restoring)）。

### 警告 {#warnings}

只是*可能*发生的冲突和环（因为涉及事件）则是警告，在仿真运行时检查。加载时把每条警告打印到 stderr，`warning: ...`，除非 `reports` 是 `Off`；警告也留在加载的报告里：

```rust
for warning in &sim.load_report().warnings {
    eprintln!("{warning}");
}
```

```text
possible write conflict, checked at run time: %MayConflict.on_a at top and
%MayConflict.on_b at top both write top.value (triggered by event top.a_out
and event top.b_out)
```

## 运行时 {#while-it-runs}

`run_until` 和 `step` 失败时返回一个 `orr::RunError`。模型里发生的问题是 `RunError::Occurrence`，它让运行结束：`at` 是时刻点，`call` 是 task 和实例（不是某一个 task 的错时为 `None`），`kind` 是一个 `orr::FailureKind`，`message` 是它显示出来的文字：

| 错误 | `kind` | 原因 |
|---|---|---|
| `%Sm.cycle at top.sms[3] failed at 1200ns: ...` | `Task(Body)` | 一个 task 返回了错误 |
| `... panicked: ...` | `Task(Panic)` | 一个 task panic 了；错误里写明是哪个 task、哪个实例 panic 的 |
| `multi-drive at 10ns+1: A and B both emit to L` | `MultiDrive` | 同一个位置、同一个时刻点上有两个事件 |
| `write conflict at 37ns: A and B both write L` | `WriteConflict` | 一个可能的写冲突真的发生了 |
| `dependency cycle among ...` | `BlockCycle` | 可能一起运行的 task 在这个时刻点形成了顺序环 |
| `more than 30 delta cycles at 5ns` | `DeltaLimit` | 一串永远稳定不下来的 delta 周期（`delta_limit`） |
| `more than N events in flight from one location` | `Task(Capacity)` | 超出了 `event_capacity` |
| `event delay ... is below min_delay ...` | `Task(Delay)` | 一次发出的延迟不在源声明的延迟之内，或者延迟为零 |
| `... is after the latest time a run ... reaches, ...` | `Task(Range)` | 一次发出的事件会在一次运行能走到的最晚时间之后到达（[delta 周期](./time#delta-cycles)） |
| `an event at ... reached the trigger ..., whose gate ... is closed, and dropped events are errors` | `DroppedEvent` | 开了 `fail_on_drop` 时，一个事件在关着的门控上被丢弃（[时钟门控](./time#clock-gates)） |

其他的 `RunError`：

- `Request`：不成立的请求，例如 `run_until` 超过一次运行能走到的最晚时间，或者给的是 `Time::DELTA`，或者用了另一个仿真的句柄。什么也不运行，仿真还可以继续。
- `Failed`：之前的一次运行失败了，`an earlier run failed; the simulation cannot continue`。
- `EventLog`：[事件日志](./running#event-log)的 sink 返回了错误；`LogClosed`：`close_event_log` 之后又运行。

### 报的是哪个错误 {#which-error-is-reported}

不止一处失败时，Orrery 报串行顺序里最早的那一个：一次只跑一个 task 的运行会按什么顺序碰到它们，就按什么顺序，即逐个时刻点，在一个时刻点之内按一个符合访问模式的固定顺序。同一个模型、同一个输入，不管用几个 worker，报的都是同一个错误。

- 只会发生在 stop 终点之后的失败不报：运行按 stop 结束（[停止](./running#stopping)）。
- 开了 `fail_on_drop` 时，被丢弃的事件让它所在的时刻点失败，这个时刻点的 task 一个也不运行。
- 一个 task 发出的事件造成了多驱动，之后它又返回错误或 panic：报多驱动。

出错之后，串行顺序里在失败之前的一切都已经运行，宿主可以用 `sim.state` 读出它留下的状态。失败的 task 可能只写了一部分，在它之后的工作也可能已经运行：这一部分状态没有规定，也可能随 worker 数而不同。之后的每次 `run_until` 和 `step` 都返回错误。

## 调试

- **单步运行。** `sim.step()` 运行一个时刻点；`sim.next_timepoint()` 给出是哪个。在两步之间用 `sim.state(handle)` 查看状态。
- **检查运行。** 用 `SimConfig { checks: true, .. }`、一个 worker 时，运行之后 `sim.check_trace()` 检查访问同一个状态、其中至少一个是写的每两次访问都按串行顺序运行了。
- **比较不同的 worker 数。** 对能用多个 worker 运行的模型，结果，包括报出的错误，从不取决于 worker 数。如果一个模型在不同的 worker 数下给出不同的结果，那是 Orrery 的 bug。
- **快照与恢复。** 在长时间的运行中拍快照，恢复到问题之前的那一个，再用 `step`
  循环或更多的日志排查（[快照](./snapshots)）。
