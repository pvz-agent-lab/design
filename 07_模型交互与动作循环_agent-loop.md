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
- control、intervention、rerun 和 search branch 编排；
- 搜索热路径通过 session capture/restore 建立候选执行分支；从持久化轨迹恢复或重算时调用 trajectory-loader；
- 显式选择快速恢复或重算以及回退策略，管理 checkpoint 预算与释放；串行搜索复用 session，并行搜索使用多个 worker，不把临时 checkpoint 假定为可跨进程迁移；
- 将 producer 事件和动作结果交给 trajectory-recorder；
- 输出 rollout/episode 引用。

## 不负责什么

agent-loop 不实现 RNG、FP、时钟或内存控制，不定义第二套 `S`，不直接启动游戏进程，不读写内存，也不构造训练 batch。搜索缓存和树策略可以作为 agent-loop 内部模块或独立 search repo，但不进入 recorder。
