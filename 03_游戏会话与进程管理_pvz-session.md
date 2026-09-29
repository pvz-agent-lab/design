# pvz-session 模块设计

Repo：`pvz-agent-lab/pvz-session`

## 角色

`pvz-session` 是物理执行宿主和 IPC 客户端。一个 Session 实例拥有一个 `popcapgame1.exe`、该进程内的一套固定 AvZ runtime、一个控制通道和一套资源/清理责任。同一时刻它承载一个活动逻辑分支，可通过恢复先后承载不同分支；session 数量表示执行资源数量，不等于逻辑对局或分支数量。

它不是泛称的 RL environment，也不在自己进程里模拟 PvZ。

模块不等同于一个全局类：Session 实例只操作自身资源，launch 等工厂入口负责创建实例。多个 session 由调用方持有，需要 worker 池时再提供显式的 SessionPool；Session 类不默认维护所有会话的全局注册表。Git 类比中，session 接近 worktree，当前活动分支接近 checkout/HEAD，AvZ 则是实际操作游戏状态的 runtime。

## 进程和通信

```text
agent-loop / trajectory-loader
        │  本机 IPC 请求
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

session 向 agent-loop 和 trajectory-loader 暴露两种分支建立策略所需的原语。策略字段称为 `fork_strategy`，取值为 `native_restore` 或 `recompute`；AvZ 是提供原语的 runtime，策略不是另一个 runtime。

- 快速恢复 `native_restore`：转发 AvZ checkpoint capture/restore，在同一 session 中串行探索不同后续动作，避免每个候选都冷启动和重放完整前缀；
- 重算 `recompute`：建立可复现 root，接受在线分支控制或离线轨迹加载器编排的动作与推进请求，执行到目标 frame；需要隔离执行时新建 session。

session 返回 AvZ 原语的支持范围和实际操作 receipts，不自行选择搜索策略、编排轨迹历史或实现恢复闭包。恢复会切换当前活动分支；旧执行代次的请求/句柄必须拒绝，未释放且仍有效的 checkpoint 可重复使用。恢复失败后阻止 commit/advance，直到重新建立并确认有效执行状态。

并行分支由多个 session/worker 承担。跨 session 传递 checkpoint 需要 AvZ 明确支持；同进程恢复能力不意味着可迁移。快速恢复不可用时返回明确错误，由上层在调用方允许时编排 recompute，并报告实际策略与重放工作量。

session 只返回执行句柄和运行结果，不定义证据树。证据树由 trajectory-recorder 记录。

## 接口心智模型

```text
observe()        读取当前稳定边界
commit(actions)  请求 runtime 在游戏线程执行语义动作
advance(n)       请求 runtime 执行 n 个实际更新
pause()          请求进入可观察的暂停边界
capture()        请求 AvZ 捕获当前稳定边界 checkpoint
restore(ref)     请求 AvZ 恢复 checkpoint，确认状态并切换执行代次
release(ref)     释放 checkpoint 及其关联资源
status()         查询 session/进程/runtime 状态
close()          关闭通道、进程和所有自有资源
```

上述为设计接口心智模型，不表示当前均已实现。所有结果必须保留实际执行量、版本和错误语义；session 不把部分执行伪装成成功。
