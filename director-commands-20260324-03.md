# Director Commands – 2026-03-24-03 (Execution Pass)

> 项目：pixelhd / 像素潜兵 0.1 Destroy → Extraction 闭环
> 目标：让 Destroy → Extraction 成功/失败 在 Desktop & Android 上实际跑通，并保持 pixelhd 仓库干净。

## 0. 通用约束（所有 CLI 必须遵守）

- 每次开工前：
  - `cd C:/Users/KAKA/pixelhd-agent-reports`
  - `git pull`
- 任何非微小行为完成后：
  - 在自己目录追加/更新一份简短报告（P*.md），写清：做了什么 / 改了哪些文件 / 如何验证 / 结果或 TODO；
  - 然后：
    - `cd C:/Users/KAKA/pixelhd-agent-reports`
    - `git add .`
    - `git commit -m "<cli-id>: <简短描述>"`
    - `git push`
- 与用户对话只给 1–3 行摘要 + 报告路径，细节一律写在报告里。
- 详细通讯规则仍按 `AGENT_COMMS_PROTOCOL.md` 执行。

## 1. 本轮总目标

- Destroy → Extraction 成功/失败 两条路径在 Desktop 可稳定跑通；
- PixelHD Android Debug APK 在真机/模拟器上成功安装并能打完一局 Destroy+Extraction；
- `C:/Users/KAKA/pixelhd` 仓库保持无未提交改动（除当前正在修改且已在报告中说明者）。

## 2. CLI 具体任务

### game-cli

1. 扩展失败路径：
   - 在 Destroy 完成后，支持“撤离失败”分支：超时 / 玩家或 defaultTeam core 全灭时设为 `mission_failed`，并触发 `GameOverEvent`（败方为玩家阵营或由设计决定）。
   - 补一条集成测试覆盖 `mission_failed` 分支。

2. 限定 ExtractObjective 的 GameOver 行为仅对 PixelHD 生效：
   - 避免非 PixelHD 地图使用 ExtractObjective 时被强制 GameOver。
   - 可通过 `Rules.tags` 或 objective 自身的 flags 控制是否在 `done()` 里 fire GameOverEvent。

3. 重新跑并确保：
   - `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"` 全绿。

### maps-cli

1. 使用已生成的两张地图实际跑 Desktop Destroy+Extraction：
   - `PixelHD_Destroy_Test`
   - `PixelHD_Destroy_Hard`
   - 记录：
     - 是否能正常 Destroy → 解锁撤离 → 撤离成功；
     - 撤离区域是否好找、路径是否合理；
     - 对手机/多人是否有明显不适（路径过窄、视野问题等）。

2. 在 `P3-maps-and-tags.md` 中追加：
   - 每张图推荐玩家数；
   - 当前发现的问题和建议（指明给 game-cli / content-cli / qa-cli）。

### content-cli

1. 整理并提交当前 HUD / Stratagem 改动：
   - 把 `HudFragment` / `PixelHDStratagems` / bundles / icons / `docs/design/ui-hud-0_1.md` 的改动在 pixelhd 仓库里整理成一组或少量 commit；
   - 确认：
     - `./gradlew core:compileJava` 通过；
     - `./gradlew :tests:test --tests "mindustry.game.pixelhd.*"` 通过。

2. HUD 与 Stratagem 校验：
   - 确保非 PixelHD 地图不会显示 PixelHD HUD 与支援按钮；
   - 保持 Phase 文案与当前实现一致；
   - 为后续接入“真实撤离时间”预留接口（本轮可继续使用 20s 占位，但在代码和文档中标明 TODO）。

### infra-cli & android-cli

1. 在有真机或模拟器连接时执行：
   - `cd C:/Users/KAKA/pixelhd`
   - `./gradlew pixelhdAndroidDev --no-daemon --stacktrace`

2. 在 `infra-cli/P3-build-verification.md` 与 `android-cli/P3-android-sdk-and-build.md` 中追加：
   - 设备信息；
   - `android:installDebug` 是否成功；
   - 首次运行 Destroy+Extraction 时的行为（能否进局、是否崩溃）。

### qa-cli

1. 使用 maps-cli 的两张地图执行首次 Desktop 回归：
   - `cd C:/Users/KAKA/pixelhd`
   - `bash "tools/pixelhd-desktop-regression.sh"`

2. 基于 `docs/qa/pixelhd-destroy-extraction-results-template.md` 写一份实际结果文件，至少覆盖：
   - Destroy 成功 + 撤离成功；
   - Destroy 成功 + 撤离失败；
   - 一次异常配置场景（错误 tag / 缺少核心）。

3. 在 `qa-cli/P3-destroy-flow-issues-orders.md` 追加：
   - 实际结果摘要；
   - 按责任方分类的问题列表（game/content/maps/infra/android）。

## 3. 代码仓库清理

- 本轮结束时要求：
  - 在 `C:/Users/KAKA/pixelhd` 执行 `git status` 时，工作区干净（除非某个 CLI 正在进行中的改动，并在其报告中明确说明）。

> 本文件为执行向调度令：请所有 CLI 停止“只同步指令”，立刻进入代码/地图/构建/回归的实际执行，并按上述要求通过 git 高频回报进度。
