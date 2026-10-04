# pvz-agent-lab 设计文档索引

这组文档按 repo 拆分，说明系统的心智模型、模块职责、接口方向和数据流。先读系统总览，再按模块阅读。当前内容为设计草稿。

项目将实现快速恢复版 fork（相对于历史长度近似 O(1)）和重算版 fork（相对于历史执行工作量 O(n)）两种正式能力。主要实现位于 AvZ，session 提供会话接口，loader 编排持久化轨迹加载；搜索热路径可直接使用 live checkpoint。能力范围和复杂度含义见系统总览，恢复契约见 AvZ 设计；此处描述交付目标，不表示已实现。

## 阅读顺序

1. [01_系统分层与模块边界.md](01_系统分层与模块边界.md)
2. [02_游戏内确定性控制_avz.md](02_游戏内确定性控制_avz.md)
3. [03_游戏会话与进程管理_pvz-session.md](03_游戏会话与进程管理_pvz-session.md)
4. [04_状态轨迹录制_trajectory-recorder.md](04_状态轨迹录制_trajectory-recorder.md)
5. [05_轨迹加载与执行支路_trajectory-loader.md](05_轨迹加载与执行支路_trajectory-loader.md)
6. [06_视频与演示录制_video-recorder.md](06_视频与演示录制_video-recorder.md)
7. [07_模型交互与动作循环_agent-loop.md](07_模型交互与动作循环_agent-loop.md)
8. [08_轨迹数据集管理_trajectory-dataset.md](08_轨迹数据集管理_trajectory-dataset.md)
9. [09_训练方案与配置_train.md](09_训练方案与配置_train.md)

## 模块索引

| 文档 / Repo | 关注内容 |
|---|---|
| 系统分层与模块边界 | 整体能力、语义状态、执行宿主与逻辑分支、Git 工作流类比 |
| `avz` | 游戏线程上的动作、确定性控制和原生状态导出 |
| `pvz-session` | 物理执行宿主、IPC、活动分支句柄和资源清理 |
| `trajectory-recorder` | 每 tick 状态、动作结果、事件和回执的录制与封存 |
| `trajectory-loader` | 离线轨迹加载、材料来源/执行策略矩阵、checkpoint 与轨迹的转换成本 |
| `video-recorder` | 视频、音频、时间轴和演示产物 |
| `agent-loop` | 模型推理、动作循环和分支探索编排 |
| `trajectory-dataset` | 封存轨迹的数据集索引、筛选、切分和转换 |
| `train` | 训练方案、配置、样本适配和模型产物 |

## 迁移与验收

- [迁移顺序与仓库变更](迁移顺序与仓库变更.md)：单独维护迁移步骤与旧职责的新归属。
- 验收在相应 issue 中维护，关注测试、testbench、负例和运行证据；不写入模块设计正文。

## 文档组织约定

- 设计文档平铺在仓库根目录，采用 `两位编号_中文主题_模块名.md`；总览省略模块名。
- 编号表达阅读顺序，迁移顺序以迁移文档为准。
- README 作为统一索引；每个 repo 的设计维护在对应文档中。
- 命名与索引结构参考 [HKCLR-AGV/design](https://github.com/HKCLR-AGV/design)。
