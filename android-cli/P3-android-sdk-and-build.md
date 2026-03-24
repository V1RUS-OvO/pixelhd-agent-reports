# Phase 3：Android SDK + Debug APK（android-cli）

## Context
- 项目目录：C:/Users/KAKA/pixelhd
- 当前状态：
  - JDK 17 已配置；`java -version` 正常。
  - Desktop 构建可通过（修完 HudFragment / PixelHDMissionLoader 后）。
  - Android 构建在本机缺 SDK 时失败（No Android SDK found）。

## Tasks

1. 安装并配置 Android SDK
- 安装方式自行选择（Android Studio 或 winget/choco 工具链均可），目标：
  - 有一个有效的 SDK 根目录（例如 `C:\Android\Sdk` 或 Android Studio 默认路径）。
- 环境变量（至少在当前 shell 生效）：
  - 设置 `ANDROID_SDK_ROOT` 或 `ANDROID_HOME` 指向 SDK 根目录；
  - 确认 `<SDK>/platform-tools` 在 PATH 中（能找到 `adb`）。
- 验证：
  - 在任意终端执行：`adb version`，能看到版本号。

2. 重跑 Android 构建

```bash
cd C:/Users/KAKA/pixelhd
./gradlew pixelhdAndroidDev --no-daemon --stacktrace
```

- 目标：
  - 成功时：完成 `android:assembleDebug` + `android:installDebug`，在连接的设备/模拟器上安装 Debug APK；
  - 若失败：记录完整错误关键信息（缺少哪些 SDK 组件 / 平台 / 构建工具等）。

3. 更新 infra 报告
- 文件：`C:/Users/KAKA/pixelhd-agent-reports/infra-cli/P3-build-verification.md`
- 在现有内容后追加：

```markdown
## Android Build (android-cli)
- ANDROID_SDK_ROOT / ANDROID_HOME：<实际路径>
- adb version：<输出>
- ./gradlew pixelhdAndroidDev：<成功 / 失败 + 关键错误信息>
```

## Done 条件
- `adb version` 正常；
- `pixelhdAndroidDev` 至少尝试过一次，并在报告里写清成功或失败原因。