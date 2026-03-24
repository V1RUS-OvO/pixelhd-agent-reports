# UI / HUD 0.1 设计与骨架实现报告

## Summary

本次任务围绕 0.1 版本的任务 UI/HUD 和支援调用界面，完成了两部分工作：

1. 在设计层面，产出 `ui-hud-0_1.md`，明确了任务选择界面、战斗 HUD、Stratagem 调用 UI、撤离阶段 HUD 的结构与资产需求，并标出 0.2+ 才需要的扩展 UI。
2. 在实现层面，在 `HudFragment` 中挂出了 PixelHD 专用的 0.1 HUD 骨架：顶部任务栏（阶段 + 当前目标 + 撤离提示）以及移动端右侧 2–3 个支援按钮，且仅在 PixelHD 任务模式下生效，不影响原版 Mindustry UI。

## Changed Files

1. 设计文档：
   - `docs/design/ui-hud-0_1.md`
     - 描述了：
       - 任务选择界面（基地/母舰）的信息结构和简单 ASCII 草图；
       - 战斗中 HUD 的整体布局（阶段/目标、Capture 进度、友伤提示位置）；
       - 手机端 Stratagem 调用方式（按钮组 / 轮盘的简化方案）；
       - 撤离阶段 HUD（倒计时 + 撤离区域指引）；
       - 0.1 必需的 UI/美术/音效资产清单；
       - 明确标记了战线/星图、高级战报、复杂 Stratagem 输入、职业构筑等为 0.2+ 才做的 UI。

2. HUD 代码骨架：
   - `core/src/mindustry/ui/fragments/HudFragment.java`
     - 新增 PixelHD 任务相关字段：
       - `pixelhdMissionTable`, `pixelhdPhaseLabel`, `pixelhdObjectiveLabel`, `pixelhdExtractionLabel`, `pixelhdStratagemTable`。
     - 导入 PixelHD 判定工具：
       - `import mindustry.game.pixelhd.PixelHDMissionLoader;`
     - 新增 PixelHD 任务模式判定方法：
       - `isPixelHDMission()`：通过 `PixelHDMissionLoader.isPixelHDMission(state.rules, state.map)` 判断当前是否为 PixelHD 任务。
       - `isPixelHDMissionActive()`：在 HUD 显示层进一步要求 `shown == true` 且 `missionRuntime.phase != idle`。
     - 新增阶段映射方法：
       - `mapPixelhdPhase(Rules.MissionPhase phase)`：将 `MissionPhase` 映射为短标签（落地 / 任务 / 坚守 / 撤离 / 成功 / 失败），idle 返回空字符串。
     - 新增 PixelHD 顶部任务栏 HUD：
       - 通过 `parent.fill(t -> { t.name = "pixelhd-mission"; t.top().center(); ... })` 挂在顶部居中区域；
       - 显示条件：`t.visible(() -> isPixelHDMissionActive());`；
       - 内容：
         - `pixelhdPhaseLabel`：根据 `missionRuntime.phase` 显示阶段标签；
         - `pixelhdObjectiveLabel`：显示 `missionRuntime.currentObjective` 文本（Destroy/Capture 等由 mission 系统填充）；
         - `pixelhdExtractionLabel`：当 phase 进入 `extraction_unlocked / extraction_called / mission_success` 时显示撤离提示文案（暂不含真实倒计时）。
     - 新增 PixelHD 移动端支援按钮区：
       - 通过 `parent.fill(t -> { t.name = "pixelhd-stratagem-buttons"; t.top().right(); ... })` 挂在右上区域；
       - 显示条件：`mobile && isPixelHDMissionActive()`；
       - 按钮骨架：
         - 三个纵向按钮："支援一" / "支援二" / "支援三"；
         - 点击行为暂时仅 `Log.info("PixelHD Stratagem X pressed")`，后续由 game-cli 接线实际 Stratagem 调用。

## Verification

> 说明：当前阶段以“0.1 HUD 骨架挂出”为目标，验证重点在于**构建通过**和**在 PixelHD 任务模式下 HUD 条件显示正确**，不要求 Stratagem 逻辑已接线完成。

1. 构建验证（桌面 + Android）

```bash
cd C:/Users/KAKA/pixelhd

# 桌面开发任务（如 infra 已配置）
./gradlew pixelhdDesktopDev

# Android 开发任务
./gradlew pixelhdAndroidDev
```

预期结果：
- 两个任务均构建成功，无编译错误；
- 尤其是 HudFragment 中新增对 `PixelHDMissionLoader` 的引用能正常解析（证明 pixelhd 核心代码与 HUD 代码一致）。

2. HUD 显示逻辑验证（桌面环境）：

> 需依赖 game-cli 提供的 PixelHD 任务地图 / 启动方式，此处只列出验证步骤骨架。

- 步骤：
  1. 启动桌面版 PixelHD（通过 `pixelhdDesktopDev` 或对应 run 配置）。
  2. 选择一张已标记为 PixelHD 任务模式的测试地图（满足 `PixelHDMissionLoader.isPixelHDMission(...) == true`）。
  3. 进入任务后，确保 `Rules.missionRuntime.phase` 在 Destroy/Capture 流程中从 `mission_started → objective_active → extraction_unlocked` 发生变化。

- 预期：
  - 当 phase 为 `idle` 时：顶部 PixelHD 任务栏和支援按钮均不显示；
  - 当 phase 进入 `mission_started / objective_active` 时：
    - 顶部任务栏出现：左侧为短标签（落地/任务），右侧为当前目标文案（Destroy/Capture 文本由 mission 系统提供）；
    - 右上角在移动端（或模拟 mobile 条件）出现“支援一/二/三”按钮，点击时在日志中看到对应 `PixelHD Stratagem X pressed` 输出；
  - 当 phase 进入 `extraction_unlocked / extraction_called` 时：
    - 顶部任务栏第三行显示撤离提示文案（当前为固定文案）；
    - 整体 UI 不遮挡主战斗区域中心视野，仅占顶部和右上边缘。

3. 非 PixelHD 模式回归验证：

- 在原版 Mindustry 模式 / 普通地图下运行：
  - `PixelHDMissionLoader.isPixelHDMission(...)` 返回 false；
  - 顶部 PixelHD 任务栏和右侧支援按钮区域均保持隐藏；
  - 原有 HUD（波次条、核心信息、移动端菜单按钮等）行为不变。

## Risks / TODO

1. Stratagem 按钮交互仍为占位
   - 目前按钮仅输出日志，尚未与实际 Stratagem 调用/落点选择逻辑对接；
   - TODO：
     - 由 game-cli 在合适位置接入实际 Stratagem 触发点（如打开落点选择模式、调用 mission/支援系统 API）；
     - 根据手机端手感选择“独立按钮组”与“轮盘”中的一种方案，为另一种保留 0.2+ 选项。

2. 顶部信息密度与原有波次 HUD 的叠加
   - PixelHD 模式下，顶部同时存在：
     - 原有波次/敌人数/计时状态条；
     - 新增的 PixelHD 任务阶段+目标+撤离提示栏。
   - 风险：在小屏移动端信息稍显拥挤，玩家可能对“最关键需要关注的信息”不够聚焦。
   - TODO：
     - 在 PixelHD 模式下适度弱化波次文本（更多使用图标 / 简略文案），或在 Extraction 阶段暂时提升任务栏权重要见度；
     - 根据后续 playtest 反馈迭代布局（尤其是 portrait 模式）。

3. 支援按钮与其他移动端控件的潜在遮挡
   - 当前支援按钮垂直排布在右上区域，在部分分辨率/布局下可能与 minimap 或其它 HUD 元素局部重叠；
   - TODO：
     - 在后续移动端整体 HUD 调优时，针对不同分辨率/比例做布局测试；
     - 若有需要，可在竖屏模式降为 2 个直出按钮 + 1 个合并入口，减少纵向高度占用。

4. 撤离倒计时目前为逻辑占位
   - 顶部撤离提示仅显示“撤离已解锁，赶往撤离点！”文本，未挂接真实剩余时间；
   - TODO：
     - 由 mission 系统在 `MissionRuntime` 或相关结构中提供标准化倒计时字段；
     - 在 HUD 中以统一格式显示（如 "撤离倒计时：10s"），并根据时间渐近增加可视紧张感（闪烁、颜色变化等）。
