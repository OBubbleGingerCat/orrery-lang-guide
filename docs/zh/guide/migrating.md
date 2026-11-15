# 从上一版迁移

这份指南对应 Orrery v2。为上一版写的模型，模块、task、参数和访问方式都不用改；变的是仿真怎么构建、怎么配置，时间的单位，以及几个名字。这一页把它们列出来。下面没有给替代的名字，就是没有替代：删掉即可。

## 构建和运行

| 上一版 | v2 |
|---|---|
| `plan.into_sim(config)` | `orr::Sim::new(plan, config)` |
| `plan.into_sim_with(config, log, bind)` | `orr::Sim::with(plan, config, event_log, bind)` |
| `sim.log_events_to(sink)` | 没有了：事件日志只能在构建仿真时给，即 `Sim::with` 的 `event_log` 参数（[事件日志](./running#event-log)）；`close_event_log` 还在 |
| `sim.state_mut(gate)` | `sim.gate_mut(gate)` |
| `plan.analysis().warnings()` | `sim.load_report().warnings`（[警告](./errors#warnings)） |
| `SimPlan::new` 报写冲突和顺序环 | 由 `Sim::new`（和 `Sim::with`）报（[报错](./errors#when-the-simulation-loads)） |
| `SnapshotEvery::Cycles { clock, cycles }` | `SnapshotEvery::cycles(clock, cycles)`，`cycles` 是 `NonZeroU64` |
| 快照文件按皮秒命名 | 按飞秒命名，`<20 位数字>.fs.orrs`；`orr::snapshot_path` 给出文件名 |

## 配置

| 上一版 | v2 |
|---|---|
| `workers: N`，运行 task 的线程数 | `workers_per_group`，**数法不同**：它是这次运行的全部线程，其中一个负责分派工作。`workers: 4` 相当于 `workers_per_group: 5`（[线程](./running#threads)） |
| `overlap` | 没有了，不需要替代 |
| `stop_delay` | 没有了：stop 延迟由 Orrery 从模型算出，用 `sim.stop_delay()` 读。算不出免费延迟的模型不再构建失败：它的 stop 延迟按 0，装载报告里会写明 |
| `skip_closed` | 没有了：所有受门控的时钟都关着的沿会自动越过 |
| `lead_limit`、`wait`、`placement`、`pin`、`topology`、`steal`，以及模块 `orr::exec`、`orr::plan` | 没有了：由 Orrery 自己定。`cpus` 用来限定多线程运行可以用哪些 CPU |
| `job_log`、`account`、`sim.account()` | 没有了。最接近的是 `reports: orr::Reports::All`、`sim.load_report()` 和 `sim.stop_report()` |
| `delta_limit` 最大 65,534 | 最大 62，默认 30 |

用户要设的 `SimConfig` 字段见[配置](./running#configuration)。

## 时间

| 上一版 | v2 |
|---|---|
| `Time` 是整数皮秒 | 是整数飞秒。`Time::ps`、`from_ps`、`as_ps` 仍然按皮秒收发，整皮秒是精确的；`Time::fs`、`as_fs` 是新的 |
| `Timepoint::at(ps)` | 不变，单位是皮秒；`Timepoint::at_fs` 按飞秒 |
| 一次运行能走多远取决于 `delta_limit` | 两者无关：仿真时间到 `u64::MAX` 飞秒为止，约 5.1 小时（[时间](./time)） |

## 门控

| 上一版 | v2 |
|---|---|
| `orr::Gate::default()`、对 `&mut orr::Gate` 用 `std::mem::take` | 编不过：仿真之外造不出门控（[时钟门控](./time#clock-gates)） |
| 宿主用 `sim.state_mut(gate)` 开门控 | `sim.gate_mut(gate)?.open()` |
