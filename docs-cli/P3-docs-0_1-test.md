# Phase 3：0.1 内测说明文档（docs-cli）

## Context
- 即将准备 PixelHD 0.1 的“内测版”：Destroy + 撤离 + 支援 的基础体验；
- 需要一个给测试玩家看的说明文档，告诉他们：怎么玩、测什么、有哪些已知问题。

## Tasks

1. 在项目中创建内测说明文档

- 文件路径：`C:/Users/KAKA/pixelhd/docs/README-pixelhd-0_1-test.md`
- 建议内容结构：

```markdown
# PixelHD 0.1 内测说明

## 这是什么
- 一句话说明：Mindustry Fork，类 Helldivers 2 的任务制合作玩法，当前是 Destroy + 撤离 垂直切片。

## 如何启动
- Desktop：
  - `./gradlew desktop:run` 或通过启动器（如有）。
- Android：
  - 安装 Debug APK（如有发放），启动后在地图列表中选择带 PixelHD 标记的地图。

## 测试重点
- 流程：
  - 是否能顺利完成 Destroy → 撤离 → 成功。
- HUD：
  - 阶段提示/目标文字是否看得懂；
  - 撤离倒计时是否清晰；
- 操作与支援：
  - 移动/射击是否顺手；
  - 支援按钮调用是否直观，有无误触；

## 已知限制
- 只支持 Destroy 任务类型；
- 支援效果较简单，主要是视觉提示；
- 长期战线/星图、复杂 Stratagem 输入尚未实现。

## 如何反馈
- 建议反馈格式（贴给玩家）：
  - 设备/平台：
  - 地图：
  - 描述：
  - 期望：
  - 复现步骤：
```

2. 在报告中记录文档位置

- 文件：`C:/Users/KAKA/pixelhd-agent-reports/docs-cli/P3-readme-0_1.md`
- 写入：
  - 文档已创建（路径）；
  - 实际目录结构中可见路径截图/描述；
  - 如你对文案有额外补充，也可简单记在这份报告中。

## Done 条件
- 仓库中存在 `docs/README-pixelhd-0_1-test.md`，内容基本完整；
- docs-cli 报告中注明该文档的位置和大致用途。