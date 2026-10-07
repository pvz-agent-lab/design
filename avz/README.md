# AvZ Runtime 与在线控制工具

状态：设计草稿，回应 [issue #6](https://github.com/pvz-agent-lab/design/issues/6)。这里的目录表达职责边界，不承诺同名仓库、进程或现成功能。

AvZ 原有 C++ 脚本 SDK、编译/注入能力与 Runtime 继续放在 AvZ fork。新增在线控制工具提供 CLI 和可编程 SDK，两者连接共同的控制后端。人、LLM、模型 runner、在线搜索器与综合器都是调用方。

## 阅读顺序

1. [Runtime、原生脚本 SDK 与恢复契约](01_Runtime与原生脚本SDK.md)
2. [在线 CLI](02_在线CLI.md)
3. [在线 SDK 与 REPL](03_在线SDK与REPL.md)
4. [CLI、SDK 共享控制后端](04_共享控制后端.md)
5. [后端内部 session 与单动作契约](05_session与执行契约.md)

```text
人工 / LLM / 模型 runner / 在线搜索 / 程序综合 / 在线采样
                          │
                 CLI 或可编程 SDK
                          │
                    共享控制后端
          session、branch/checkout、checkpoint、任务
             │            │              │
             │            ├── trajectory-loader（持久化材料）
             │            ├── trajectory-recorder（执行事实）
             │            └── video-recorder（可选）
             ▼
       游戏内 AvZ bridge / Runtime
           原生 C++ SDK、调度、动作、确定性与恢复
                          │
                         PvZ
```

图中的 recorder 接收事件，loader 解析材料并生成受校验的执行计划，video-recorder 接收画面与时间轴。三者是后端集成的独立职责组件，不要求同进程、同实例或全部启用。公共记录 schema 仍由 trajectory-recorder 拥有。

## 原生脚本与在线控制

原有脚本路径是预先编写 C++、编译、装载/注入后在游戏内执行，脚本仍可包含动态逻辑。在线路径让外部调用者持续观察、执行动作、推进和 checkout，不要求每次生成并编译 C++。

CLI 执行原生脚本可以调用已有工具链，不要求内置 C++ 解释器；Python REPL 可以在外部通过 SDK 操作游戏，不要求游戏进程内嵌 Python。在线接口复用底层动作实现，但不直接远程暴露 C++ 指针、回调或进程内对象。

## 部署与命名

CLI 是短命令入口；SDK 可维持连接；后端在客户端命令结束后继续持有会话和任务。CLI、SDK、后端可以属于同一项目，后端也可以嵌入单个使用者进程；若多个客户端要操作同一 session，必须连接同一个权威控制实例。

`pvz-session` 在本设计中成为后端内部的宿主与 IPC 层。是否保留独立包/仓库由实现确定。现有 `agent-loop` 收缩为可选 runner，通用分支管理迁到后端。工具产品名、通信传输、SDK 语言和拆仓方案待定；Python 是首选交互示例，不是已发布接口。
