# Phase 3：Destroy 流程内测 & 问题清单（qa-cli）

## Context
- PixelHD Destroy + 撤离 + 支援 的 0.1 垂直切片已完成：
  - Destroy 目标 core → 解锁撤离 → 进入撤离区 → mission_success → GameOver；
  - 支援按钮调用 PixelHDStratagems，在玩家附近生成冲击波特效；
  - HUD 显示任务阶段、当前目标、撤离提示和简单倒计时。
- maps-cli 会提供若干 PixelHD 测试地图（带 `pixelhd.mode = destroy_0_1`）。

## Tasks

1. 桌面内测：至少完整跑 3 局 Destroy → 撤离

- 针对每张 PixelHD 测试地图，至少完整跑 1 局：
  - 记录：
    - 是否能稳定完成整条流程；
    - 在 Destroy / 撤离 各阶段 HUD 是否清晰可懂；
    - 支援按钮是否可用，是否易误触；
    - 有无明显性能问题（卡顿、掉帧）。

2. 手机/模拟器内测（如条件允许）

- 在 Android Debug APK 可用后：
  - 在真机/模拟器上跑 1–2 局 Destroy → 撤离；
  - 特别关注：
    - 触控操作冲突（支援按钮 + 射击 + 走位）；
    - 顶部信息是否挡视野；
    - 字体/按钮是否过小/过大。

3. 输出问题清单

- 文件：`C:/Users/KAKA/pixelhd-agent-reports/qa-cli/P3-destroy-flow-issues.md`
- 建议结构：

```markdown
# P3 Destroy Flow QA

## Critical
- [ ] <严重影响玩下去的问题，例如流程无法完成、严重崩溃>

## Major
- [ ] <明显体验问题，例如撤离提示看不懂、支援误触严重>

## Minor / Polish
- [ ] <UI/文案/手感小问题>

## Test Runs
- Map: <地图名> / Device: <PC/Android 型号> / Result: <通过/失败 + 简述>
```

## Done 条件
- 至少有 3 次完整 Destroy → 撤离 流程的测试记录；
- 问题清单中有明确的待修项（哪怕只有几条），供下一轮开发作为输入。