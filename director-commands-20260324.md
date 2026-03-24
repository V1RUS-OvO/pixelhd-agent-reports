# Director Commands – 2026-03-24

> 项目：pixelhd / 像素潜兵 0.1 Destroy → Extraction 闭环
> 角色：项目总监（主 CLI）

## android-cli

目标：打通 Android Debug 构建，让后续射击手游迭代可以在真机/模拟器上快速验证。

任务：
- 配置 Android SDK：
  - 确保已安装 Android Studio 或独立 Android SDK。
  - 设置 ANDROID_SDK_ROOT / ANDROID_HOME，保证 `adb version` 可用。
- 在 `C:/Users/KAKA/pixelhd` 下执行：
  - `./gradlew android:assembleDebug`
  - 如有设备/模拟器：`./gradlew android:installDebug`
- 如构建失败：
  - 记录缺少的 SDK 组件 / 平台 / build-tools 版本。
- 在 `pixelhd-agent-reports/infra-cli/P3-build-verification.md` 中追加：
  - SDK 路径、`adb version` 输出、assembleDebug 结果和关键信息。

---

## game-cli

目标：完成 PixelHD Destroy 任务的完整闭环：Destroy → 解锁撤离 → 撤离成功/失败 → GameOver。

任务：
- 阅读并理解：
  - `Rules.MissionRuntime` / `MissionPhase`（core/src/mindustry/game/Rules.java）。
  - `MapObjectives`（core/src/mindustry/game/MapObjectives.java）：`resetMissionRuntime`、`syncMissionRuntime`、`ExtractObjective`。
  - PixelHD 相关类：`DestroyMissionConfig`、`PixelHDMissionLoader`、`World.loadMap` 中的注入点。
- 在 Destroy 0.1 模式下：
  - 保持已实现的 Destroy → `extraction_unlocked` 行为。
  - 基于 `ExtractObjective` 完成撤离逻辑：
    - 提取 /确认撤离区域（默认队伍 core 附近，固定半径+等待时间占位即可）。
    - 当 ExtractObjective 完成时：
      - 通过 MapObjectives.setMissionOutcome(...) 设置 success / failed。
      - 让 `Rules.missionRuntime.phase` 进入 `mission_success` / `mission_failed`。
      - 触发 `GameOverEvent`，走原有结算流程。
- 补充测试：
  - 在 tests 模块中增加/扩展 PixelHD 测试：
    - Destroy 目标 → 解锁撤离 → 完成撤离 → mission_success。
    - Destroy 目标 → 解锁撤离 → 撤离失败路径 → mission_failed。
- 不新增平行任务系统，所有逻辑通过现有 Rules + MapObjectives + MissionRuntime 扩展完成。

---

## content-cli

目标：让 Destroy → Extraction 阶段在 HUD 上清晰可见，并让手机端 Stratagem 按钮有“真正的支援行为”（哪怕是占位版）。

任务：
- HUD：
  - 按 `mission-system-0_1` 和 `ui-hud-0_1` 文档，校准 MissionPhase 对应的文案：
    - Landing / Objective / Extraction_Unlocked / Extraction_Called / Success / Fail。
  - 去掉纯占位的撤离倒计时数值：
    - 与 game-cli 协调，从 MissionRuntime 或相关配置中获取真实时间（如果暂时没有，则在报告中标注仍为占位）。
- Stratagem：
  - 保留现有 3 个按钮布局，仅在 PixelHD + mobile 时显示。
  - 为 `PixelHDStratagems.callSupport1/2/3` 实现简单但可见的行为：
    - 支援一：范围爆炸或强力效果。
    - 支援二：在玩家附近生成临时炮塔/防御设施（可复用现有 block/unit 类型）。
    - 支援三：简单补给/回复效果（或暂时复用支援一/二，占位即可）。
  - 注意：
    - 行为尽量通过内容系统完成；
    - 如当前只在本地生效，需在 content-cli 报告中说明，方便后续 net 同步改造。
- 文档：
  - 更新 `docs/design/ui-hud-0_1.md`：记录实际使用的文案 key 和 Stratagem UI 行为。
  - 在 `pixelhd-agent-reports/content-cli` 新增一篇报告，列出本次 HUD/Stratagem 改动与验证方法。

---

## maps-cli

目标：提供至少 1–2 张适合 Destroy 0.1 垂直切片的 PixelHD 专用地图，支持 Desktop + Android 验证。

任务：
- 在 pixelhd 中创建/选取 1–2 张地图：
  - 必须包含：
    - 玩家默认核心（defaultTeam）。
    - 敌方核心（waveTeam）。
  - 地图尺寸不宜过大，便于手机端操作与观察。
- 为这些地图添加 tag：
  - `pixelhd.mode = "destroy_0_1"`，以触发 PixelHDMissionLoader。
- 使用 Desktop 验证 Destroy 流程：
  - 加载上述地图；
  - 确认 Destroy 目标为敌方 core；
  - 摧毁目标后，MissionRuntime 进入 `extraction_unlocked`（可用日志或 HUD 观察）。
- 在 `pixelhd-agent-reports/maps-cli/P3-maps-and-tags.md` 中记录：
  - 地图名称/文件；
  - tag 配置；
  - 适用的测试场景（比如 Destroy+Extraction 验证）。

---

## infra-cli

目标：让开发者和 CI 有一条清晰、可复用的 PixelHD 构建/测试路径，尤其覆盖 Destroy+Extraction 的测试。

任务：
- 汇总并写入文档的推荐命令：
  - Desktop 开发：`./gradlew desktop:run` 或已有的 `pixelhdDesktopDev` 任务（视当前配置而定）。
  - PixelHD 测试：`./gradlew :tests:test --tests "mindustry.game.pixelhd.*"`。
  - Android Debug：`./gradlew android:assembleDebug`，安装使用 `android:installDebug`。
- 更新：
  - `docs/engineering/dev-quickstart.md`：增加一小节，专门写 PixelHD Destroy 任务的开发/测试循环。
  - `pixelhd-agent-reports/infra-cli/P3-build-verification.md`：记录最新 Desktop/Android/Tests 状态与已知缺口。
- 审阅 `.github/workflows/*.yml`：
  - 检查 CI 是否已执行 tests 模块；
  - 如未覆盖 PixelHD 测试，用文字形式在报告中提出建议（暂不强改 CI）。

---

## qa-cli

目标：为 Destroy → Extraction → 成功/失败 + HUD/Stratagem 提供一套可复现的 QA 场景，用于回归。

任务：
- 定义至少 3 组测试场景（Desktop 和 Android 皆适用）：
  1. 正常通关：Destroy 目标 → 解锁撤离 → 玩家按规则撤离 → mission_success。
  2. 撤离失败：Destroy 目标完成，但未成功撤离（超时或全队阵亡） → mission_failed。
  3. 异常配置：Destroy 目标缺失或地图 tag 配置错误时的行为（游戏应优雅降级或提示，而不是卡死/crash）。
- 为每个场景写清操作步骤：
  - 使用 maps-cli 提供的 PixelHD Destroy 地图；
  - 说明期望看到的 HUD 文案和 MissionPhase 变化；
  - 标记与 Stratagem 交互相关的观察点（例如在撤离阶段使用支援的效果）。
- 在 `pixelhd-agent-reports/qa-cli/P3-destroy-flow-issues-orders.md` 中记录：
  - 每个场景的实际结果；
  - 发现的问题及推荐责任人（game-cli / content-cli / maps-cli / infra-cli）。

---

> 所有 CLI：完成任务后，请在自己的子目录下追加一份 P3/P4 报告，便于主 CLI 和其他代理统一追踪进度。
