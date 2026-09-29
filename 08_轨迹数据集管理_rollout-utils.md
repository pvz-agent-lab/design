# rollout-utils 模块设计

Repo：`pvz-agent-lab/rollout-utils`

## 角色

rollout-utils 是封存轨迹的数据集工具。它消费 trajectory-recorder 的 Episode/Branch，不参与游戏执行，也不重新定义 `S`。

## 负责什么

- 数据集 manifest、索引和版本组合；
- 按任务、seed、root、branch、构建和实验范围筛选；
- 去重、过滤失败/截断/未知终局；
- train/validation/test 切分；
- step-level、trajectory-level、preference pair 等样本转换；
- 数据集统计、分片、合并和导出。

dataset metadata 可以在这里增加，但 `S`、Frame、Transition、Episode 和 Branch 的公共定义只能来自 trajectory-recorder。

## 不负责什么

rollout-utils 不启动游戏，不调用模型，不创建 session，不加载执行分支，不修改轨迹事实，不执行 reward 依赖的游戏动作。reward 的离线派生可以作为转换步骤，但原始游戏结果仍以 recorder 记录为准。
