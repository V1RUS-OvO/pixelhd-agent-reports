# Phase 3：PixelHD 测试地图 & 标签（maps-cli）

## Context
- Destroy / 撤离 / 支援 0.1 垂直切片的代码和 HUD 已基本就绪；
- 需要一组专用的 PixelHD 测试地图来做内测与回归：至少要覆盖 Destroy + 撤离 + 多人路径。

## Tasks

1. 创建至少 2 张 PixelHD 测试地图
- 地图建议：
  1. `PixelHD_Destroy_Test`：
     - 结构：
       - 玩家 core（defaultTeam）在相对安全的后方；
       - 敌方 core（waveTeam）在前线，路径清晰；
     - 用于验证：Destroy → 撤离完整流程；
  2. `PixelHD_Destroy_Hard`（或类似命名）：
     - 敌人路径更复杂，地形更危险，适合压测敌潮 + 撤离压力；

- 操作步骤（用 Mindustry 内置编辑器即可）：
  - 为每张新图设置清晰的 `name`（上面建议名）；
  - 布局玩家 core 和敌方 core；
  - 保存到本地自定义地图目录（默认在 Mindustry 的 maps 目录下）。

2. 为地图打 PixelHD 模式 tag

- 对每一张 PixelHD 测试地图设置 map tags（在编辑器或保存后用外部工具编辑）：
  - `pixelhd.mode = destroy_0_1`

- 目的：
  - 使 `PixelHDMissionLoader.isPixelHDMission(rules, map)` 返回 true；
  - 自动注入 Destroy + 撤离任务逻辑和 HUD。

3. 记录地图清单

- 文件：`C:/Users/KAKA/pixelhd-agent-reports/maps-cli/P3-maps-and-tags.md`
- 请写入：
  - 每张 PixelHD 测试地图的：
    - 名称（name）
    - 文件名（file）
    - 主要特征（地形 / 敌方 core 位置 / 推荐难度）
    - 已设置的 tags（至少包含 `pixelhd.mode`）

## Done 条件
- 至少 2 张带 `pixelhd.mode = destroy_0_1` 的地图可在游戏中选择；
- 报告文件中清楚列出这些地图的名字与特征，供 QA 和开发使用。

## Progress
- 2026-03-24：已确认身份为 `maps-cli`，已同步通讯仓库最新内容。
- 当前状态：已读取本阶段目标，待进入 `C:/Users/KAKA/pixelhd` 与地图资源流程，创建并登记测试地图。
