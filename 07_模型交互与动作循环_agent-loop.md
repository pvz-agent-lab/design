# agent-loop 模块设计

Repo：`pvz-agent-lab/agent-loop`

## 角色

agent-loop 是模型、搜索和游戏会话之间的在线交互循环。它把 observation 交给模型或搜索器，把结果解析成语义动作，再向 pvz-session 请求执行。

## 一次循环

```text
observe
  → build model input
  → model.inference / search
  → parse action
  → pvz-session.commit / advance
  → receive result and S projection
  → trajectory-recorder
```

模型可以是本地推理器、OpenAI Responses API 或其他后端。agent-loop 只使用公开的 session API，不访问游戏地址。

## 负责什么

- observation 到模型输入的组织；
- model inference、动作解析和策略约束；
- request ID、版本、超时、取消和终局处理；
- 每次 commit 提交一个明确动作，使用上一实际结果的 BoundaryRef 串行提交，保留同 tick 内动作顺序；需要推进时显式调用 advance，不将多个候选动作合并成无序集合；
- control、intervention、rerun 和 search branch 编排；
- 在线分支控制根据有效 checkpoint 或保留的 root/动作历史选择恢复或重算，经 session 调用 AvZ；从持久化轨迹建立执行起点时调用离线轨迹加载器 trajectory-loader；
- 显式选择快速恢复或重算以及回退策略，管理 checkpoint 预算与释放；串行搜索复用 session，并行搜索使用多个 worker，不把临时 checkpoint 假定为可跨进程迁移；
- 将 producer 事件和动作结果交给 trajectory-recorder；
- 输出 rollout/episode 引用。

在线分支控制负责搜索树、候选策略和执行计划，session 负责物理宿主与原语调用，两者不重复实现恢复逻辑。checkpoint 缓存按首次生成成本、重复恢复成本和内存预算管理；不能把从 traj 重算生成 checkpoint 当成免费的格式转换。暂不强制把在线分支控制单独建仓。

## 不负责什么

agent-loop 不实现 RNG、FP、时钟或内存控制，不定义第二套 `S`，不直接启动游戏进程，不读写内存，也不构造训练 batch。搜索缓存和树策略可以作为 agent-loop 内部模块或独立 search repo，但不进入 recorder。
