# AvZ fork 模块设计

Repo：`pvz-agent-lab/avz`

## 角色

AvZ fork 是唯一进入游戏进程、拥有游戏线程控制权的模块。它提供语义动作、hook 与可重复推进原版 PvZ 所需的确定性控制能力。

它的基本心智模型是：

```text
AvZ action / runtime request
        ↓
游戏线程边界
        ↓
确定性控制器
        ↓
原版 update / draw / native call
        ↓
S + receipts
```

## 原生 SDK 与在线 bridge

C++ 脚本 SDK、编译与注入能力由 AvZ fork 维护。原生脚本在游戏内执行，可以使用调度、条件和回调；预先编译不意味着策略是静态动作表。

在线 bridge 将受支持的观察、动作、推进、checkpoint 和脚本控制请求送到游戏线程，复用底层动作与调度实现。远程接口使用有版本的语义请求与结果，不直接导出进程内指针或 C++ 对象。进程外生命周期和 CLI/SDK 适配归共享控制后端。

原生脚本与外部动作的控制权由后端显式配置，Runtime 在稳定边界落实；脚本调度和回调状态属于恢复闭包审计对象。任意 C++ 全局状态、外部资源和未审计插件不能默认恢复，需返回支持范围或拒绝。

## 负责什么

- AvZ 语义动作和原生调用封装；
- 单动作提交的顺序、状态版本和不可交错执行，动作提交与 tick update 分离；
- 游戏线程、pre/update/post 边界和 single-step；
- RNG 实例、seed、游标、调用点、参数、返回值和前后状态；
- x87/SSE 浮点环境；
- GameClock、MjClock 等会影响模拟的时钟；
- 对象池、动画、粒子、音效和出怪路径的确定性控制或观测；
- 从稳定边界导出 `S` 所需的 native 状态；
- root 重建、checkpoint capture/restore、确定性 recompute 和 branch primitive；
- 将动作结果和 native receipts 发布给进程外的 session。

## 不负责什么

AvZ 不负责启动或清理外部进程，不负责 IPC client，不负责模型调用、数据集、reward、训练或视频编码。它也不因为提供 restore primitive 就自动声称支持真实进程 fork。

## 动作执行契约

基础 commit 请求包含一个 Action、预期 BoundaryRef 和 request_id，具体参数见 [session 单动作提交](05_session与执行契约.md#单动作提交契约)。AvZ 在受控暂停的稳定边界检查执行代次与状态版本，在游戏线程执行该动作，结束后返回实际前后边界和结果。执行期间不得插入另一个外部动作、checkpoint 操作或下一次 update；commit 本身不推进 tick。

同 tick 的 shovel→plant 与 plant→shovel 是两条有序执行路径，不得排序、合并或视为同一个动作集合。每次动作导致状态变化时更新状态版本；advance 才请求 tick update。动作内部原生调用的副作用、失败前可能完成的工作和终局行为必须逐个定义。

这里的原子性指调度上不可交错，不默认承诺事务回滚。部分执行或未知结果不得报告为完整成功，也不能仅因超时自动重复动作；无法确认稳定状态时阻止后续变更，直到实际状态被确认或重新建立。批量提交若将来需要，必须另行定义有序序列、逐项结果与失败停止规则，不属于当前基础接口。

## 输出

AvZ producer 输出三类事实：

1. `Observation`: 稳定边界上的 `S` 或构成 `S` 的 native 字段；
2. `ActionReceipt`: 动作是否进入游戏线程、是否执行、实际推进量和失败原因；
3. `NativeReceipt`: RNG、FP、clock、对象池、动画和其他已审计调用的前后证据。

session 负责传输这些事实，trajectory-recorder 负责记录这些事实。

## 分支模型

项目将实现两种 fork，主要原生实现均由 AvZ 承担：

- `native_restore`：捕获稳定边界 checkpoint，恢复后接受新的动作，用于高频搜索。目标是恢复成本不随历史长度 n 增长，即相对于 n 近似 O(1)。捕获、状态复制、资源重建和校验仍有成本，具体内存表示与优化方式由实现验证后选择。
- `recompute`：重建可复现 root，按记录的动作与推进边界执行到目标 frame，成本随前缀工作量 n 增长，近似 O(n)。AvZ 提供 root 重建和受控执行，在线分支控制或离线轨迹加载器编排历史重放；此路径长期保留用于复现、兼容和恢复行为的对照。

同进程串行恢复可以实现逻辑 fork，不代表多个分支同时运行。跨进程 checkpoint、独立进程快照或 copy-on-write 是需单独声明的能力，不由上述复杂度目标自动推出。

## 恢复契约

契约从未来执行依赖推导，不从 `S` 字段列表直接推导。设完整运行时状态为 X，C = capture(X)，X' = restore(C)：在声明的版本、边界和动作范围内，从 X 和 X' 执行相同后续动作，应得到相同的逐 tick 语义状态、动作结果、终局结果和约定参与比较的 receipts。当前帧 `S` 相等只是必要检查。

AvZ 维护执行依赖清单：从 commit/advance、原版 update 及调度入口追踪读取的可变状态，对每项明确保存恢复、固定校验、确定性重建、隔离控制或有依据的排除。清单应审计游戏对象及对象池、RNG、clock、FP 环境、动画等运行时依赖，以及 AvZ 调度队列、动作游标和控制器状态；具体范围以审计结果为准。

checkpoint 契约需要明确：

- 允许 capture/restore 的稳定边界；没有执行到一半的动作或 native 调用，待处理请求和 producer 事件有明确处置；
- checkpoint 身份、父 frame、格式版本及游戏/AvZ/资源兼容性；
- 保存状态和重建规则，以及指针、对象身份和外部资源的处理；
- 所属 session、有效期、释放方式、可重复恢复性，以及是否支持跨进程或持久化；
- 恢复后的实际状态确认、执行代次和旧句柄失效规则；失败或部分恢复时不得继续暴露为可执行分支。

首个快速恢复范围可限定为固定版本、同 session、稳定 tick 边界和串行探索；具体支持范围必须显式返回。checkpoint 的原生 payload 由 AvZ 定义，不构成第二套公共 `S` schema，native receipts 也不自动构成恢复材料。

root 同样需要重建契约：记录可重放的初始化 recipe 或受支持的 checkpoint 及其兼容性，不能仅凭 root_id 声称可重建。只在原 session 有效的 handle 不提供冷启动复现能力。

契约可信度来自依赖审计和差分验证：比较直接执行与恢复后执行的相同后缀，并检查 A→恢复→B 与干净起点 B 的一致性、交换分支顺序、重复和嵌套恢复。按首个分歧回溯遗漏依赖；详细测试与性能验收在 issue 中维护。性能分别报告 capture、restore、初始化、校验、重放耗时和内存占用，不能把全历史读取或校验隐藏在 O(1) 指标外而宣称端到端常数成本。

策略与底层实现的差异不改变 `S` 和 receipts 的公共格式。若 restore 没有覆盖影响约定未来结果的状态，AvZ 不得把它包装成可执行分支。

## checkpoint 表示与成本实验

live checkpoint 是供 native_restore 使用的完整恢复材料及其有效句柄，不能仅保存指向仍会变化的活动状态的引用。AvZ 需要保证后续执行不污染它；具体采用复制、共享且隔离修改的状态，还是其他表示，待实验确定。持久化 artifact 的 export/import 是另外的能力，不由同进程 capture/restore 自动推出。

先在固定版本与稳定边界上验证同 session capture/restore，再判断持久化导出/导入是否值得实现，以及是否需要独立格式。实验比较同一目标边界的 root 重算、首次重算后 capture、重复 live restore；在导入/导出受支持时再加入持久化加载路径。

各路径分别记录冷启动、历史读取/校验、实际推进、capture、restore、export/import、结果校验和内存占用，并改变历史长度、状态规模、分支次数与缓存预算。每项性能结果都要伴随相同后续动作的逐 tick 一致性证据，检查分支交叉执行是否污染状态。这里列出待验证问题，不声称实验已完成；具体 testbench 与关闭条件在 issue 中维护。

AvZ 提供原语测量，上层提供读取与计划编排的端到端测量，避免只报 restore 耗时而遗漏首次生成 checkpoint 的重算成本。时间轴推进与表示变换的路径见 [loader 转换关系](../05_轨迹加载与执行支路_trajectory-loader.md#live-checkpointtraj-与转换路径)。
