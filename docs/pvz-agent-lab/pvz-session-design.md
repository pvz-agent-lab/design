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
- 管理 `SessionHandle`、`RootHandle`、`BranchHandle` 等执行句柄；
- 处理超时、取消、断线、部分执行、未知结果和清理失败。

## 支路

session 本身只拥有它创建的那一局。trajectory-loader 可以：

- 创建新 session，从 trajectory root 重放到目标 frame；
- 调用 AvZ 的 native restore，在后端支持时建立 branch handle；
- 调用未来的独立进程 branch backend。

session 只返回执行句柄和运行结果，不定义证据树。证据树由 trajectory-recorder 记录。

## 接口心智模型

```text
observe()        读取当前稳定边界
commit(actions)  请求 runtime 在游戏线程执行语义动作
advance(n)       请求 runtime 执行 n 个实际更新
pause()          请求进入可观察的暂停边界
status()         查询 session/进程/runtime 状态
close()          关闭通道、进程和所有自有资源
```

所有结果必须保留实际执行量、版本和错误语义；session 不把部分执行伪装成成功。
