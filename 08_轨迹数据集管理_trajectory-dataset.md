# trajectory-dataset 模块设计

Repo：`pvz-agent-lab/trajectory-dataset`

## 角色

trajectory-dataset 是封存轨迹的数据集工具。它消费 trajectory-recorder 的 Episode/Branch，不参与游戏执行，也不重新定义 `S`。

## 调用方与依赖边界

调用方：

- `train`：把数据集索引、切分和转换结果输入训练 pipeline；
- search/synthesis 与评估工具：读取示范、历史证据、评估任务与显式执行材料引用；
- 数据集运维与分析脚本：只读消费方，用于构建 manifest、统计、分片、合并和导出，不要求训练框架。

依赖方向：

```text
trajectory-recorder ← trajectory-dataset ← train / search-synthesis / 离线分析
```

trajectory-dataset 只读取 trajectory-recorder 的封存格式，不依赖 AvZ、pvz-session、trajectory-loader 或任何 native runtime；离线读取数据集不要求启动游戏。

与 trajectory-loader 的分界：

- trajectory-dataset 拥有数据集目录、manifest、索引、切分和 metadata，产出数据集引用和样本；
- trajectory-loader 按显式 `trajectory_id` / `root_id` / `frame_id` 引用建立执行起点，负责材料引用解析、完整性与版本校验、重放计划编排和目标状态核对，不查询数据集的筛选、切分或 manifest；
- 数据集模块可以产出引用供 loader 使用，但不加载执行分支，也不判断材料能否恢复执行。

对外稳定的是数据集 manifest/index 格式，而不是任何训练框架的内部 batch 格式。

## 负责什么

- 数据集 manifest、索引和版本组合；
- 按任务、seed、root、branch、构建和实验范围筛选；
- 去重、过滤失败/截断/未知终局；
- train/validation/test 切分；
- step-level、trajectory-level、preference pair 等样本转换；
- 数据集统计、分片、合并和导出。

dataset metadata 可以在这里增加，但 `S`、Frame、Transition、Episode 和 Branch 的公共定义只能来自 trajectory-recorder。

## 不负责什么

trajectory-dataset 不启动游戏，不调用模型，不创建 session，不加载执行分支，不修改轨迹事实，不执行 reward 依赖的游戏动作。reward 的离线派生可以作为转换步骤，但原始游戏结果仍以 recorder 记录为准。

## 在线工具的导出入口

CLI 可以组合 dataset 构建、统计与导出命令，在线 SDK 的调用者也可离线消费封存记录。这些入口不改变 dataset 的离线依赖边界，不要求安装游戏或启动在线控制后端。train/validation/test 切分应按共享 root/探索分支关系分组，避免同一起点的近重复支路泄漏到不同集合；观察、悔棋和探索预算可用于筛选与评估条件标注。
