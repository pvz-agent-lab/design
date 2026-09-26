# pvz-agent-lab 文档草稿

状态：设计草稿与验收草稿分离。

这组文档把 issue 102 的迁移拆成三类内容：

- 设计文档：说明每个 repo 的职责、心智模型、接口方向和数据流；
- 迁移文档：说明先后顺序、旧实现如何退出和跨仓依赖如何落地；
- 验收草稿：面向 issue 的测试台、负例、真机实验和关闭条件。

设计文档不把测试门槛写成职责，不把“尚未通过”写成设计概念。验收草稿也不反过来定义模块内部实现。

## 设计

- [总览](overview.md)
- [AvZ fork](avz-design.md)
- [pvz-session](pvz-session-design.md)
- [trajectory-recorder](trajectory-recorder-design.md)
- [trajectory-loader](trajectory-loader-design.md)
- [video-recorder](video-recorder-design.md)
- [agent-loop](agent-loop-design.md)
- [rollout-utils](rollout-utils-design.md)
- [train](train-design.md)

## 迁移

- [迁移顺序与仓库变更](migration-plan.md)

## 验收草稿

- [issue 102 验收草稿](acceptance-draft.md)
