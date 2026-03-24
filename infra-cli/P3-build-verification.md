# P3 - Infra Build Verification (infra-cli)

## Summary

- 本 Phase 对 infra-cli **没有新增代码/配置任务**，主要目标是：
  - 在本地按 `dev-quickstart.md` 验证 Desktop / Android 开发命令；
  - 记录当前构建是否能跑通，以及若失败的阻塞原因，供后续 Phase 统一处理。
- 目前在本环境中尝试执行 Android 构建命令时，阻塞在 **JDK/JAVA_HOME 未配置**，尚未完成完整构建验证。

## Changed Files

- 本 Phase **未修改任何代码或配置文件**。
- 相关上下文仍沿用前一 Phase 已完成的改动：
  - `android/build.gradle` 中的 `applicationId` / SDK / version 配置；
  - 根 `build.gradle` 中的 `pixelhdDesktopDev` / `pixelhdAndroidDev` 任务；
  - `.github/workflows/pr.yml` 中的 `Android debug build` step。

## Verification

在当前环境下尝试按 P3-orders 执行以下命令：

```bash
cd C:/Users/KAKA/pixelhd
./gradlew pixelhdDesktopDev
./gradlew pixelhdAndroidDev
```

- 实际执行中，底层 `./gradlew android:assembleDebug` 报错：

```text
ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.

Please set the JAVA_HOME variable in your environment to match the
location of your Java installation.
```

- 说明：
  - Gradle Wrapper 已存在，任务定义无语法错误；
  - 当前阻塞点在 **本机 Java 环境尚未配置**，而非构建脚本本身；
  - 一旦 JDK 安装完毕且 `JAVA_HOME` / PATH 设置正确，上述命令才有意义地继续验证。

## Risks / TODO

- **环境未就绪 → 无法真正验证 dev-quickstart 流程**
  - 在没有 JDK / JAVA_HOME 的情况下，所有基于 Gradle 的验证都无法进行，包括：
    - `./gradlew pixelhdDesktopDev`（Desktop 调试）；
    - `./gradlew pixelhdAndroidDev`（Android Debug 构建 + 安装）；
    - CI 本地复现实验（`./gradlew android:assembleDebug` 等）。

- **TODO（需由使用者或后续 Phase 处理）**
  1. 安装与 CI 一致版本的 JDK（建议 Java 17）。
  2. 在当前 Shell / 系统中正确配置：
     - `JAVA_HOME` 指向 JDK 安装目录；
     - 将 `$JAVA_HOME/bin` 加入 PATH。
  3. 再次执行：
     ```bash
     cd C:/Users/KAKA/pixelhd
     ./gradlew pixelhdDesktopDev
     ./gradlew pixelhdAndroidDev
     ```
  4. 若命令仍因构建脚本或依赖问题失败，在后续 Phase 的报告中详细记录新的错误信息。

- **对后续工作的影响**
  - 在 JDK 环境配置完成之前，虽然 CI（GitHub Actions）仍可完成远端构建与测试，但本地无法按 `dev-quickstart` 实际跑起游戏，对开发调试效率有影响；
  - 建议在进入“构建/发布自动化”和“CI 强约束” Phase 之前优先解决本机 JDK 配置问题。

## Local Build Verification (opencode)

- JDK 版本：Temurin OpenJDK 17.0.18+8（`openjdk version "17.0.18" 2026-01-20`）
- `pixelhdDesktopDev`：失败。`JAVA_HOME` 问题已解决，但编译卡在代码错误：`core/src/mindustry/ui/fragments/HudFragment.java:425/427/429` 的 `c.button(String, ImageButtonStyle, Runnable)` 无匹配重载；同时 `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java:29` 的 `map.tags.get(MODE_KEY, rules.tags.get(MODE_KEY, null))` 调用不匹配，最终任务 `:pixelhd:core:compileJava` 失败。
- `pixelhdAndroidDev`：失败。当前环境仍提示 `No Android SDK found. Skipping Android module.`，随后 `:pixelhdAndroidDev` 内部调用 `android:assembleDebug` 时失败：`Cannot locate tasks that match 'android:assembleDebug' as project 'android' not found in project ':pixelhd'.`
- 备注：本机 JDK 已安装并在当前 shell 验证可用，后续阻塞已从环境问题转为项目/依赖问题：Desktop 是 Java 源码编译错误，Android 是本机未配置 Android SDK 导致模块被跳过、任务不存在。

## Desktop & Android Build (opencode-build)

- Desktop:
  - `./gradlew pixelhdDesktopDev --no-daemon`：源码编译问题已修复；任务成功进入 `:pixelhd:desktop:run` 并拉起 Desktop，日志显示游戏完成初始化（SDL / OpenGL / Mindustry custom 版本已启动）。本次 CLI 会话在应用持续运行约 120s 后超时终止，因此任务未自然退出，但 Desktop 已可启动。
  - 本次修复点：
    - `core/src/mindustry/ui/fragments/HudFragment.java`：将 mission phase 的 Java 14+ `switch ->` 改为普通 `switch/case`，并把支援按钮样式改为 `TextButtonStyle` 兼容现有 `Table.button(String, TextButtonStyle, Runnable)` 重载。
    - `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java`：改为先读 `map.tags`，为空再回退到 `rules.tags`，避免 `ObjectMap.get(..., null)` 重载歧义。
- Android:
  - SDK：已检测到 `C:\Users\KAKA\AppData\Local\Android\Sdk`，并补充了本地 `local.properties`：`sdk.dir=C:\Users\KAKA\AppData\Local\Android\Sdk`。
  - `adb version`：成功，版本为 `1.0.41`（platform-tools `37.0.0-14910828`）。
  - `./gradlew pixelhdAndroidDev --no-daemon --stacktrace`：Android 模块已成功纳入配置，不再报 `No Android SDK found`；当前新的阻塞点为 `:pixelhd:android:mergeDebugResources`，AAPT 资源编译失败，关键错误：
    - `FileNotFoundException: C:\Users\KAKA\pixelhd\android\build\intermediates\incremental\debug\mergeDebugResources\merged.dir\values\values.xml`
    - `FileNotFoundException: C:\Users\KAKA\pixelhd\android\build\intermediates\incremental\debug\mergeDebugResources\merged.dir\values-v21\values-v21.xml`
  - 额外信息：构建过程中还有 AGP/compileSdk 提示（AGP `8.2.2` 对 `compileSdk = 36` 仅给出 warning），但当前实际失败点不是 SDK 缺失，而是 Android 资源合并阶段文件不存在。
- 环境：
  - `JAVA_HOME`：`C:\Program Files\Eclipse Adoptium\jdk-17.0.18.8-hotspot`
  - `ANDROID_HOME`：`C:/Users/KAKA/AppData/Local/Android/Sdk`（本次构建命令中已设置）
  - `ANDROID_SDK_ROOT`：`C:/Users/KAKA/AppData/Local/Android/Sdk`（本次构建命令中已设置）

## Director Sync (2026-03-24)

- 已按协作要求在 `C:/Users/KAKA/pixelhd-agent-reports` 执行 `git pull`，当前通讯仓库同步正常。
- 已重新确认当前身份为 `infra-cli`，并读取最新总监文件 `director-commands-20260324.md`。
- 当前 infra-cli 后续执行项保持不变：
  - 更新 `C:/Users/KAKA/pixelhd/docs/engineering/dev-quickstart.md`，补充 PixelHD Destroy 开发/测试循环；
  - 审阅 `C:/Users/KAKA/pixelhd/.github/workflows/*.yml`，确认 tests 模块与 PixelHD 测试覆盖情况；
  - 持续在本报告记录 Desktop / Android / Tests 的验证结果与已知缺口。
- 本次同步报告未新增源码仓库改动，属于指令对齐与执行准备状态更新。

## Director Sync (2026-03-24-02)

- 已执行 `director-commands-20260324-02`，本次改动属于“infra-cli & android-cli：继续围绕 `./gradlew pixelhdAndroidDev` 排查 Android 构建问题”的准备同步。
- 已重新读取 `AGENT_COMMS_PROTOCOL.md` 与最新调度令，并确认本轮 infra-cli 最高优先级不再分散，集中在 Android Debug 构建链路排障。
- 当前 infra-cli 下一步聚焦项：
  - 继续排查 `./gradlew pixelhdAndroidDev` 在 Android 资源/打包阶段的失败原因；
  - 如构建恢复，再补充 `docs/engineering/dev-quickstart.md` 中 PixelHD Destroy 开发/测试循环；
  - 审阅 `.github/workflows/*.yml`，核对 tests 模块与 PixelHD 测试覆盖情况，并把结论补充到本报告。
- 本次同步仅更新调度认知与执行优先级，尚未新增 `pixelhd` 源码改动。

## Android Build (android-cli)
- ANDROID_SDK_ROOT / ANDROID_HOME：`C:/Users/KAKA/AppData/Local/Android/Sdk`
- adb version：
  ```text
  Android Debug Bridge version 1.0.41
  Version 37.0.0-14910828
  Installed as C:\Users\KAKA\AppData\Local\Microsoft\WinGet\Packages\Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe\platform-tools\adb.exe
  ```
- ./gradlew pixelhdAndroidDev：失败，Android 模块已参与构建，当前阻塞点为 `:pixelhd:android:minifyDebugWithR8`，关键错误：
  - R8 处理 `C:\Users\KAKA\pixelhd\android\build\intermediates\merged_java_res\debug\base.jar` 时抛出 `NoSuchFileException`，提示该文件不存在；
  - 说明当前环境下 Android SDK 与 Gradle 配置已生效，但 Debug 构建在资源压缩/混淆阶段仍有构建产物缺失问题，后续需要针对 R8 / merged_java_res 的生成流程进一步排查。

## Android Dev Task Fix (2026-03-24)

- 已执行 `director-commands-20260324-02`，本次改动属于“infra-cli & android-cli：继续围绕 `./gradlew pixelhdAndroidDev` 排查 Android 构建问题”。
- 做了什么：
  - 在 `C:/Users/KAKA/pixelhd/build.gradle` 为 `pixelhdAndroidDev` 增加设备检测逻辑；
  - 让该快捷任务在检测到 `adb devices` 有真实连接设备/模拟器时继续执行 `android:installDebug`，无设备时自动退化为仅执行 `android:assembleDebug`；
  - 更新 `C:/Users/KAKA/pixelhd/docs/engineering/dev-quickstart.md`，把新的 Android 开发循环写入文档。
- 改动文件：
  - `C:/Users/KAKA/pixelhd/build.gradle`
  - `C:/Users/KAKA/pixelhd/docs/engineering/dev-quickstart.md`
- 如何复现 / 验证：
  ```bash
  cd C:/Users/KAKA/pixelhd
  ./gradlew android:assembleDebug --no-daemon --stacktrace
  ./gradlew pixelhdAndroidDev --no-daemon
  adb devices
  ```
- 验证结果：
  - `./gradlew android:assembleDebug --no-daemon --stacktrace`：成功；此前 `mergeDebugResources` / R8 阶段失败在本机未复现。
  - `./gradlew pixelhdAndroidDev --no-daemon`：成功；当前无连接设备时不再因为 `installDebug` 报 `No connected devices!` 而整体失败。
  - `adb devices`：当前为空，因此本次仅验证到 APK 构建成功，未执行真机安装。
  - 产物：`C:/Users/KAKA/pixelhd/android/build/outputs/apk/debug/android-debug.apk`
- CI 审阅：
  - `C:/Users/KAKA/pixelhd/.github/workflows/pr.yml` 已执行 `./gradlew tests:test --stacktrace --rerun` 和 `./gradlew android:assembleDebug`；
  - `C:/Users/KAKA/pixelhd/.github/workflows/push.yml` 已执行 `./gradlew tests:test --rerun --stacktrace`；
  - 当前 CI 已覆盖 `tests` 模块，但尚未显式只跑 `mindustry.game.pixelhd.*` 过滤集；若后续 PixelHD 测试数量扩大，可补一条定向任务提高信号可见度。
- 源码仓库提交：`fa91b52817`（`infra: 优化 Android 开发任务`）
- TODO：
  - 连接真机或模拟器后再次执行 `./gradlew pixelhdAndroidDev`，补记 `installDebug` 与实际运行结果；
  - 待 game-cli / content-cli / maps-cli 第一轮功能落地后，补充 `:tests:test --tests "mindustry.game.pixelhd.*"` 的最新验证结论。
