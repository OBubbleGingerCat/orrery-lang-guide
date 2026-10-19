# 错误与调试

大多数建模错误在任何东西运行之前、构建 plan 时就会被发现，并用模型里的实例路径指明。其余的在发生时被发现，并给出时刻点。

## 构建 plan 时

`SimPlan::new` 在以下情况失败，并给出消息：

**参数**
- 一个不对应任何实例的表，或者一个不是参数的键：
  `top.x is not a parameter`；
- 类型不对的值，或者 `validate` 失败：
  `top.limits: validation failed: ...`；
- `DynShape` 的长度为零或负数。

**连接**
- 从未连接的中继、引用或接口：
  `x: unbound handle (referenced by y)`；
- 连接了两次的：`x: duplicate driver (from y)`；
- 连到源上的连接，或者在不能相连的种类之间的连接（时钟连到事件）：`invalid trigger connection ...`；
- 由引用构成的环：`connection cycle: a -> b -> a`；
- 从未配置的时钟：`top.clock: Clock was not initialized`，或者配置了两次的时钟。

**task**
- 类型与字段不一致的参数：
  `... state value has type u32, but the bound state has type u64`；
- 同一个 task 的两个参数以不同的模式访问同一个位置：
  `state parameters current and latest conflict on ...`。

**顺序**，由 Orrery 根据访问模式推出（[状态与 task](./state-and-tasks#access-modes)）：

- 两个 task 在同一个时刻点必然写同一个位置：

  ```text
  write conflict: %WriteConflict.bump at top and %WriteConflict.clear at top
  both write top.value (both triggered by clock top.clock (1ns phase 0ps))
  ```

- 总是一起运行的 task 之间形成环：

  ```text
  dependency cycle (triggered by clock top.clock ...): %Cycle.cycle_a at top
  reads top.x (curr) before %Cycle.cycle_b at top writes it; ...
  ```

  对 FIFO 来说，这就是两侧都声称自己先动作，或者都声称在后（[组件](./components#who-acts-first)）。

只是*可能*发生的冲突和环（因为涉及事件）则是警告，在仿真运行时检查：

```rust
for warning in plan.analysis().warnings() {
    eprintln!("{warning}");
}
```

## 构建仿真时

`into_sim` 在以下情况失败：`delta_limit` 超过 65,534；外部端口没有绑定或绑定了两次；配置了快照时，task 写（`&mut`）的外部端口的源对象不能保存；以及没有设置 `stop_delay`、而 Orrery 算不出模型的免费延迟时（[停止](./running#stopping)）。

## 运行时 {#while-it-runs}

以下情况 `run_until` 和 `step` 返回 `Err`，运行结束：

| 错误 | 原因 |
|---|---|
| `%Sm.cycle at top.sms[3] failed at 1200ns: ...` | 一个 task 返回了错误 |
| `... panicked: ...` | 一个 task panic 了；错误里写明是哪个 task、哪个实例 panic 的 |
| `multi-drive at 10ns+1: A and B both emit to L` | 同一个位置、同一个时刻点上有两个事件 |
| `write conflict at 37ns: A and B both write L` | 一个可能的写冲突真的发生了 |
| `dependency cycle among ...` | 可能一起运行的 task 在这个时刻点形成了顺序环 |
| `more than 30 delta cycles at 5ns` | 一串永远稳定不下来的 delta 周期（`delta_limit`） |
| `600000s is after the latest time a run with a delta limit of 30 reaches, ...` | 运行在时间上走得比 `delta_limit` 允许的更远（[delta 周期](./time#delta-cycles)） |
| `more than N events in flight from one location` | 超出了 `event_capacity` |
| `event delay ... is below min_delay ...` | 一次发出的延迟不在源声明的延迟之内 |
| `an event at ... reached the trigger ..., whose gate ... is closed, and dropped events are errors` | 开了 `fail_on_drop` 时，一个事件在关着的门控上被丢弃（[时钟门控](./time#clock-gates)） |

### 报的是哪个错误 {#which-error-is-reported}

不止一处失败时，Orrery 报串行顺序里最早的那一个：一次只跑一个 task 的运行会按什么顺序碰到它们，就按什么顺序，即逐个时刻点，在一个时刻点之内按一个符合访问模式的固定顺序。同一个模型、同一个输入，不管用几个 worker，报的都是同一个错误。

- 只会发生在 stop 终点之后的失败不报：运行按 stop 结束（[停止](./running#stopping)）。
- 开了 `fail_on_drop` 时，被丢弃的事件让它所在的时刻点失败，这个时刻点的 task 一个也不运行。
- 一个 task 发出的事件造成了多驱动，之后它又返回错误或 panic：报多驱动。

出错之后，串行顺序里在失败之前的一切都已经运行，宿主可以用 `sim.state` 读出它留下的状态。失败的 task 可能只写了一部分，在它之后的工作也可能已经运行：这一部分状态没有规定。之后的每次 `run_until` 和 `step` 都返回错误。

## 调试

- **单步运行。** `sim.step()` 运行一个时刻点；`sim.next_timepoint()` 给出是哪个。在两步之间用 `sim.state(handle)` 查看状态。
- **检查运行。** 用 `SimConfig { checks: true, .. }` 时，运行之后 `sim.check_trace()` 检查访问同一份状态、其中至少一个是写的每两次访问都按串行顺序运行了。
- **比较不同的 worker 数。** 结果，包括报出的错误，从不取决于 worker 数。如果一个模型在不同的 worker 数下给出不同的结果，那是 Orrery 的 bug，请报告。
- **快照与恢复。** 在长时间的运行中拍快照，恢复到问题之前的那一个，再用 `step`
  循环或更多的日志排查（[快照](./snapshots)）。
