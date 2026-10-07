# video-recorder 模块设计

Repo：`pvz-agent-lab/video-recorder`

## 角色

video-recorder 负责 demo、视频、音频和时间轴产物。它服务展示、复盘和人工检查，不是 trajectory-recorder 的别名。

## 输入

- 共享控制后端的 session 层或 AvZ Runtime 提供的画面/绘制回执；
- 可选的音频回执；
- session version、tick、trajectory_id、frame_id 等时间轴元数据。

## 输出

- 视频文件或 demo 包；
- 帧时间轴和展示元数据；
- 可选地引用 trajectory-recorder 的 trajectory/frame identity。

视频文件不是 `S`，不能作为 root、transition 或 branch 的唯一证据。video-recorder 不负责动作、状态恢复、确定性控制、训练数据格式或搜索分支。

## 工具集成与生命周期

CLI/SDK 可通过共享后端启动、查询和停止视频任务。视频服务可独立部署，录制与搜索不要求始终启用视频。时间轴需区分 session、执行代次和分支 checkout，避免相同 tick 的不同分支帧混淆；视频录制状态不自动随游戏 checkpoint 回滚。
