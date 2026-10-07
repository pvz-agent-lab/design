# pvz-agent-lab 设计文档索引

状态：设计草稿。回应 [#6](https://github.com/pvz-agent-lab/design/issues/6)，以 AvZ Runtime 与在线控制工具为共同底座。人工、LLM、模型 runner、在线搜索与程序综合使用同一套 CLI/SDK；训练和数据集工具消费执行记录。

项目将实现快速恢复 fork（相对于历史长度近似 O(1)）和重算 fork（相对于历史执行工作量 O(n)）。原生实现归 AvZ，在线后端编排 checkout，loader 解析持久化材料；复杂度不表示功能已实现，也不承诺任意轨迹冷加载为 O(1)。

## 阅读顺序

1. [系统分层与模块边界](01_系统分层与模块边界.md)
2. [AvZ Runtime 与在线控制工具目录](avz/README.md)：Runtime、原生 SDK、CLI、在线 SDK/REPL、共享后端与内部 session。
3. [状态轨迹录制](04_状态轨迹录制_trajectory-recorder.md)
4. [轨迹加载与执行支路](05_轨迹加载与执行支路_trajectory-loader.md)
5. [视频与演示录制](06_视频与演示录制_video-recorder.md)
6. [可选模型与 LLM runner](07_模型交互与动作循环_agent-loop.md)
7. [轨迹数据集离线管理](08_轨迹数据集管理_trajectory-dataset.md)
8. [训练方案与配置](09_训练方案与配置_train.md)
9. [搜索与程序综合](10_搜索与程序综合_search-synthesis.md)

## 组件索引

| 文档 / 组件 | 关注内容 |
|---|---|
| [AvZ Runtime / 原生脚本 SDK](avz/01_Runtime与原生脚本SDK.md) | 游戏内动作、调度、确定性、原生状态导出与恢复 |
| [在线 CLI](avz/02_在线CLI.md) | 终端命令、结构化结果、脚本工具链与任务入口 |
| [在线 SDK / REPL](avz/03_在线SDK与REPL.md) | 可编程客户端、外部解释器和状态边界 |
| [共享控制后端](avz/04_共享控制后端.md) | session、branch/checkout、checkpoint、脚本任务及组件集成 |
| [内部 session](avz/05_session与执行契约.md) | 物理宿主、IPC、资源、执行代次与单动作契约 |
| trajectory-recorder | 公共记录 schema、执行事实、证据与封存 |
| trajectory-loader | 持久化解析、重放计划、目标核对与转换成本 |
| video-recorder | 可选视频、音频与时间轴 |
| 可选 runner / agent-loop | 模型、LLM 与工具交互 |
| trajectory-dataset | 离线索引、切分、转换与导出 |
| train | 模型开发与训练产物 |
| search/synthesis | 在线求解、程序开发、候选评估与交付 |

## 迁移与验收

[迁移顺序与仓库变更](迁移顺序与仓库变更.md)维护职责迁移。验收、testbench、负例与运行证据在相应 issue 中维护。

## 文档组织约定

总览与独立职责文档保留根目录编号；AvZ 原生能力与在线工具文档放在 `avz/`，目录内编号表达阅读顺序。目录与文档划分不强制独立仓库或同进程部署；现有原生 SDK 暂不从 Runtime 拆仓。README 维护统一索引，迁移文档维护旧路径与职责对应关系。
