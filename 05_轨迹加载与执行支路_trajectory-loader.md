# trajectory-loader 模块设计

Repo：`pvz-agent-lab/trajectory-loader`

## 角色

trajectory-loader 把已经封存的轨迹重新变成可执行的游戏分支，服务 agent-loop 和搜索。它是“轨迹到执行实例”的适配层，不是状态编辑器。

项目同时实现快速恢复和重算两种 fork。loader 负责持久化材料的解析和后端编排；高频搜索可直接通过 session 使用 live checkpoint，无需每次封存并重新读取轨迹。恢复与确定性执行的主要实现位于 AvZ。

## 输入和输出

输入：

```text
trajectory_id
root_id / frame_id
action_plan
branch_backend
checkpoint_ref（快速恢复时提供或从轨迹引用解析）
fallback_policy（是否允许回退到重算）
```

输出：

```text
SessionHandle / BranchHandle
load_receipt
reached_frame
actual_backend / replay_work / execution_generation
```

`reached_frame` 必须由 AvZ/session 的实际状态和版本确认，不能只按重放步数推断。

## 后端

1. `native_restore`：解析兼容且有效的 checkpoint，请求 session/AvZ 恢复到目标边界。相对于历史长度 n 近似 O(1)，适用于搜索；普通 Frame 的 `S` 和 receipts 不足以自动启用此后端。
2. `recompute`：请求 session/AvZ 建立可复现 root，重放已记录动作和推进边界到目标 frame。相对于前缀工作量 n 为 O(n)，另计初始化成本，长期用于复现和恢复对照。

两者都是项目交付范围，不能以 recompute 完成替代快速恢复能力。实现可分阶段接入，loader 的公共结果格式不因 backend 改变。独立进程或 savestate 机制属于另行声明的后端能力。

可以从最近 checkpoint 恢复后短距离重算 k 步，此时成本是恢复加 O(k)，receipt 必须注明混合路径和实际推进量。调用方明确要求快速恢复时，不得静默从 root 重算。持久化轨迹的全量读取、seal 校验和新进程启动另有成本；loader 的冷加载不承诺端到端 O(1)。

root 引用必须能解析为兼容的初始化 recipe 或恢复材料；checkpoint 必须声明 session 作用域、有效期和持久化/迁移能力。缺少材料、版本不兼容、句柄过期或恢复闭包未受支持时，返回明确失败，或按调用方允许的策略回退。

## 负责什么

- 读取 trajectory-recorder 的 root/frame/action 记录；
- 校验版本、seal、动作边界和重放可达性；
- 选择或接受指定的 branch backend；
- 请求 pvz-session 建立执行实例；
- 将加载失败、部分重放和目标 frame 不匹配如实返回。

## 不负责什么

loader 不直接读写游戏内存，不实现 RNG/FP/clock 控制，不定义 `S`，不创建 EvidenceTree 节点，也不把修改几个字段当作状态恢复。
