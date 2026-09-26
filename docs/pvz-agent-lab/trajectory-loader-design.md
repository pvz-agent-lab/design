# trajectory-loader 模块设计

Repo：`pvz-agent-lab/trajectory-loader`

## 角色

trajectory-loader 把已经封存的轨迹重新变成可执行的游戏分支，服务 agent-loop 和搜索。它是“轨迹到执行实例”的适配层，不是状态编辑器。

## 输入和输出

输入：

```text
trajectory_id
root_id / frame_id
action_plan
branch_backend
```

输出：

```text
SessionHandle / BranchHandle
load_receipt
reached_frame
```

`reached_frame` 必须由 AvZ/session 的实际状态和版本确认，不能只按重放步数推断。

## 后端

1. `recompute`：新建 pvz-session，从 root 重放已记录动作到目标 frame；
2. `native_restore`：请求 AvZ 恢复语义状态和确定性控制状态；
3. `process_branch`：未来的独立进程或 savestate backend。

第一版优先实现 `recompute`，因为它不要求先完成任意时间点 restore。loader 的公共结果格式不因 backend 改变。

## 负责什么

- 读取 trajectory-recorder 的 root/frame/action 记录；
- 校验版本、seal、动作边界和重放可达性；
- 选择或接受指定的 branch backend；
- 请求 pvz-session 建立执行实例；
- 将加载失败、部分重放和目标 frame 不匹配如实返回。

## 不负责什么

loader 不直接读写游戏内存，不实现 RNG/FP/clock 控制，不定义 `S`，不创建 EvidenceTree 节点，也不把修改几个字段当作状态恢复。
