# train 模块设计

Repo：`pvz-agent-lab/train`

## 角色

train 负责训练方案、配置和优化运行。默认面向已经由 trajectory-recorder 记录并由 trajectory-dataset 整理的数据。

## 依赖

```text
train → trajectory-recorder
train → trajectory-dataset
train -. online collection .-> agent-loop
```

train 不直接依赖 AvZ 或 pvz-session。在线训练需要采样时，由 agent-loop 产生 rollout，再回到数据接口。

## 负责什么

- DQN、Q-learning、actor-critic、PPO、policy/value 和 model-based RL 配置；
- 模型、优化器、batch、reward/advantage 和 checkpoint；
- 将 trajectory-dataset 输出转换为训练框架输入；
- 训练指标、配置和模型产物的身份记录。

## 不负责什么

train 不启动游戏，不控制 RNG，不定义 `S`，不恢复 trajectory，不管理视频，也不把训练框架的内部 batch 格式写回 recorder schema。
