# Phase 3：撤离成功 + 简易支援 API（game-cli）

## Context

- Destroy 任务 0.1 垂直切片已完成：
  - PixelHD Destroy 任务通过 `pixelhd.mode = "destroy_0_1"` 激活；
  - 使用 `DestroyBlockObjective` 以 `rules.waveTeam` 第一个 core 为目标；
  - 摧毁目标后，`MissionRuntime.extractionUnlocked == true` 且 `phase == extraction_unlocked`，有集成测试覆盖。
- HUD 已能根据 `MissionRuntime.phase` 显示阶段与撤离提示，并在移动端显示 2–3 个支援按钮（目前只打 log）。

本 Phase 目标：
- 在 Destroy 任务基础上，打通最简版「撤离成功 → mission_success → GameOver」闭环；
- 定义并实现一个极简支援 API，让 HUD 的支援按钮有真实效果（哪怕暂时只是视觉/玩法原型）。

---

## Tasks

### 1. 提取撤离区域并实现撤离成功逻辑

1.1 提取/创建撤离 Objective

- 利用现有 `ExtractObjective`（见 `MapObjectives` 注册列表）或类似机制，为 Destroy 任务增加一个“撤离”目标：
  - 触发条件：Destroy 目标完成且 `MissionRuntime.extractionUnlocked == true`；
  - 撤离区域：
    - 简化方案：以玩家出生点或某个固定位置附近若干 tile 为撤离区；
    - 可通过 map tag 或 `Rules.tags` 传入一个坐标（后续再扩展）。
- 建议做法：
  - 在 `PixelHDMissionLoader.initPixelHDMission(...)` 中，Destroy 目标注入后，再构造一个挂在 Destroy 之后的 `ExtractObjective`：
    - 仅在 PixelHD 模式下追加；
    - 让玩家在 Extraction Phase 时进入指定区域并等待短时间视为成功。

1.2 撤离成功 → mission_success → GameOver

- 当 ExtractObjective 完成时：
  - 使用现有工具链更新 `Rules.objectiveFlags` / `MissionRuntime`：
    - 标记 `MissionRuntime.success == true`；
    - phase == `MissionPhase.mission_success`；
  - 触发 GameOver：
    - 在合适位置（例如监听 `WinEvent` / `GameOverEvent` 或直接在 PixelHD 逻辑里）调用：
      - `Events.fire(new GameOverEvent(rules.defaultTeam));`
    - 复用 `Control` 中现有 GameOver 处理逻辑（不要改其内部）。

> 要求：Destroy → extraction_unlocked → extraction 完成 → mission_success → GameOver 这条链路在单人本地可以稳定重放。


### 2. 定义并实现简易支援 API（PixelHDStratagems）

2.1 API 契约

- 新建类：`core/src/mindustry/game/pixelhd/PixelHDStratagems.java`
- 提供静态方法（初版只要 1–2 个即可）：

```java
public class PixelHDStratagems {
    public static void callSupport1(float worldX, float worldY) {
        // TODO: 最小可见效果，例如在 (worldX, worldY) 生成一波小范围炮击/爆炸或落地补给箱
    }

    public static void callSupport2(float worldX, float worldY) {
        // TODO: 另一种支援，占位即可
    }
}
```

- 初版实现可以非常简单：
  - 例如：在目标点附近生成几发弹丸 / 爆炸效果，或放置一个临时防御结构；
  - 不要求平衡，只要视觉上“看得见支援来了”。

2.2 HUD 调用约定

- 约定 HUD（content-cli）会在点击支援按钮时：
  - 使用当前玩家位置或准星位置作为 (worldX, worldY)；
  - 调用 `PixelHDStratagems.callSupport1(...)` / `callSupport2(...)`。
- 你只需保证：
  - 这些方法在 PixelHD 模式下调用时不会抛异常；
  - 在非 PixelHD 模式不会被调用（HUD 已经做了显式判定）。


### 3. 新增/扩展测试

3.1 撤离成功集成测试

- 在 `PixelHDMissionLoaderTest` 或新测试类中新增用例，大致流程：
  1. 像 Destroy 集成测试一样初始化 PixelHD Destroy 任务；
  2. 模拟 Destroy 目标被摧毁（已有用例逻辑可复用）；
  3. 手动推进到 Extraction Phase（确保有 ExtractObjective 或等效逻辑）；
  4. 模拟玩家到达撤离区域并满足条件（例如：调用 ExtractObjective 的完成逻辑或设置必要标志）；
  5. 调用 `rules.objectives.update()` / `syncMissionRuntime()` 并断言：
     - `MissionRuntime.success == true`；
     - phase == `mission_success`；
     - 如合适，验证 GameOverEvent 至少被触发一次（可通过事件监听或标志实现）。

3.2 支援 API 烟雾测试

- 为 `PixelHDStratagems` 添加一个简单测试：
  - 在初始化好的 world/state 上调用 `callSupport1(...)`：
    - 确认不会抛异常；
    - 如有简单统计（例如生成的弹丸数量），可做 1–2 个断言。


---

## Reporting

- 完成本 Phase 后，请在此目录下更新：
  - `game-cli/P3-extraction-and-stratagems.md`
- 报告内容：
  - Summary：撤离成功 + 支援 API 的最终行为；
  - Changed Files：类和路径列表；
  - Verification：自动测试名称 + 手动测试步骤；
  - Risks / TODO：你认为下一步最该完善的 1–2 个点。