# AvZ fork 模块设计

Repo：`pvz-agent-lab/avz`

## 角色

AvZ fork 是唯一进入游戏进程、拥有游戏线程控制权的模块。它把上游 AvZ 的语义动作和 hook 能力扩展成可重复推进原版 PvZ 所需的确定性控制层。

它的基本心智模型是：

```text
AvZ action / runtime request
        ↓
游戏线程边界
        ↓
确定性控制器
        ↓
原版 update / draw / native call
        ↓
S + receipts
```

## 负责什么

- AvZ 语义动作和原生调用封装；
- 游戏线程、pre/update/post 边界和 single-step；
- RNG 实例、seed、游标、调用点、参数、返回值和前后状态；
- x87/SSE 浮点环境；
- GameClock、MjClock 等会影响模拟的时钟；
- 对象池、动画、粒子、音效和出怪路径的确定性控制或观测；
- 从稳定边界导出 `S` 所需的 native 状态；
- `RootCapture`、`StateRestore`、`Recompute` 或 branch primitive；
- 将动作结果和 native receipts 发布给进程外的 session。

## 不负责什么

AvZ 不负责启动或清理外部进程，不负责 IPC client，不负责模型调用、数据集、reward、训练或视频编码。它也不因为提供 restore primitive 就自动声称支持真实进程 fork。

## 输出

AvZ producer 输出三类事实：

1. `Observation`: 稳定边界上的 `S` 或构成 `S` 的 native 字段；
2. `ActionReceipt`: 动作是否进入游戏线程、是否执行、实际推进量和失败原因；
3. `NativeReceipt`: RNG、FP、clock、对象池、动画和其他已审计调用的前后证据。

session 负责传输这些事实，trajectory-recorder 负责记录这些事实。

## 分支模型

AvZ 可以提供多种后端：

- 重新从 root 重算；
- 同进程恢复语义状态和确定性控制状态；
- 独立进程或更强的 native branch。

后端差异不改变 `S` 和 receipts 的公共格式。若 restore 没有覆盖会影响 `S` 的状态，AvZ 不得把它包装成可执行分支。
