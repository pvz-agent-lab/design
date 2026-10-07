# trajectory-loader 模块设计

Repo：`pvz-agent-lab/trajectory-loader`

## 角色

trajectory-loader 定位为离线轨迹加载器：从持久化轨迹解析材料并建立可执行起点，作为共享控制后端的组件服务 CLI、SDK 和上层搜索。“离线”描述材料来源，不表示加载时不运行游戏；在线 agent 也可以调用它加载历史起点。名称与独立建仓方式暂沿用当前规划，是否进一步拆包待接口与实验结果确定。

项目同时实现快速恢复和重算两种 fork。loader 负责 root/frame/action 与 artifact 引用解析、证据完整性与版本检查、执行计划编排和目标边界核对，使 AvZ 无需理解 dataset 目录、seal 或轨迹存储格式。在线分支与 checkout 由共享控制后端编排，同样通过内部 session 调用 AvZ 原语，不要求每个候选都经过封存和文件加载。若实现最终仅剩 restore 转发，则不必保留独立 loader repo。

## 材料来源与执行策略

| 入口/材料来源 | 快速恢复 native_restore | 重算 recompute |
|---|---|---|
| 在线搜索 | 有效 live checkpoint | 可重建 root 与仍保留的动作/推进历史 |
| 持久化轨迹加载 | 兼容且允许导入的 checkpoint artifact | root 材料与轨迹中的动作/推进历史 |

两个维度独立，每个组合是否可用取决于材料与 runtime 能力。在线不等于快速恢复，离线不等于重算；同 session checkpoint 的支持也不代表支持持久化导入。

## live checkpoint、traj 与转换路径

- `X_t`：游戏和 AvZ 在边界 t 的完整活动执行状态。
- `C_t`（live checkpoint）：AvZ 在受支持边界捕获、可重复恢复的材料，由 session 管理其句柄和寿命。它服务 native_restore，不是活动游戏实例；具体是复制、共享状态还是其他表示，尚待实验确定。
- `checkpoint_artifact_t`：边界 t 的可选持久化恢复材料，只有 AvZ 明确提供兼容的导出/导入能力时存在；普通句柄不能直接序列化后跨进程使用。
- traj：包含语义状态 S 的完整轨迹，保留有序动作、推进边界与执行结果/证据，可引用额外 checkpoint artifact；语义状态轨迹与完整 native 恢复材料的覆盖范围需分别验证。

必须分开描述时间轴上的实际执行和状态表示的变换：

```text
root 状态 X_0 --按完整 traj 的动作/推进记录执行，并核对 S--> X_t
X_t --capture--> C_t --restore--> X_t'  同一逻辑边界的捕获/恢复
C_t --export（若支持）--> checkpoint_artifact_t
checkpoint_artifact_t --import（若支持）--> C_t'  表示与资源转换
```

最后一条不指定实现必须创建中间 C_t' 对象；AvZ 也可支持直接将 checkpoint artifact 恢复到活动 runtime。即使格式转换不推进模拟时间，仍可能需要解码、资源重建和兼容性校验。

完整 traj 没有额外的 native 恢复材料时，loader 从中派生 root、动作与推进的临时重放计划，重算到 X_t，并使用记录的 S 核对沿途与目标结果，之后按需 capture 成 C_t。这个过程包含实际模拟，不是 JSON 到内存的纯格式变换。临时计划不替代完整 traj，也不作为新的薄轨迹类型持久化。

包含 S 不自动证明覆盖了完整恢复契约；若将来证明 S 及其配套控制状态已足够恢复，可启用相应直接恢复能力。在此之前不能把某帧 S 自动转成 checkpoint。反过来，C_t 可导出当前观察或受支持的 checkpoint artifact，但不能凭一个 checkpoint 还原过去的全部动作和 receipts；附加 artifact 也不替代 traj 的语义状态记录。

可省去的步骤取决于已有材料：

- 在线已有 C_t：直接 restore，省去 traj 封存/读取、export/import 和历史重放；仍需恢复结果确认。
- 已在 X_t：直接执行子分支；若后续需返回该边界，先 capture 并保留 C_t。
- 完整 traj 附带兼容 checkpoint artifact：加载并恢复，核对目标 S 和恢复契约要求的状态，省去 root 到 t 的重放；能否省去中间 live 表示由实现决定。
- 完整 traj 未附额外恢复材料：按其动作与推进记录重算并核对 S，必要时 capture 并缓存，将首次重算成本摊到后续搜索；已有较早 checkpoint 时只重放短前缀。

因此首次物化与重复 checkout 类操作分别计费：近似 O(1) 是有效 checkpoint 已具备时、相对于历史长度的恢复目标，不是任意 traj 首次加载的承诺。

## 待实验确定的表示与成本

checkpoint 表示、是否支持 export/import、以及中间表示能否省略，由 [AvZ 实验](avz/01_Runtime与原生脚本SDK.md#checkpoint-表示与成本实验)确定。loader 配合测量材料读取、校验、编排与首次建立执行起点的端到端成本，再决定缓存与格式转换路径；尚未承诺 traj 与 checkpoint 可低成本互转。

## 输入和输出

输入：

```text
trajectory_id
root_id / frame_id
fork_strategy
checkpoint_ref（快速恢复时提供或从轨迹引用解析）
fallback_policy（是否允许回退到重算）
```

输出：

```text
SessionHandle / BranchHandle
load_receipt
reached_frame
actual_strategy / replay_work / execution_generation
```

`reached_frame` 必须由 AvZ/session 的实际状态和版本确认，不能只按重放步数推断。

历史执行计划由 loader 从完整 traj 派生，不把 action_plan 当作独立持久化格式或可覆盖轨迹历史的调用参数。重放保留每个动作的实际顺序、tick 内阶段和前后状态，逐个调用单动作 commit，再按记录调用 advance；使用新执行实例的 BoundaryRef 和请求身份，与历史记录保持关联，而不直接复用历史 session 的在线令牌。记录状态用于逐边界核对，失败、部分执行或顺序不符必须如实返回。

## 分支建立策略

1. `native_restore`：解析兼容且有效的 checkpoint，请求 session/AvZ 恢复到目标边界。相对于历史长度 n 近似 O(1)，适用于搜索；普通 Frame 的 `S` 和 receipts 不足以自动启用此策略。
2. `recompute`：请求 session/AvZ 建立可复现 root，重放已记录动作和推进边界到目标 frame。相对于前缀工作量 n 为 O(n)，另计初始化成本，长期用于复现和恢复对照。

两者都是项目交付范围，不能以 recompute 完成替代快速恢复能力。实现可分阶段接入，loader 的公共结果格式不因策略改变。AvZ 提供底层原语，独立进程或 savestate 机制属于另行声明的实现能力。

可以从最近 checkpoint 恢复后短距离重算 k 步，此时成本是恢复加 O(k)，receipt 必须注明混合路径和实际推进量。调用方明确要求快速恢复时，不得静默从 root 重算。持久化轨迹的全量读取、seal 校验和新进程启动另有成本；loader 的冷加载不承诺端到端 O(1)。

root 引用必须能解析为兼容的初始化 recipe 或恢复材料；checkpoint 必须声明 session 作用域、有效期和持久化/迁移能力。缺少材料、版本不兼容、句柄过期或恢复闭包未受支持时，返回明确失败，或按调用方允许的策略回退。

## 负责什么

- 读取 trajectory-recorder 的 root/frame/action 记录；
- 校验版本、seal、动作边界和重放可达性；
- 选择或接受指定的 fork_strategy；
- 通过共享后端内部 session 接口建立执行实例；
- 将加载失败、部分重放和目标 frame 不匹配如实返回。

## 不负责什么

loader 不直接读写游戏内存，不实现 RNG/FP/clock 控制，不定义 `S`，不创建 EvidenceTree 节点，也不把修改几个字段当作状态恢复。

## 与在线控制后端、数据集的边界

loader 负责持久化材料解析、版本/完整性校验、重放计划编排与目标状态核对；后端拥有活动 session、分支身份和资源生命周期。loader 使用后端内部执行接口，不再次调用公共 checkout 入口。

dataset 负责目录、manifest、筛选和切分，可以解析出显式 trajectory/root/frame 引用；loader 接受这些引用及材料定位信息，不查询数据集筛选策略。只有在线 live 材料时无需经过持久化 loader。
