# P4 HUD / Stratagem Pass - 2026-03-24

> 已执行 `director-commands-20260324-02`，本次改动属于 `content-cli` 的第 3 条任务：完善 HUD 阶段文案 + Stratagem 可见行为。

## 做了什么

- 直接在 `C:/Users/KAKA/pixelhd` 落地了 PixelHD HUD / Stratagem 代码，而不是只写同步文档。
- 修正 `HudFragment` 的 PixelHD HUD 阶段文案：
  - `mission_started` -> `pixelhd.phase.landing`
  - `objective_active` -> `pixelhd.phase.objective`
  - `extraction_unlocked` -> `pixelhd.phase.extraction_unlocked`
  - `extraction_called` -> `pixelhd.phase.extraction_called`
  - `mission_success` -> `pixelhd.phase.success`
  - `mission_failed` -> `pixelhd.phase.fail`
- 修正撤离倒计时更新方式：从 `build()` 时只跑一次，改为 `parent.update(...)` 每帧更新，本地占位倒计时终于会真正递减。
- 将移动端 3 个按钮改为实际文案 key：
  - `pixelhd.stratagem.support1`
  - `pixelhd.stratagem.support2`
  - `pixelhd.stratagem.support3`
- 扩展 `PixelHDStratagems`：
  - Support 1：范围爆炸 + 伤害；
  - Support 2：优先在玩家附近放一个临时 `Blocks.duo` 炮塔，放不下时回退生成临时 `UnitTypes.fortress` 守卫；
  - Support 3：对附近友军单位/建筑做一次治疗与补给脉冲。
- 更新 `docs/design/ui-hud-0_1.md`，补充实际 key、当前占位逻辑与支援按钮行为说明。

## 改了哪些文件

- `core/src/mindustry/ui/fragments/HudFragment.java`
- `core/src/mindustry/game/pixelhd/PixelHDStratagems.java`
- `core/assets/bundles/bundle.properties`
- `core/assets/bundles/bundle_zh_CN.properties`
- `docs/design/ui-hud-0_1.md`

## 如何复现 / 验证

1. 编译验证：
   - `cd C:/Users/KAKA/pixelhd`
   - `./gradlew core:compileJava`
2. 桌面运行后加载带 `pixelhd.mode = "destroy_0_1"` 的地图，观察顶部 HUD：
   - Landing / Objective / Extraction Ready / Extraction Active / Success / Fail 文案会按 `MissionPhase` 切换；
   - 进入 `extraction_unlocked` / `extraction_called` 后，倒计时会持续递减，而不是停在初始值。
3. 移动端或 `Vars.mobile == true` 环境下：
   - 右上角出现 3 个支援按钮；
   - 点击 Support 1 可见爆炸；
   - 点击 Support 2 可见临时炮塔或守卫单位；
   - 点击 Support 3 可见治疗/补给波效果。

## 结果 / TODO

- 结果：`./gradlew core:compileJava` 已通过，HUD/Stratagem 代码完成一轮可编译落地。
- 结果：本轮实现仍以单机/房主占位行为为主，尤其 Support 2 的临时炮塔放置还需要 game/net 侧后续确认联机权限与同步边界。
- TODO：等 game-cli 提供真实撤离时间字段后，把 HUD 的本地 20s 占位倒计时替换成统一 runtime 时间源。
- TODO：后续若手机端误触明显，需要再收缩按钮尺寸或改为二段式落点交互。
