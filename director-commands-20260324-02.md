# Director Commands – 2026-03-24-02 (High Priority)

> 项目：pixelhd / 像素潜兵 0.1 Destroy → Extraction 闭环
> 本文件在 `director-commands-20260324.md` 基础上，收紧当前一轮的执行优先级和要求。

## 0. 通用要求（所有 CLI 必须遵守）

- 每次开始工作前：
  - `cd C:/Users/KAKA/pixelhd-agent-reports`
  - `git pull`
- 高频通过 git 报告进度：任何非微小行为完成后，必须在自己目录追加/更新一份简短报告，并：
  - `git add .`
  - `git commit -m "<cli-id>: <简短描述>"`
  - `git push`
- 与用户对话只给 1–3 行摘要，细节全部写在报告里。
- 详细规则见：`AGENT_COMMS_PROTOCOL.md`。

## 1. 本轮最高优先级

优先级顺序（从高到低）：

1. **game-cli**
   - 立即开始实现 Destroy → ExtractObjective → `mission_success` / `mission_failed` → `GameOverEvent` 闭环。
   - 在 `pixelhd` 源码中落地：
     - 使用现有 `MapObjectives.ExtractObjective` 和 `Rules.MissionRuntime`；
     - 不另造任务系统，不破坏非 PixelHD 模式。
   - 最少补一条集成测试覆盖成功路径。

2. **maps-cli**
   - 在 `pixelhd` 工程中创建至少 2 张带 `pixelhd.mode = "destroy_0_1"` 的测试地图，并在 `P3-maps-and-tags.md` 记录清单。
   - 保证 game-cli / qa-cli 能用这些地图复现 Destroy + Extraction 流程。

3. **content-cli**
   - 配合 game-cli 的 MissionPhase，实现可读的 HUD 文案：Landing / Objective / Extraction_Unlocked / Extraction_Called / Success / Fail。
   - 保持 3 个手机端 Stratagem 按钮，只在 PixelHD+mobile 时显示，并为 `PixelHDStratagems.callSupport1/2/3` 提供可见的占位行为。

4. **infra-cli & android-cli**
   - 继续围绕 `./gradlew pixelhdAndroidDev` 排查 Android 构建问题，优先解决当前 R8/资源阶段的错误。
   - 一旦 Debug APK 可安装，记录设备型号与基本运行结果。

5. **qa-cli**
   - 暂不跑实机回归，先维持 P3 场景文档；
   - 等待 game-cli + maps-cli + content-cli 第一轮实现落地后，再执行桌面端 Destroy/Extraction 回归。

## 2. 报告约定

- 报告文件命名继续沿用各目录现有习惯（`P3-*.md`, `P4-*.md` 等）。
- 每次提交报告时，在开头简要写明：
  - "已执行 director-commands-20260324-02，本次改动属于哪一条任务"。

> 本文件发布后，上一版 `director-commands-20260324.md` 中与本轮优先级冲突的地方，以本文件为准。