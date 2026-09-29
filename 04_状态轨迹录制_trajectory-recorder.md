# trajectory-recorder 模块设计

Repo：`pvz-agent-lab/trajectory-recorder`

## 角色

trajectory-recorder 是根据 `S` 记录每 tick/稳定边界轨迹的模块。它是事实记录器和离线证据包，不是游戏控制器、transition 执行器或搜索器。

`Transition` 在这里是一个记录对象：

```text
TransitionRecord = {
    state_before,
    action,
    state_after,
    receipts,
    execution_result,
}
```

它不计算或执行这个 transition。

## producer 和 recorder

AvZ/session/agent-loop 是 producer。它们决定：

- 哪个边界开始记录；
- 哪个动作已经提交；
- 哪个 native 事件发生；
- 是否开始、切换或结束一段记录。

trajectory-recorder 保存 producer 发来的 `root_started`、`tick_recorded`、`action_recorded`、`receipt_recorded`、`segment_started`、`branch_started`、`segment_closed` 和 `terminal` 等事实。它不解释这些事件的游戏原因。

## 记录对象

- `S`：语义状态的唯一公共 schema；
- `Frame`：某个稳定边界的状态记录；
- `ActionRecord`：请求、参数、版本和实际执行结果；
- `Receipt`：RNG/native/FP/clock/对象池/动画等 producer 证据；
- `TransitionRecord`：相邻状态和动作的记录关系；
- `Episode`：一局或一段连续记录；
- `BranchRecord`：父 frame、分支动作和子轨迹的证据关系。

快速恢复和重算的分支都使用上述公共记录。recorder 保存 producer 提供的 checkpoint 引用、父 frame、兼容性身份、作用域、实际 backend 和恢复/重放 receipts；checkpoint 的原生 payload 与恢复闭包由 AvZ 定义。`S` 或 native receipts 本身不承诺可以恢复执行。

持久化包若声明支持恢复，必须封存其所引用的可持久化恢复材料并纳入完整性关系；仅进程内有效的 checkpoint handle 要明确记录为临时引用，不能在 session 结束后仍宣称可加载。离线读取轨迹不要求启动 native runtime。

## 负责什么

- 每 tick/边界的结构化写入；
- 状态、动作、观察、实际推进量和终止原因的关联；
- root、parent、branch、segment 的记录关系；
- 内容身份、哈希链、seal、只读加载和离线校验；
- 旧 `lvz.*` 格式的显式适配。

## 不负责什么

它不启动游戏，不调用 AvZ，不控制 RNG，不执行动作，不推进 tick，不创建执行分支，不决定事件语义，不调用模型，不计算 reward，不管理视频，也不做训练采样。

live recorder 可以存在于 session 外部，也可以由 session 转发 AvZ runtime 的 producer 事件；这不改变 recorder 只记录事实的职责。
