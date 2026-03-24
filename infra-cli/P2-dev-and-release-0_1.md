# P2 - Dev & Release 0.1 (infra-cli)

## Summary

- 为 PixelHD 0.1 版本设计并落地了开发与发布通路的文档骨架：
  - `docs/engineering/dev-quickstart.md`：定义 Desktop + Android 本地运行/调试的标准流程，以及推荐的“任务开发循环”。
  - `docs/engineering/release-pipeline-0_1.md`：定义 0.1 阶段的版本编号策略、构建产物类型、手工发布流程，以及最小 CI 需求（不直接修改 YAML）。
- 目标是：在不大改 CI 的前提下，让新开发者可以按文档快速启动开发，并为后续 0.2 的 CI 自动化和强约束打基础。

## Changed Files

- `docs/engineering/dev-quickstart.md`
  - 新增文件。
  - 内容：
    - 前置环境要求（JDK、Git Bash、Android Studio/adb 等）。
    - Desktop 模式调试：`./gradlew desktop:run` 与 IDE Debug 说明。
    - Android 模式调试：`./gradlew android:assembleDebug`、`./gradlew android:installDebug`，以及在 Android Studio 中 Run/Debug 的步骤。
    - 推荐的“任务开发循环”：先在 Desktop 验证任务逻辑，再在 Android 验证手感与性能，问题再回到 Desktop 复现与修复，最后按 `/plan` `/tdd` `/test-coverage` `/verify`（必要时 `/security-review`）收尾。

- `docs/engineering/release-pipeline-0_1.md`
  - 新增文件。
  - 内容：
    - 版本策略：0.1 阶段使用 `0.1.x`（Tag 形如 `v0.1.0`），区分内部/测试发布。
    - 构建产物：
      - Desktop：开发/展示用发行包（示例命令：`./gradlew desktop:dist`）。
      - Android：Debug / Release APK（示例命令：`./gradlew android:assembleDebug` / `./gradlew android:assembleRelease`）。
    - 手工发布流程：
      - 从 `main` 或 `release/0.1` 选定发版 Commit，确保相关 PR / push CI 全绿；
      - 在本地或 CI 中构建 Desktop/Android 产物；
      - 创建 Git Tag + GitHub Release，并上传对应产物；
      - 将 Release 链接或 APK 分发给测试/目标用户。
    - Minimal CI Requirements for 0.1：
      - PR 必须通过 `pr.yml` 中的构建/测试 Job；
      - push 到关键分支（`main` / `release/0.1`）必须通过 `push.yml`；
      - 发版 Commit 需满足上述 CI 全绿；
      - 覆盖率与安全扫描等强约束先标为 TODO，计划在 0.2 起逐步落地。

## Verification

> 目标：验证文档是否足够让新开发者“按图索骥”跑起 0.1 的开发与发布流程。

1. **按 dev-quickstart 跑 Desktop 调试**
   - 前置：本机安装 JDK（版本满足项目要求）、Git Bash。
   - 步骤：
     1. 进入项目根目录：`cd C:/Users/KAKA/pixelhd`。
     2. 执行 `./gradlew desktop:run`。
     3. 预期：Desktop 版游戏成功启动，可用于快速迭代任务逻辑；如在 IDE 中导入项目并以 `desktop` 模块 Debug，应能打断点调试任务/状态机代码。

2. **按 dev-quickstart 跑 Android 调试**
   - 前置：Android Studio / Android SDK / adb 已就绪，有真机或模拟器可连接。
   - 步骤：
     1. 在项目根目录执行：`./gradlew android:assembleDebug`。
     2. 设备连上后执行：`./gradlew android:installDebug`。
     3. 或在 Android Studio 中以 `android` 模块 Run/Debug 到目标设备。
     4. 预期：APK 构建成功并能在设备上启动，适合验证手感与性能。

3. **按 release-pipeline-0_1 跑一条手工发布“演练”**
   - 步骤：
     1. 选择一个稳定 Commit，并保证相关 PR / push 的 CI（`pr.yml` / `push.yml`）为绿灯。
     2. 本地执行：
        - Desktop：`./gradlew desktop:dist`（如配置存在）；
        - Android Debug：`./gradlew android:assembleDebug`。
     3. 进入 GitHub，针对该 Commit 创建 Tag（如 `v0.1.0-dev`）与 Release，并上传上述产物。
     4. 预期：所有步骤均有对应文档指引，执行过程中不需要额外口头说明。

4. **核对 Minimal CI Requirements for 0.1**
   - 在 GitHub Actions 页面检查：
     - PR 打开时，是否始终触发 `pr.yml`，且相关 Job（构建/测试）通过；
     - push 到 `main` 时，是否触发 `push.yml` 并成功；
   - 验证 release-pipeline 文档对这些要求的描述是否与实际 CI 行为一致。

## Risks

- **文档与实际 Gradle/CI 配置可能出现漂移**
  - 后续如果调整模块结构、任务名或 CI Workflow（如新增 Android 构建 Job），需要同步更新 dev-quickstart 与 release-pipeline 文档，否则新同事会按错命令。

- **环境差异带来的偏差**
  - 文档基于 Windows + Git Bash + Android Studio 的典型环境，其他平台（Linux/macOS）可能在命令前缀、路径格式上略有差异，需要后续增加平台注记。

- **0.1 阶段仍依赖人工执行流程**
  - 目前发布流程为“文档驱动 + 手工操作”，如果执行者忽略某些步骤（如忘记检查 CI 状态），仍有可能导致不稳定版本被发出；
  - 该风险预计在 0.2 开始通过 CI 自动化（自动产物构建、自动上传、硬性校验）逐步缓解。

- **覆盖率与安全扫描仍是 TODO**
  - 文档中已明确标注覆盖率检查与安全扫描会在 0.2 起逐步收紧，但短期内 0.1 仍可能存在：
    - 个别关键代码路径测试覆盖率不足；
    - 依赖安全或 Secret 提交缺少自动化防线。
  - 需要在后续 Phase 有明确的 CI 工作项，将这些 TODO 转成具体的 workflow 实现。
