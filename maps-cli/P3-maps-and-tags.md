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

## 2026-03-24 / director-commands-20260324-02 / 20260324-ALL

### 本次完成
- 已在 `C:/Users/KAKA/pixelhd` 新增地图生成器与任务，基于内置地图产出 2 张 PixelHD 测试地图，并同步安装到本机 Mindustry 自定义地图目录。
- 已生成并验证以下地图均带有 `pixelhd.mode = destroy_0_1`：
  - `PixelHD_Destroy_Test`
    - file: `tests/src/test/resources/pixelhd/maps/pixelhd_destroy_test.msav`
    - source: `serpulo/extractionOutpost`
    - 特征：清晰主路径、敌方 core 位于前线、适合走通 Destroy -> Extraction 冒烟流程
    - tags: `pixelhd.mode=destroy_0_1`, `pixelhd.profile=clear lane, enemy core forward, low pressure`, `pixelhd.source=serpulo/extractionOutpost`
  - `PixelHD_Destroy_Hard`
    - file: `tests/src/test/resources/pixelhd/maps/pixelhd_destroy_hard.msav`
    - source: `serpulo/planetaryTerminal`
    - 特征：路径更长、地形更复杂、敌压更高，适合多人/高压回归
    - tags: `pixelhd.mode=destroy_0_1`, `pixelhd.profile=longer approach, harder terrain, higher pressure`, `pixelhd.source=serpulo/planetaryTerminal`

### 代码与资源改动
- `C:/Users/KAKA/pixelhd/tools/src/mindustry/tools/PixelHDMapPackGenerator.java`
- `C:/Users/KAKA/pixelhd/tools/build.gradle`
- `C:/Users/KAKA/pixelhd/tests/src/test/resources/pixelhd/maps/pixelhd_destroy_test.msav`
- `C:/Users/KAKA/pixelhd/tests/src/test/resources/pixelhd/maps/pixelhd_destroy_hard.msav`
- 本机安装副本：`C:/Users/KAKA/AppData/Roaming/Mindustry/maps/pixelhd_destroy_test.msav`
- 本机安装副本：`C:/Users/KAKA/AppData/Roaming/Mindustry/maps/pixelhd_destroy_hard.msav`

### 复现 / 验证
- 在源码仓库执行：`./gradlew :tools:generatePixelHDMaps`
- 生成任务会：
  - 从内置地图载入源图；
  - 重写 `name/description/pixelhd.*` tags；
  - 输出到 `tests/src/test/resources/pixelhd/maps/`；
  - 同步复制到 `C:/Users/KAKA/AppData/Roaming/Mindustry/maps/`
- 本轮实际验证结果：`./gradlew :tools:generatePixelHDMaps` 运行成功，生成目录与本机地图目录均已出现 2 个 `.msav` 文件。

### 结果 / TODO
- 结果：maps-cli 已提供 2 张可供 game-cli / qa-cli 使用的 PixelHD destroy_0_1 测试地图。
- 代码仓库本地提交：`e37a69e040` `feat: 添加 PixelHD 测试地图生成器`
- TODO：待 game-cli 的 Destroy -> Extraction 闭环进一步稳定后，用这两张地图补一次桌面端实际进图验证与截图记录。

## 2026-03-24 / director-commands-20260324-03

### 本次执行
- 已再次同步通讯仓库并读取 `AGENT_COMMS_PROTOCOL.md`、`director-commands-20260324-03.md`。
- 已执行 Desktop 回归准备脚本：`bash "tools/pixelhd-desktop-regression.sh" "PixelHD_Destroy_Test"`
- 已执行 PixelHD 测试命令：`./gradlew :tests:test --tests "mindustry.game.pixelhd.*"`
- 已执行 Desktop 开发构建启动：`./gradlew pixelhdDesktopDev`

### 本轮验证结果
- `pixelhdDesktopDev` 可成功编译并启动桌面客户端；30s 冒烟窗口内完成 SDL/OpenGL/音频初始化并进入主程序加载阶段，说明本机可加载本轮地图包进行后续人工回归。
- `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"` 当前失败，阻塞点不在地图资源本身，而在现有测试源码编译：
  - `tests/src/test/java/mindustry/game/pixelhd/PixelHDMissionLoaderTest.java`
  - `tests/src/test/java/mindustry/game/pixelhd/PixelHDStratagemsTest.java`
  - 典型报错：`arc.util.Events` 找不到、`PixelHDMissionLoader` / `PixelHDStratagems` 找不到、`Map.copy()` 不存在。
- 当前 CLI 环境可完成“桌面启动冒烟”，但无法在无交互图形控制下直接完成一整局手动 Destroy -> Extraction 成功/失败流程；该步骤需要 qa-cli / 人工桌面操作继续完成，并以本节命令作为起点。

### 地图建议补充
- `PixelHD_Destroy_Test`
  - 推荐玩家数：`1-2`
  - 适配判断：适合单人或双人跑通 Destroy -> Extraction 成功路径，适合作为 game-cli / content-cli 的基础验证图。
  - 风险与建议：
    - 给 `game-cli`：优先用此图验证 extraction_unlocked / mission_success / mission_failed 的状态切换；地形简单，便于排除逻辑噪声。
    - 给 `content-cli`：HUD 文案与撤离区域提示在此图最容易观察，建议先用它检查阶段切换与按钮遮挡。
    - 给 `qa-cli`：适合作为首张回归图，先跑成功路径，再人为拖延/弃防测试失败路径。
- `PixelHD_Destroy_Hard`
  - 推荐玩家数：`2-4`
  - 适配判断：更适合多人或高压场景，适合验证更长推进与更复杂撤离路径下的 UI 可读性与失败链路。
  - 风险与建议：
    - 给 `game-cli`：重点观察 defaultTeam 核心承压时 mission_failed 触发是否过早/过晚。
    - 给 `content-cli`：路径更长时应确认 Extraction 提示足够醒目，避免玩家找不到撤离点。
    - 给 `qa-cli`：适合复现多人/手机端路径拥堵、视野压迫、撤离提示不明显等问题。

### 当前问题
- 由于测试命令未通过，maps-cli 无法把“地图可用性”与“PixelHD 自动化逻辑”一起判定为绿；当前地图资源已就位，但需等待 game-cli 修复相关 PixelHD 测试编译问题。
- 由于 CLI 会话不具备图形交互控制，本轮只能确认 Desktop 可启动到主程序，不能在本会话内独立完成整局手打记录。

### 下一步
- 等 `mindustry.game.pixelhd.*` 测试恢复后，按同样命令重新跑一遍，并与 qa-cli 对齐两张图的成功/失败实际结果。
