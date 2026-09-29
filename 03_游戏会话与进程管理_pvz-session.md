# pvz-session 模块设计

Repo：`pvz-agent-lab/pvz-session`

## 角色

`pvz-session` 是单局游戏会话宿主和 IPC 客户端。一个 session 实例拥有一局 live game：一个 `popcapgame1.exe`、一个固定 AvZ runtime、一个控制通道和一套资源/清理责任。

它不是泛称的 RL environment，也不在自己进程里模拟 PvZ。

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

session 本身只拥有它创建的那一局，并向 agent-loop 和 trajectory-loader 暴露两种必需的 fork 能力：

- 快速恢复 `native_restore`：转发 AvZ checkpoint capture/restore，在同一 session 中串行探索不同后续动作，避免每个候选都冷启动和重放完整前缀；
- 重算 `recompute`：建立可复现 root，接受 loader 编排的动作与推进请求，执行到目标 frame；需要隔离执行时新建 session。

session 返回 backend 的支持范围、实际使用方式和 receipts，不自行实现恢复闭包。恢复会切换当前活动分支；旧执行代次的请求/句柄必须拒绝，未释放且仍有效的 checkpoint 可重复使用。恢复失败后阻止 commit/advance，直到重新建立并确认有效执行状态。

并行分支由多个 session/worker 承担。跨 session 传递 checkpoint 需要 AvZ 明确支持；同进程恢复能力不意味着可迁移。快速恢复不可用时返回明确错误，仅在调用方允许时回退到 recompute，并报告实际 backend 与重放工作量。

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
