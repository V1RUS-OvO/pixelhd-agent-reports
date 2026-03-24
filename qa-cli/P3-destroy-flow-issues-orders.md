# P3 Destroy Flow QA Report (qa-cli)

## 2026-03-24 最新执行记录
- 已执行 `director-commands-20260324-02` 与总攻调度令，本次改动属于 qa-cli 的“先准备 Desktop 回归脚本和记录模板，等待 game/maps/content 第一轮落地后立即执行回归”。
- 已在 `C:/Users/KAKA/pixelhd` 新增：
  - `tools/pixelhd-desktop-regression.sh`
  - `docs/qa/pixelhd-destroy-extraction-results-template.md`
- 用途：把 qa-cli 后续桌面回归固定为一套可重复执行的入口命令 + 结果模板，减少 maps-cli 地图交付后的启动成本。
- 本次验证：
  - 在 `C:/Users/KAKA/pixelhd` 执行 `bash "tools/pixelhd-desktop-regression.sh"`，脚本输出正常。
  - 尚未执行真实 Destroy/Extraction 桌面回归；等待 maps-cli 地图名与 game-cli 流程实现落地。
- 结果 / TODO：
  - 结果：QA 准备工作已进入可执行状态，回归记录模板已标准化。
  - TODO：待 maps-cli 提供最终地图后，用该脚本串起 `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"` 与 `./gradlew pixelhdDesktopDev`，补录实际结果。

## 2026-03-24 执行阶段回归结果（director-commands-20260324-03）
- 已按调度令执行：
  - `bash "tools/pixelhd-desktop-regression.sh" PixelHD_Destroy_Test`
  - `./gradlew :tests:test --tests "mindustry.game.pixelhd.*" --no-daemon`
  - `./gradlew :tests:test --tests "PixelHDStratagemsTest.supportCalls_doNotThrow" --no-daemon`
  - `./gradlew :tests:test --tests "PixelHDMissionLoaderTest.isPixelHDMission_falseByDefault" --no-daemon`
- 已在 `C:/Users/KAKA/pixelhd` 新增实际结果文件：
  - `docs/qa/pixelhd-destroy-extraction-results-20260324.md`

### 实际结果摘要
- 场景 1（Destroy 成功 + 撤离成功）：失败 / 阻塞
  - `PixelHDMissionLoaderTest.destroyObjective_unlocksExtractionAfterTargetDestroyed()` 与 `extractObjective_setsMissionSuccessAndFiresGameOver()` 在当前本地工作树下失败。
  - 关键信息：日志报 `[PixelHD] No enemy core found for destroy mission on map 'Ground Zero'.`，说明自动化成功链路尚未跑通。
- 场景 2（Destroy 成功 + 撤离失败）：阻塞
  - 当前未见稳定可执行的 `mission_failed` 自动化结果；在成功链路未通过前，本轮未继续做 GUI 手动失败路径验证。
- 场景 3（异常配置 / 缺失 tag）：部分通过
  - `PixelHDMissionLoaderTest.isPixelHDMission_falseByDefault` 通过，说明“未打 PixelHD tag 时不进入 PixelHD 模式”的基础门控正常。
- Stratagem 基础冒烟：通过
  - `PixelHDStratagemsTest.supportCalls_doNotThrow` 通过，3 个支援入口在测试环境中调用不抛异常。

### 按责任方归类的问题
- `game-cli`
  - Critical：Destroy 成功链路自动化未跑通，目标 core 未被正确纳入测试基线，阻塞 extraction success 回归。
  - Major：尚缺稳定的 `mission_failed` 分支验证结果，导致撤离失败回归无法闭环。
- `maps-cli`
  - 当前地图资源文件已存在：`pixelhd_destroy_test.msav` / `pixelhd_destroy_hard.msav`。
  - 但本轮阻塞点不在地图文件缺失，而在 game-cli 当前测试基线与地图/测试载入方式尚未对齐。
- `content-cli`
  - Stratagem 冒烟调用通过，但 HUD 阶段切换与按钮遮挡仍需等 success/failure 链路跑通后做 GUI 验证。
- `infra-cli` / `android-cli`
  - 本轮仅做 Desktop 回归，未覆盖 Android 实机结果。

### 下一步
- 待 game-cli 修复 PixelHD success / failure 自动化后，qa-cli 立即重跑：
  - `PixelHD_Destroy_Test`：正向链路
  - `PixelHD_Destroy_Hard`：高压与失败链路
- 若下一轮可稳定进入桌面战局，再补充 HUD 文案、撤离区域可识别性与 Stratagem 视觉效果的人工观察记录。

## 当前状态
- 已同步 `director-commands-20260324.md`，qa-cli 当前目标为 Destroy -> Extraction -> 成功/失败 + HUD/Stratagem 回归场景整理。
- 已完成本轮 QA 场景设计与观察点整理，供 desktop 和 Android 共用。
- 目前尚未执行实机/桌面完整回归；实际结果区先记录为待验证，待 maps-cli / game-cli / content-cli 产物可跑后补实测结果。

## 测试前置条件
- 地图来源：使用 maps-cli 提供并标记 `pixelhd.mode = "destroy_0_1"` 的 PixelHD 专用地图。
- 构建入口：桌面优先使用 `./gradlew desktop:run`；Android 使用 `./gradlew android:assembleDebug` 与 `./gradlew android:installDebug`。
- 核心观察对象：
  - MissionPhase 文案是否按阶段切换；
  - Destroy 完成后是否进入 `extraction_unlocked`；
  - 撤离完成/失败后是否进入 `mission_success` / `mission_failed`；
  - Stratagem 按钮是否只在 PixelHD + mobile 显示，行为是否可见。

## 场景 1：正常通关

### 目标
- 覆盖 Destroy 目标 -> 解锁撤离 -> 玩家按规则撤离 -> `mission_success` -> GameOver 的完整正向闭环。

### 操作步骤
1. 启动 PixelHD 对应地图，确认存在玩家默认 core 与敌方 core。
2. 观察开局 HUD，确认任务阶段处于 Landing 或 Objective，并显示 Destroy 相关目标文案。
3. 摧毁敌方核心，观察 HUD 与任务阶段变化。
4. 确认出现撤离已解锁提示，MissionPhase 进入 `extraction_unlocked` 或后续撤离相关阶段。
5. 控制玩家进入撤离区域并持续停留至撤离完成。
6. 在撤离阶段至少触发 1 次 Stratagem，观察特效/炮塔/补给是否出现且不打断任务流程。
7. 确认任务以 `mission_success` 结束，并进入正常 GameOver / 结算流程。

### 期望结果
- HUD 文案从 Destroy 目标顺利切换到撤离提示。
- 撤离区域可识别、可进入、可完成。
- 撤离完成后触发成功结算，没有卡死、无响应或错误阶段回退。
- Stratagem 行为可见，且不会遮挡关键 HUD 或导致误判任务完成状态。

### 实际结果
- 待验证。

### 责任归属建议
- 若 Destroy 后未解锁撤离：`game-cli` / `maps-cli`
- 若 HUD 文案不对或倒计时占位错误：`content-cli`
- 若 Stratagem 无效果或遮挡严重：`content-cli`
- 若桌面或 Android 无法启动测试：`infra-cli`

## 场景 2：撤离失败

### 目标
- 覆盖 Destroy 完成后未成功撤离时的失败闭环，验证 `mission_failed` 与 GameOver 触发是否正确。

### 操作步骤
1. 启动带 `destroy_0_1` tag 的 PixelHD 地图并完成 Destroy 目标。
2. 等待撤离解锁，确认 HUD 显示撤离相关提示。
3. 不进入撤离区，或在撤离期间故意让玩家死亡 / 队伍全灭 / 超时。
4. 观察 MissionPhase 是否进入失败路径。
5. 若移动端可测，在失败前尝试使用 Stratagem，观察是否存在按钮误触导致角色失控或失败判断异常。
6. 确认任务最终进入 `mission_failed` 并触发结算流程。

### 期望结果
- 失败原因对应的 HUD 提示清楚可见。
- 状态流从撤离阶段正确转入 `mission_failed`。
- 失败后不应卡在地图内，也不应出现成功/失败双重触发。

### 实际结果
- 待验证。

### 责任归属建议
- 若失败条件不生效或阶段卡住：`game-cli`
- 若失败提示不清晰：`content-cli`
- 若地图导致无法重现失败路径：`maps-cli`

## 场景 3：异常配置 / 优雅降级

### 目标
- 验证 Destroy 目标缺失、地图 tag 错误或关键信息不完整时，游戏能优雅降级或给出提示，而不是 crash / 卡死。

### 操作步骤
1. 启动一个未正确配置 `pixelhd.mode = "destroy_0_1"` 的地图，或使用缺少敌方 core 的测试图。
2. 观察游戏是否仍能进入地图、HUD 是否给出合理状态。
3. 检查是否出现空目标、无撤离点、任务文案错乱或报错。
4. 如可进入战局，尝试基础操作与打开/使用 Stratagem UI，观察是否存在空引用、无按钮反馈或界面残缺。
5. 记录日志、报错提示和最终状态。

### 期望结果
- 游戏不崩溃、不黑屏、不无限卡住。
- 若无法启用 Destroy 模式，应回退到普通流程或明确提示当前地图不支持该模式。
- HUD / Stratagem 在异常配置下应尽量隐藏无效元素，避免误导玩家。

### 实际结果
- 待验证。

### 责任归属建议
- 若模式注入与空配置处理异常：`game-cli`
- 若地图 tag / 地图资源本身不合法：`maps-cli`
- 若 UI 在异常模式下仍暴露错误入口：`content-cli`

## 回归观察清单
- 桌面端：HUD 文案是否清晰、撤离提示是否足够明显、流程是否能稳定完成 3 次以上。
- Android 端：触控是否与移动/射击冲突、顶部 HUD 是否遮挡视野、Stratagem 按钮是否过小或易误触。
- 通用：Destroy / Extraction / Success / Fail 阶段切换是否一致，是否存在本地可见但不同步的问题。

## 当前问题与阻塞
- 尚缺 maps-cli 交付后的明确地图名与文件名，当前场景先按 director 要求抽象定义。
- 尚缺 desktop / Android 最新可运行产物验证结果，因此“实际结果”部分待补录。
- 建议 game-cli、content-cli、maps-cli 完成各自本轮提交后，由 qa-cli 统一执行一轮桌面回归，再视 Android 构建情况追加移动端回归。

## 下一步
- 等待地图、流程和 HUD/Stratagem 改动落地后补齐实际测试结果。
- 下一轮将把每次测试记录整理为：地图 / 平台 / 结果 / 问题级别 / 建议责任人。
