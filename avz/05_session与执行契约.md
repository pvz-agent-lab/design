# 后端内部 session 与执行契约

归属：在线控制后端的宿主/IPC 层；旧模块名 `pvz-session`，独立包/仓库安排待定。

## 角色

`pvz-session` 是物理执行宿主和 IPC 客户端。一个 Session 实例拥有一个 `popcapgame1.exe`、该进程内的一套固定 AvZ runtime、一个控制通道和一套资源/清理责任。同一时刻它承载一个活动逻辑分支，可通过恢复先后承载不同分支；session 数量表示执行资源数量，不等于逻辑对局或分支数量。

它不是泛称的 RL environment，也不在自己进程里模拟 PvZ。

Session 层不等同于整个后端；后端可有显式会话注册表和 worker 池。模块不等同于一个全局类：Session 实例只操作自身资源，launch 等工厂入口负责创建实例。多个 session 由调用方持有，需要 worker 池时再提供显式的 SessionPool；Session 类不默认维护所有会话的全局注册表。Git 类比中，session 接近 worktree，当前活动分支接近 checkout/HEAD，AvZ 则是实际操作游戏状态的 runtime。

## 进程和通信

```text
CLI / SDK
        │  控制后端（含 checkout 与 loader 集成）
        │  内部执行请求
        ▼
pvz-session client
        │  启动/拥有 popcapgame1.exe
        │  连接游戏内 AvZ runtime
        ▼
AvZ runtime server in game process
        │  游戏线程执行
        ▼
原版 PvZ
```

请求方向是 client → session/runtime，结果方向是 runtime → session → caller。session 不把 `advance(n)` 当成自己的模拟循环；它请求 runtime 执行 `n` 个实际受控更新。

## 负责什么

- 启动固定游戏、DLL、资源和隔离用户档；
- 创建并持有进程、IPC 通道、句柄、线程和临时文件；
- 向 AvZ runtime 转发 `observe`、`commit`、`advance`、`pause`、`status`、`close`；
- 返回观察、动作结果、实际推进量、终局状态和 receipts；
- 管理 `SessionHandle`、`RootHandle`、`CheckpointHandle`、`BranchHandle` 等句柄的所属关系、执行代次和释放；
- 处理超时、取消、断线、部分执行、未知结果和清理失败。

## 支路

session 向共享后端的 checkout 管理器和 loader 适配层暴露两种分支建立策略所需的原语。策略字段称为 `fork_strategy`，取值为 `native_restore` 或 `recompute`；AvZ 是提供原语的 runtime，策略不是另一个 runtime。

- 快速恢复 `native_restore`：转发 AvZ checkpoint capture/restore，在同一 session 中串行探索不同后续动作，避免每个候选都冷启动和重放完整前缀；
- 重算 `recompute`：建立可复现 root，接受共享后端或离线轨迹加载器编排的动作与推进请求，执行到目标 frame；需要隔离执行时新建 session。

session 返回 AvZ 原语的支持范围和实际操作 receipts，不自行选择搜索策略、编排轨迹历史或实现恢复闭包。恢复会切换当前活动分支；旧执行代次的请求/句柄必须拒绝，未释放且仍有效的 checkpoint 可重复使用。恢复失败后阻止 commit/advance，直到重新建立并确认有效执行状态。

并行分支由多个 session/worker 承担。跨 session 传递 checkpoint 需要 AvZ 明确支持；同进程恢复能力不意味着可迁移。快速恢复不可用时返回明确错误，由后端在调用方允许时编排 recompute，并报告实际策略与重放工作量。

session 只返回执行句柄和运行结果，不定义证据树。证据树由 trajectory-recorder 记录。

## 接口心智模型

```text
observe()                                     读取当前稳定边界
commit(action, expected_boundary, request_id)   请求执行一个语义动作
advance(n)                                    请求 n 个实际更新
pause()                                       请求进入可观察的暂停边界
capture() -> CheckpointHandle                  捕获当前实际边界的恢复材料
restore(checkpoint: CheckpointHandle)          恢复并确认状态，切换执行代次
release(checkpoint: CheckpointHandle)          释放恢复材料及其关联资源
status()                                      查询 session/进程/runtime 状态
close()                                       关闭通道、进程和所有自有资源
```

上述为设计接口心智模型，不表示当前均已实现。所有结果必须保留实际执行量、版本和错误语义；session 不把部分执行伪装成成功。

## 单动作提交契约

```text
commit(
    action: Action,
    expected_boundary: BoundaryRef,
    request_id: RequestId
) -> ActionReceipt
```

| 参数 | 含义与约束 |
|---|---|
| action | 一个有类型的操作及该类型定义的参数，例如 shovel 的目标位置或 plant 的卡槽与位置；字段含义固定，不接受动作集合 |
| expected_boundary | session_id、execution_generation、tick、稳定阶段 phase 和 state_version；同一 tick 内状态版本仍可变化 |
| request_id | 在该 session 的执行代次内唯一，标识一次提交，用于关联请求、执行结果和证据；不能用新 ID 重发结果未知的动作来冒充安全重试 |

BoundaryRef 是在线执行的并发校验令牌；FrameRef 是轨迹中的记录身份；CheckpointHandle 是受生命周期约束的恢复材料引用。producer 负责关联它们，调用方不能把某个历史 FrameRef 当成当前执行令牌或 checkpoint。

ActionReceipt 保留 request_id、执行前后 BoundaryRef、动作类型及参数、实际执行结果与已完成工作、失败或未知原因。AvZ 在游戏线程验证预期边界；旧执行代次或旧状态版本的请求必须在动作产生副作用前拒绝。

一个动作完成并确认稳定边界后，调用方才使用返回的新边界提交下一个动作。执行期间不夹入另一个外部动作、capture/restore 或 tick update。commit 不请求 update，每次动作改变状态时 state_version 更新，tick 可以不变；advance 才请求推进 tick。例如先 shovel 再 plant 和先 plant 再 shovel，即使随后都 advance(1)，仍是不同的动作路径。并发请求不能靠网络到达顺序替调用方定义意图，必须逐个匹配预期边界。

“不可交错执行”不等于“失败自动回滚”。每种动作要声明可能产生的部分副作用；未知结果不得自动重试，无法确认实际边界时先阻止后续变更。基础接口不提供批量 actions，未来如需批量优化，另行定义顺序、逐项结果、失败停止及 update 插入规则。
