# Task: CODE-G-03

## Summary

- 为 PixelHD 模式实现了 Destroy 任务 0.1 垂直切片的核心逻辑：
  - 在带有 `pixelhd.mode = "destroy_0_1"` tag 的地图上，自动挂载一个 `DestroyBlockObjective`，目标为敌方 waveTeam 的第一个 core。
  - 当目标 core 被摧毁时，通过 `flagsAdded(Rules.MissionRuntime.flagExtractionUnlocked)` 与 `MapObjectives.syncMissionRuntime()` 联动，将任务状态切换到 `MissionPhase.extraction_unlocked`。
- 删除了对 `flagMissionStarted` 的硬写，让 `MissionRuntime` 生命周期完全由 `MapObjectives.resetMissionRuntime/syncMissionRuntime` 管理。
- 新增集成测试覆盖“Destroy → Extraction 解锁”链路，确保行为在 headless 测试环境下稳定可重放。

## Changed Files

- `core/src/mindustry/game/pixelhd/DestroyMissionConfig.java`
  - 新增 Destroy 任务配置数据类，描述目标 tile 坐标、目标 block 以及可选时间限制，为后续扩展/配置留出空间。
- `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java`
  - 新增 PixelHD 任务 Loader：
    - 通过 `pixelhd.mode == "destroy_0_1"` 检测是否启用 Destroy 任务。
    - 选取 `rules.waveTeam` 的第一个 core tile 作为 Destroy 目标。
    - 创建 `MapObjectives.DestroyBlockObjective` 并设置 `flagsAdded(flagExtractionUnlocked)`，完成时自动解锁撤离。
    - 确保 `rules.objectives` 存在并追加该 objective，确保 `rules.missionRuntime` 非 null。
    - 移除了对 `flagMissionStarted` 的直接写入，避免与 MapObjectives 重置逻辑打架。
- `core/src/mindustry/core/World.java`
  - 在 `loadMap(Map map, Rules checkRules)` 中，在 map 合法性校验通过且 `invalidMap == false` 时调用：
    - `PixelHDMissionLoader.initPixelHDMission(state, state.rules, this, map);`
  - 对非 PixelHD 地图无行为变化，对 PixelHD Destroy 模式地图则完成 Destroy 任务的注入。
- `tests/src/test/java/mindustry/game/pixelhd/PixelHDMissionLoaderTest.java`
  - 新增测试类：
    - `isPixelHDMission_falseByDefault`：验证默认 `ApplicationTests.testMap` 不会误识别为 PixelHD 模式。
    - `destroyObjective_unlocksExtractionAfterTargetDestroyed`：
      - 使用 `ApplicationTests.launchApplication(false)` 初始化完整 headless 环境。
      - 为 `testMap.copy()` 打 tag：`pixelhd.mode = "destroy_0_1"`。
      - 调用 `world.loadMap(map, rules)` 触发 PixelHD Loader 注入 DestroyBlockObjective。
      - 找到 `state.teams.cores(rules.waveTeam).first().tile` 作为目标，使用 `targetTile.removeNet()` 模拟摧毁目标。
      - 调用 `rules.objectives.update()` 推进 MapObjectives 状态机，并断言：
        - `rules.missionRuntime.extractionUnlocked == true`；
        - `rules.missionRuntime.phase == Rules.MissionPhase.extraction_unlocked`。

## Verification

### 自动化

在 pixelhd 仓库根目录执行（命令按你的构建环境调整）：

```bash
cd C:/Users/KAKA/pixelhd
./gradlew :tests:test --tests "mindustry.game.pixelhd.PixelHDMissionLoaderTest.*"
```

期望：
- `PixelHDMissionLoaderTest.isPixelHDMission_falseByDefault` 通过；
- `PixelHDMissionLoaderTest.destroyObjective_unlocksExtractionAfterTargetDestroyed` 通过；
- 现有 `ApplicationTests` 相关测试保持绿灯（保证 PixelHD 插桩未破坏默认模式）。

### 手动（Desktop，可选）

1. 准备一张带敌方 core 的地图（如 groundZero 变体），添加 tag：`pixelhd.mode = "destroy_0_1"`。
2. 启动 Desktop，加载该地图：
   - 查看日志中是否出现：
   - `[PixelHD] Destroy mission initialized at (x, y) block=core-... on map '...'`。
3. 在游戏中摧毁敌方 waveTeam 的 core：
   - DestroyBlockObjective 完成，`Rules.missionRuntime.extractionUnlocked` 变为 true，phase 变为 `extraction_unlocked`（可通过调试工具或临时日志查看）。

## Risks / TODO

- **MissionRuntime 生命周期与 MapObjectives.reset 的顺序**
  - MapObjectives 会在 `WorldLoadEvent` / `ResetEvent` 调用 `resetMissionRuntime`，重置 flags + runtime；
  - PixelHD 初始化当前在 `World.loadMap` 里执行，位于 SaveIO.load 之后、WorldLoadEvent 之前，Destroy 垂直切片测试通过，但后续增加更多 PixelHD 初始化逻辑时，需要重新审视顺序，避免与 reset 行为竞态。

- **Destroy 目标目前硬编码为 waveTeam 第一个 core**
  - 有利于快速起步，但不适合复杂任务/地图：
    - 未来需要从 map tags 或独立任务配置里加载目标列表；
    - 支持多目标 / 任意 block 类型（利用 DestroyBlocksObjective）。

- **撤离阶段当前仅实现 extraction_unlocked**
  - 尚未实现：提取具体 Extraction 区域（ExtractObjective）、玩家到达/等待逻辑，以及撤离成功 → `mission_success` → GameOverEvent；
  - 下一步需要在此基础上设计和实现 Evac 逻辑，并补充相应测试。

- **测试依赖 ApplicationTests 全局环境**
  - 集成测试基于 `ApplicationTests.launchApplication(false)`，运行时间相对普通 UT 更长，但换来了与真实运行环境高度一致的语义；
  - 若后续测试规模继续增长，可能需要拆出更多纯逻辑层 UT，减少对完整引擎初始化的依赖。