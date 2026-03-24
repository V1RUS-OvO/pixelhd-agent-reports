# P4 Execution Pass - 2026-03-24-03

> 已执行 `director-commands-20260324-03`，本次工作属于 `content-cli`：整理并提交 HUD / Stratagem 相关改动，并确保 `core:compileJava` 与 `mindustry.game.pixelhd.*` 测试可跑。

## 做了什么

- 再次同步并读取最新协议 / 调度令。
- 在 `C:/Users/KAKA/pixelhd` 重新验证了：
  - `./gradlew core:compileJava`
  - `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"`
- 修复了 PixelHD 定向测试本身的编译/筛选问题，使其能真正匹配 `mindustry.game.pixelhd.*`：
  - 为测试类补上包名 `mindustry.game.pixelhd`；
  - 新增 `PixelHDTestSupport`，通过反射复用 `ApplicationTests` 启动测试环境；
  - 修复 `Events` 导入、缺失的 PixelHD 类导入、`Map.copy()` 不存在的问题；
  - 将 Destroy/Extract 测试切到带双核心的 `serpulo/extractionOutpost`，避免 `groundZero` 缺少敌方 core；
  - 初始化测试 bundle，避免 `MapObjectives` 文案格式化时空指针。
- 将测试修复提交到了代码仓库。

## 改了哪些文件

- `C:/Users/KAKA/pixelhd/tests/src/test/java/mindustry/game/pixelhd/PixelHDMissionLoaderTest.java`
- `C:/Users/KAKA/pixelhd/tests/src/test/java/mindustry/game/pixelhd/PixelHDStratagemsTest.java`
- `C:/Users/KAKA/pixelhd/tests/src/test/java/mindustry/game/pixelhd/PixelHDTestSupport.java`

## 如何复现 / 验证

1. `cd C:/Users/KAKA/pixelhd`
2. `./gradlew core:compileJava`
3. `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"`

本轮实际结果：
- `core:compileJava` 通过；
- `:tests:test --tests "mindustry.game.pixelhd.*"` 通过。

代码仓库提交：
- `b2150c9249` `test: 修复 PixelHD 定向测试`

## 结果 / TODO

- 结果：当前 `content-cli` 相关 HUD / Stratagem 代码已具备可编译基础，并且 PixelHD 定向测试已恢复可运行。
- 结果：非 PixelHD 地图不显示 HUD / 支援按钮的约束仍由 `HudFragment.isPixelHDMissionActive()` 控制，代码侧未回退。
- TODO：CLI 当前仍无法直接完成桌面端“人工打一局”视觉验证；待 qa-cli / 人工桌面回归时，优先在 `PixelHD_Destroy_Test` 上确认：
  - Phase 切换是否与文案一致；
  - 20s 撤离占位倒计时是否足够清晰；
  - 3 个支援按钮在移动端布局是否遮挡视野。
- 备注：`C:/Users/KAKA/pixelhd` 当前仍有一份非本 CLI 的未跟踪文件 `docs/qa/pixelhd-destroy-extraction-results-20260324.md`，我未改动它。
