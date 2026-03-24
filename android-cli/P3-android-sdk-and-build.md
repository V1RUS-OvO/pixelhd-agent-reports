# Phase 3：Android SDK + Debug APK（android-cli）

## Context
- 项目目录：C:/Users/KAKA/pixelhd
- 当前状态（2026-03-24 更新）：
  - JDK 17 已配置；`java -version` 正常。
  - Desktop 构建可通过（修完 HudFragment / PixelHDMissionLoader 后）。
  - Android SDK 已配置：`ANDROID_SDK_ROOT` / `ANDROID_HOME` = `C:/Users/KAKA/AppData/Local/Android/Sdk`，`adb version` 正常。
  - `./gradlew pixelhdAndroidDev --no-daemon --stacktrace` 最新结果：已成功完成 `:pixelhd:android:assembleDebug`，Debug APK 构建通过。

## Actions

1. SDK 与环境验证
- 命令：
  ```bash
  adb version
  ```
- 结果：
  - Android Debug Bridge version 1.0.41（platform-tools 37.0.0-14910828），路径 `C:/Users/KAKA/AppData/Local/Microsoft/WinGet/Packages/Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe/platform-tools/adb.exe`。

2. 修复 base.jar / R8 构建问题
- 目标：解决之前 `:pixelhd:android:minifyDebugWithR8` 因 `android/build/intermediates/merged_java_res/debug/base.jar` 缺失导致的 `NoSuchFileException`。
- 命令：
  ```bash
  cd C:/Users/KAKA/pixelhd
  ./gradlew :android:mergeDebugJavaResource --no-daemon --stacktrace
  ```
- 结果：
  - 任务 `:android:mergeDebugJavaResource` 成功，目录 `android/build/intermediates/merged_java_res/debug` 下出现 `base.jar` 与 `mergeDebugJavaResource/`；
  - 说明基础 android 模块的 Java 资源合并流程正常运行，能够生成 R8 期望的 base.jar。

3. 重新验证 PixelHD Android Debug 构建
- 命令：
  ```bash
  cd C:/Users/KAKA/pixelhd
  ./gradlew pixelhdAndroidDev --no-daemon --stacktrace
  ```
- 关键日志摘要：
  - `:pixelhd:android:mergeDebugJavaResource` / `:pixelhd:android:minifyDebugWithR8` 等任务均为 UP-TO-DATE，不再抛出 base.jar 缺失错误；
  - 最终成功执行 `:pixelhd:android:assembleDebug`，构建 Debug APK 完成；
  - 本轮尚未执行 `android:installDebug`，等待接入真实设备/模拟器后再补验证。

## Verification / Repro

- 复现步骤：
  ```bash
  cd C:/Users/KAKA/pixelhd
  # 可选：单独验证 android 模块资源合并
  ./gradlew :android:mergeDebugJavaResource --no-daemon --stacktrace

  # 主命令：构建 PixelHD Android Debug APK
  ./gradlew pixelhdAndroidDev --no-daemon --stacktrace
  ```
- 期望结果：
  - 构建流程中不再出现 `android/build/intermediates/merged_java_res/debug/base.jar` 相关的 `NoSuchFileException`；
  - `:pixelhd:android:assembleDebug` 成功；
  - `android/build/outputs/apk/debug/` 下存在最新 Debug APK（具体文件名由 Gradle/AGP 决定）。

## Status & TODO

- Status：
  - PixelHD Android Debug APK 已能在本机成功构建；
  - 之前的 R8/base.jar 缺失问题已通过重新运行 `:android:mergeDebugJavaResource` 得到缓解，当前构建路径稳定通过。
- TODO：
  - 在连接 Android 设备或模拟器后补跑：
    ```bash
    cd C:/Users/KAKA/pixelhd
    ./gradlew android:installDebug --no-daemon --stacktrace
    ```
    并记录安装及首启行为；
  - 如后续接入 CI 或其他环境出现相同 base.jar 问题，再比对本报告的命令顺序与差异，补充稳定化方案。