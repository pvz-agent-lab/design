# 设计变更记录

本文件记录目标架构的变化、历史名称和路径迁移；模块正文描述当前设计。条目不代表功能已经实现。

## 2026-10-07 — 在线控制系统与共享后端

关联 [issue #6](https://github.com/pvz-agent-lab/design/issues/6)、[PR #7](https://github.com/pvz-agent-lab/design/pull/7)，承接 [PR #5](https://github.com/pvz-agent-lab/design/pull/5) 的 dataset 命名。

- 共同底座由以 agent-loop 为中心的在线循环调整为 AvZ Runtime + 在线控制工具，CLI 与 SDK 共享控制后端。
- pvz-session 的宿主/IPC/资源职责成为后端内部 session 层；agent-loop 的通用分支与 checkout 编排移入后端，模型/LLM 交互留在可选 runner。
- AvZ 原有 C++ SDK、工具链与 Runtime 同属 AvZ fork；在线动作增加外部持续交互入口，不要求 C++ REPL。
- recorder、loader、video 作为后端集成组件，保持各自职责；外部 REPL 使用 SDK，探索记忆与可恢复执行状态分开。
- dataset 的程序内调用方从 train 扩展到搜索、综合与评估；修正 loader 描述，明确其承担重放编排和目标核对。
- 新增详细 Mermaid 架构图、组件状态归属表，以及共享后端对象、checkout 和任务/资源设计；Git 类比仅解释引用、历史与物化，不引入通用状态合并。
- 架构正文移除历史模块归属和迁移叙述，集中在本文件；交付文档聚焦实现顺序与约束。

## 旧职责与新归属

| 旧职责 | 新归属 |
|---|---|
| AvZ 原生 C++ SDK、编译/注入能力 | AvZ fork 保留实现，在线工具编排构建/装载 |
| RNG/FP/clock/frame gate、原生动作 | AvZ Runtime |
| root 重建、恢复闭包、capture/restore 与受控重算原语 | AvZ Runtime |
| pvz-session / 旧 launcher-client 的进程、IPC 与资源生命周期 | 共享后端内部 session 层 |
| agent-loop 的通用分支、checkpoint 预算与 checkout 编排 | 共享控制后端 |
| 搜索树、候选选择、探索记忆与搜索预算分配 | 上层 search/synthesis 或 LLM 求解器 |
| agent-loop 的模型适配、上下文和动作循环 | 可选 runner |
| evidence_tree、records、seal、只读读取 | trajectory-recorder |
| 持久化材料解析、重放计划与目标核对 | trajectory-loader，作为后端组件 |
| 视频、截图、演示包 | 可选 video-recorder |
| rollout-utils 的目录、索引、样本转换 | trajectory-dataset |
| reward、模型训练配置与产物 | train |
| 候选程序开发、评估与交付 | search/synthesis |

## 文档路径迁移

- 原根目录 `02_游戏内确定性控制_avz.md` 移至 `avz/01_Runtime与原生脚本SDK.md`。
- 原根目录 `03_游戏会话与进程管理_pvz-session.md` 移至 `avz/05_session与执行契约.md`。
- `avz/README.md` 索引 CLI、SDK/REPL、共享后端及原生能力。
- 根目录 07 保留路径，内容改为可选 runner；新增 10 描述搜索与程序综合。编号允许留空，不为路径连续性重排其他文件。

- `迁移顺序与仓库变更.md` 改为 `交付顺序与约束.md`，历史映射移入本文件。

## 历史命名与兼容

- `rollout-utils` 在 PR #5 中命名为 `trajectory-dataset`，表达封存轨迹的离线管理职责。
- `pvz-session`、`agent-loop` 作为历史模块名和部分文件路径继续用于查阅与追溯；正文使用内部 session 和可选 runner 表达当前职责。
- trajectory-core 的 schema、证据树、seal 与读取能力规划归 trajectory-recorder；`lvz.*` 等历史格式以显式适配读取，旧实验、轨迹和 seal 保持不可变。
- 对 [issue #3](https://github.com/pvz-agent-lab/design/issues/3) 的程序综合讨论扩展为在线控制架构；search/synthesis 是工具调用方，LLM 可作为多个角色的后端。
