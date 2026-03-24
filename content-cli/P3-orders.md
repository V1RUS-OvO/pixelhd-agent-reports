# Phase 3：HUD 联动 Destroy/撤离 + 接线支援 API（content-cli）

## Context

- game-cli 已实现 PixelHD Destroy 任务垂直切片：
  - Destroy 目标完成后，`MissionRuntime.extractionUnlocked == true` 且 phase == `extraction_unlocked`；
  - 后续会实现 Extraction 成功 → `mission_success` 的逻辑。
- 你已经在 `HudFragment` 中挂出：
  - PixelHD 顶部任务栏（阶段标签 + `currentObjective` + 撤离提示占位）；
  - 移动端右上支援按钮骨架（当前仅打印日志）。

本 Phase 目标：
- 让 HUD 更紧密地反映 Destroy / Extraction 状态（包括简单的撤离倒计时占位）；
- 把支援按钮从“打 log”升级为调用 game-cli 提供的 `PixelHDStratagems` API。

---

## Tasks

### 1. 优化 PixelHD 模式判定（如尚未完全改成统一逻辑，请确认）

- 确保 `HudFragment` 中的 PixelHD HUD 显示条件统一使用：

```java
boolean pixelhd = state.rules != null && state.map != null
    && PixelHDMissionLoader.isPixelHDMission(state.rules, state.map);

boolean pixelhdActive = pixelhd
    && state.rules.missionRuntime != null
    && state.rules.missionRuntime.phase != Rules.MissionPhase.idle;
```

- 顶部任务栏与支援按钮的 `visible` 条件都基于 `pixelhdActive`（支援按钮还需 `mobile`）。

### 2. 撤离阶段 HUD 联动（含倒计时占位）

2.1 撤离提示细化

- 在顶部任务栏中，针对以下 phase 增强显示：
  - `MissionPhase.extraction_unlocked`
  - `MissionPhase.extraction_called`
  - `MissionPhase.mission_success`

- 具体表现：
  - 在 `pixelhdExtractionLabel` 内：
    - `extraction_unlocked`：显示“撤离已解锁，前往撤离点！”
    - `extraction_called`：显示“撤离进行中，坚守至登舰！”
    - `mission_success`：显示“撤离成功！”

2.2 倒计时占位

- 与 game-cli 协商的最小约定（先占位）：
  - 若 `MissionRuntime` 或其他结构中暂时没有标准字段，可先：
    - 在 HUD 内维护一个简单的 `float extractionTimeLeft` 字段，
      - 当 phase 变为 `extraction_unlocked` 时初始化为一个常量（例如 20s），每帧递减；
      - 当 phase 进入 `mission_success` 或 `mission_failed` 时停止显示；
    - 文本显示类似：“撤离倒计时：{ceil(extractionTimeLeft)}s”。
  - 后续当 game-cli 提供真实倒计时时，再切换为读取真实字段。

> 这一段的目标是：在 Extraction 段，玩家明显感觉“有时间压力”，哪怕暂时是本地 HUD 自己算的时间，不是真实服务器逻辑。


### 3. 把支援按钮连到 PixelHDStratagems

- 约定 game-cli 会提供类：`mindustry.game.pixelhd.PixelHDStratagems`，包含静态方法：

```java
public class PixelHDStratagems {
    public static void callSupport1(float worldX, float worldY) { ... }
    public static void callSupport2(float worldX, float worldY) { ... }
}
```

- 请在 `HudFragment` 中：
  - 为三个支援按钮分别绑定：
    - 支援一 → `PixelHDStratagems.callSupport1(targetX, targetY)`
    - 支援二 → `PixelHDStratagems.callSupport2(targetX, targetY)`
    - 支援三 → 当前可以继续打 log 或复用其中一种支援。
  - `targetX/targetY`：
    - 简化方案：直接使用玩家当前世界坐标（`player.x / player.y`）；
    - 后续可以改为“准星位置”或“点击位置”。

- 注意：
  - 仅在 `pixelhdActive && mobile` 时显示并允许点击；
  - 调用前做好 `state.player` / `player.unit()` 的 null 检查，避免在玩家死亡/未加载时 NPE。


---

## Reporting

- 完成本 Phase 后，请在：
  - `content-cli/P3-hud-and-stratagems.md`
  写入：
  - Summary：
    - 撤离阶段 HUD 实际表现；
    - 支援按钮目前触发的具体效果（从玩家视角描述）。
  - Changed Files：
    - 修改过的 HUD 类与方法。
  - Verification：
    - Desktop / Android 实际点击支援按钮、观察效果和日志的步骤；
  - Risks / TODO：
    - 你认为下轮最值得优先解决的 UI 问题（信息拥挤 / 误触等）。