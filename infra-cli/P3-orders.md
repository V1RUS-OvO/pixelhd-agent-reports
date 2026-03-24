# Phase 3：infra-cli 状态

当前状态：

- 0.1 开发通路：
  - `docs/engineering/dev-quickstart.md` 已定义 Desktop / Android 本地运行与调试流程；
  - `docs/engineering/release-pipeline-0_1.md` 已定义 0.1 版本策略与手工发布流程；
- 构建与 CI：
  - `android/build.gradle` 已设置：
    - `applicationId "com.pixelhd.game"`
    - 合理的 `compileSdk/targetSdk/minSdk`；
    - `versionCode = 1`, `versionName = "0.1.0"`；
  - 根 `build.gradle` 新增：
    - `pixelhdDesktopDev`（包装 `desktop:run`）
    - `pixelhdAndroidDev`（包装 `android:assembleDebug`, `android:installDebug`）
  - `.github/workflows/pr.yml` 已添加 `Android debug build` step（`./gradlew android:assembleDebug`）。

本 Phase 对 infra-cli 暂无新代码任务：

- 请在你本地环境配置好 JDK / JAVA_HOME 后，自行验证：

```bash
cd C:/Users/KAKA/pixelhd
./gradlew pixelhdDesktopDev
./gradlew pixelhdAndroidDev
```

- 确认两条命令均能完成构建；
- 如遇构建/环境问题，可在后续报告中记录，待下一 Phase 统一处理。

后续：

- 当 Destroy / 撤离 / Capture 等玩法稳定后，会安排你进入：
  - 0.1 版本的 Desktop/Android Release 自动化（构建产物上传 GitHub Release 等）；
  - 0.2 起的覆盖率/安全扫描 CI 强约束实现。

目前请保持待命，仅在报告中记录你本机的构建验证结果。