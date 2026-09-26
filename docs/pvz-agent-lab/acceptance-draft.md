# issue 102 验收草稿

状态：验收侧草稿。本文只准备 issue 的测试台、负例和关闭条件，不定义各 repo 的内部设计。

## 验收对象

第一阶段的端到端链路是：

```text
avz + pvz-session + agent-loop + trajectory-recorder
```

trajectory-loader 负责支路重建验收；video-recorder、rollout-utils 和 train 分别有独立验收，不混入经典十二炮的环境确定性门槛。

## 测试台分层

### 纯离线测试台

- trajectory-recorder schema、哈希、seal、tree、旧格式适配；
- 损坏字节、错误摘要、缺父节点、跨 root 链接、坏 schema、路径越界；
- trajectory-loader 缺 root、版本不匹配、seal 失败、动作边界断裂和目标 frame 不可达；
- rollout-utils 的筛选、去重、切分、转换和数据集 manifest；
- train 的配置解析、样本适配和 checkpoint identity。

### 无游戏集成夹具

- pvz-session 的进程/IPC/句柄所有权；
- 正常 close、超时、取消、断线、初始化失败、部分执行和清理失败；
- agent-loop 的 request ID、过期版本、结果未知和禁止盲目重试；
- trajectory-recorder producer 事件的顺序和重复拒绝；
- video-recorder 输入断流和时间轴负例。

### 真实游戏测试台

- 固定游戏、AvZ、资源、构建和执行模式的独立启动；
- observe/commit/advance 的实际执行量和 native receipts；
- B(0) root、每 tick `S`、RNG/FP/clock 证据和动作边界；
- trajectory-loader 从 root/frame 重放到目标 frame；
- control、control-rerun、intervention、intervention-rerun 四条轨迹；
- 支路加载后同一 action 得到预期的后继 `S`；
- 进程正常关闭、失败关闭和自有资源清理。

## 经典四轨迹

四条真实轨迹的执行者是 agent-loop，游戏控制由 AvZ/session 完成，事实由 trajectory-recorder 记录：

```text
control-a
control-b
intervention-a
intervention-b
```

验证重点是 root、动作边界、每 tick 状态、RNG/native receipts、分支父子关系、rerun 一致性和真实终点。失败轨迹必须保留，不能以兄弟轨迹成功替代。

## trajectory-loader 支路验收

- 从封存 root/frame 建立新 pvz-session；
- 验证实际到达的 frame identity 和 `S` 摘要；
- 重放动作与原动作边界逐项对齐；
- loader 返回的执行句柄可以继续提交新动作；
- restore/recompute 失败时有明确失败回执；
- 不能通过直接写入几个字段伪造加载成功。

## video-recorder 验收

- 输出视频/demo 与 trajectory/frame identity 可关联；
- 画面/音频断流、时间轴异常和关闭失败可定位；
- 视频数据不能被 trajectory-recorder 读取器当成 `S` 或 transition；
- 视频采集不会默默改变被声明为严格确定性的执行模式。

## 关闭条件草稿

- 每个 repo 有独立安装入口、公共 schema/API 和负例测试；
- 真实游戏链路有绑定版本组合和可复查的运行产物；
- 同根四轨迹和至少一个支路加载实验完整封存；
- trajectory-recorder 可以在无游戏环境读取全部封存记录；
- trajectory-loader 的 backend 明确标记为 recompute、native_restore 或 process_branch；
- video-recorder、rollout-utils、train 的产物不冒充环境确定性证据；
- 旧证据不可变，新入口和旧入口的兼容/退出路径明确。

这里的“通过”必须对应具体测试台和产物；未执行的测试保留为未执行，失败保留为失败，不把状态名称当作替代证据。
