# pvz-agent-lab 迁移顺序与仓库变更

状态：迁移草稿；不包含 issue 的具体测试关闭条件。

## 顺序

1. **AvZ fork**：确认 RNG、FP、clock、frame gate、native state 导出和 restore/recompute 边界，登记上游差异。
2. **trajectory-recorder**：将 trajectory-core 的 schema、树、seal、读取器迁入新身份，冻结 `S`、Frame、Transition、Episode、Branch；保留旧格式适配。
3. **pvz-session**：从旧 launcher/client/runtime 提取单局会话宿主、IPC、资源所有权和清理；不搬走 AvZ 确定性控制。
4. **trajectory-loader**：先实现新 session + action replay 的 `recompute` backend，再根据 AvZ 能力接 `native_restore`。
5. **agent-loop**：从旧 agent-rollout 中迁出固定策略驱动、模型接口和分支编排，先接 session/loader/trajectory-recorder。
6. **video-recorder**：从旧 demo/视频路径中提取画面、音频和时间轴输出，不把它混进 trajectory schema。
7. **rollout-utils**：读取封存轨迹并提供数据集索引、筛选、切分和转换。
8. **train**：接 trajectory-recorder/rollout-utils，先做离线训练，再接可选在线采样。

## 旧仓库对应关系

| 旧职责 | 新归属 |
|---|---|
| native RNG/FP/clock/frame gate | `avz` |
| launcher、client、runtime 生命周期 | `pvz-session` |
| evidence_tree、records、seal、只读读取 | `trajectory-recorder` |
| 从 root/Frame 重算或恢复分支 | `trajectory-loader` |
| 模型适配、动作循环、分支编排 | `agent-loop` |
| 视频、截图、演示包 | `video-recorder` |
| 数据集目录和样本转换 | `rollout-utils` |
| reward/训练方案和配置 | `train` |

## 迁移约束

- 旧实验、旧 seal、旧轨迹保持不可变；
- 新仓库只能有一个 `S` schema 权威；
- trajectory-loader 不能通过直接改字段伪造恢复；
- video-recorder 的输出不能代替 trajectory-recorder 的状态证据；
- train 不直接依赖游戏进程；
- `pvz-session` 的一个实例只拥有一局 live game；
- 支路的执行句柄和证据树记录必须分开。
