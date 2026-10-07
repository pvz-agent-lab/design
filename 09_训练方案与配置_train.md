# train 模块设计

Repo：`pvz-agent-lab/train`

## 角色与依赖

train 拥有模型、优化器、训练方案与模型产物，默认读取 recorder 封存、trajectory-dataset 整理的数据。在线采样通过在线 SDK，可组合可选 runner，不直接管理游戏进程或恢复闭包。

```text
train → trajectory-dataset → trajectory-recorder
train → trajectory-recorder（按需读取公共记录）
train -. online collection .-> 在线 SDK → 共享控制后端 → AvZ Runtime
训练好的 policy/value → 可选 runner / SDK 调用方 → 在线控制工具
```

## 负责什么

- DQN、Q-learning、actor-critic、PPO、policy/value 和 model-based RL 配置；
- 模型、优化器、batch、reward/advantage 和模型 checkpoint；
- 将 dataset 输出转换为训练框架输入；
- 训练指标、配置、数据集/策略版本和模型产物身份；
- 在线采样任务的策略与预算配置，消费实际执行结果与轨迹引用。

模型 checkpoint 与游戏恢复 checkpoint 是不同产物，不互换。任务明确观察、悔棋和探索权限，样本筛选及验证不得混淆探索证据与策略实际可用信息。

## 边界

train 不定义 S，不实现原生动作、RNG、恢复或 session/分支管理，不管理视频，不把训练框架 batch 写回 recorder schema。程序开发归 search/synthesis，混合方法可以显式引用模型与程序产物，不按梯度/搜索算法强制互斥。
