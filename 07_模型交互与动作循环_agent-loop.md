# 可选模型与 LLM runner（原 agent-loop）

状态：设计草稿。`agent-loop` 是可选高层 runner 的旧模块名，不再是所有在线使用方式的必经入口；是否保留独立仓库待实现确定。

## 角色

runner 用在线 SDK 将 observation 交给模型或 LLM，组织上下文与工具调用，并将决策转为语义请求。人、搜索器或 LLM 也可直接使用 CLI/SDK，不要求经过 runner。

```text
observe → 模型输入 / LLM 上下文
        → inference / tool call / 外部 REPL
        → SDK commit / advance / checkout
        → 实际结果与证据引用 → 下一次决策
```

## 负责什么

- 模型后端适配、输入投影、动作解析和策略约束；
- LLM 对话上下文、工具交互、推理预算与停止条件；
- 模型/策略版本、任务及实验 metadata；
- 候选程序在外部的执行状态恢复或重建契约；
- 每次 commit 使用上一实际结果的 BoundaryRef，显式 advance，处理部分执行和未知结果；
- 通过 SDK 查询任务、取消和终局，返回 episode/branch 证据引用。

## 下沉与组合

session、IPC、活动分支、checkpoint 生命周期和 checkout 编排归[共享控制后端](avz/04_共享控制后端.md)。runner 可以发起 checkout，但不重复维护恢复机制。搜索器拥有搜索树、候选选择与探索记忆，可组合 runner 或直接使用 SDK。持久化材料由后端集成 loader 解析。

Runtime/后端统一产生并交付 recorder 所需执行事实；runner 可补充模型调用与策略事件，不能成为动作轨迹唯一的记录入口。

## 状态与边界

游戏状态、候选程序执行状态和求解器记忆分开。游戏回退不自动清空 LLM 的失败经验；评估配置需声明可用观察、回退与预算。外部 Python 计数器/队列若影响候选后续行为，需由 runner 恢复或重建；细节见[SDK 与 REPL](avz/03_在线SDK与REPL.md)。

runner 不启动或清理游戏，不管理原生恢复材料，不定义第二套 S，不读写游戏内存，也不构造训练 batch。原生 AvZ 脚本可以直接经在线工具运行，无需逐步模型循环。
