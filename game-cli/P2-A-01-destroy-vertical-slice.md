# P2-A-01 Destroy Mission 0.1 垂直切片报告

## 1. 任务概览

- 调度令：P2-A-01 / 代码令 #CODE-G-02, #CODE-G-03
- 目标：在不破坏原版 Mindustry 的前提下，为 PixelHD 模式实现 Destroy 任务 + 撤离解锁的 0.1 垂直切片（单人本地），并补上基础自动化测试。
- 当前状态：
  - PixelHD Destroy 任务已能在带 `pixelhd.mode = "destroy_0_1"` 的地图上自动挂载一个 DestroyBlockObjective，以敌方 waveTeam 的第一个 core 为目标；
  - 摧毁该 core 后，会通过 MapObjectives/MissionRuntime 自动切换到 `extraction_unlocked` 阶段；
  - 新增集成测试覆盖“Destroy → Extraction 解锁”链路。

## 2. 新增 / 修改文件

### 2.1 新增

1. `core/src/mindustry/game/pixelhd/DestroyMissionConfig.java`
   - 作用：
     - 定义 Destroy 任务的配置数据结构（id、目标 tile 坐标、目标 block、可选时间限制）。
     - 当前版本主要作为 PixelHDMissionLoader 构造 Destroy 目标的容器，便于后续扩展和编辑器集成。

2. `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java`
   - 作用：
     - 检测当前 map / rules 是否是 PixelHD Destroy 模式（通过 `pixelhd.mode == "destroy_0_1"`）。
     - 在 WaveTeam 有 core 时，选取第一个 core 的 tile 作为 Destroy 目标。
     - 构造 `DestroyMissionConfig` 并创建一个 `MapObjectives.DestroyBlockObjective`：
       - 目标为 `targetBlock @ (targetX, targetY)`，team = `rules.waveTeam`。
       - 通过 `flagsAdded(Rules.MissionRuntime.flagExtractionUnlocked)` 在完成时自动添加 extraction 解锁 flag。
     - 确保 `rules.objectives` 存在并追加该 objective。
     - 确保 `rules.missionRuntime` 非 null，生命周期交由 MapObjectives.reset/sync 管理（不再直接写 missionStarted flag）。

3. `tests/src/test/java/mindustry/game/pixelhd/PixelHDMissionLoaderTest.java`
   - 作用：
     - 校验 PixelHD Destroy 任务注入与阶段切换逻辑。
   - 主要测试用例：
     1. `isPixelHDMission_falseByDefault`
        - 使用 `ApplicationTests.launchApplication(false)` 初始化环境；
        - 验证默认 `ApplicationTests.testMap` 未打 PixelHD tag 时 `isPixelHDMission` 为 false，避免误触发 PixelHD 逻辑。
     2. `destroyObjective_unlocksExtractionAfterTargetDestroyed`
        - 初始化测试环境后：
          - `Rules rules = new Rules();`
          - `Map map = ApplicationTests.testMap.copy();`
          - `map.tags.put("pixelhd.mode", "destroy_0_1");`
          - `world.loadMap(map, rules);` → 触发 `PixelHDMissionLoader.initPixelHDMission(...)`。
        - 断言：
          - `rules.objectives != null` 且首个 objective 为 `DestroyBlockObjective`；
          - 敌方 `rules.waveTeam` 至少有一个 core；
        - 模拟摧毁目标：
          - `CoreBlock.CoreBuild targetCore = state.teams.cores(rules.waveTeam).first();`
          - `Tile targetTile = targetCore.tile;`
          - `targetTile.removeNet();`（移除 building + block，等价于目标被摧毁）；
        - 推进 MapObjectives 状态机：
          - `rules.objectives.update();`
        - 最终断言：
          - `rules.missionRuntime != null`；
          - `rules.missionRuntime.extractionUnlocked == true`；
          - `rules.missionRuntime.phase == Rules.MissionPhase.extraction_unlocked`。

### 2.2 修改

1. `core/src/mindustry/core/World.java`
   - 方法：`loadMap(Map map, Rules checkRules)`
   - 改动：在原有 map 合法性检查之后、`invalidMap` 分支之前插入 PixelHD 初始化调用：

   ```java
   state.map = map;
   invalidMap = false;

   // 原有非 headless / headless map 校验逻辑...

   if (!invalidMap) {
       mindustry.game.pixelhd.PixelHDMissionLoader.initPixelHDMission(state, state.rules, this, map);
   }

   if (invalidMap) Core.app.post(() -> state.set(State.menu));
   ```

   - 行为保证：
     - 仅在 map 合法 (`invalidMap == false`) 且 PixelHD 模式 tag 命中时才进行 Destroy 任务注入；
     - 非 PixelHD 地图（默认）完全保持原有行为。

2. `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java`（二次修改）
   - 移除对 `Rules.MissionRuntime.flagMissionStarted` 的硬写：

   ```java
   if (rules.missionRuntime == null) {
       rules.missionRuntime = new Rules.MissionRuntime();
   }
   // 原先的 rules.objectiveFlags.add(flagMissionStarted) 已删除
   ```

   - 由 `MapObjectives.resetMissionRuntime(...)` + `syncMissionRuntime()` 统一负责：
     - 根据是否存在 `qualified()` 的 objective 和当前 flags 推导 `missionStarted/objectiveActive/phase`；
     - 避免与 WorldLoadEvent 时的 reset 逻辑打架。

## 3. 验证方法

### 3.1 自动化测试

在项目根目录下执行 tests 模块测试（命令以实际构建脚本为准，可类似）：

```bash
cd C:/Users/KAKA/pixelhd
./gradlew :tests:test --tests "mindustry.game.pixelhd.PixelHDMissionLoaderTest.*"
```

期望结果：
- `PixelHDMissionLoaderTest.isPixelHDMission_falseByDefault` 通过；
- `PixelHDMissionLoaderTest.destroyObjective_unlocksExtractionAfterTargetDestroyed` 通过；
- 现有 `ApplicationTests` 系列测试保持绿灯（确保 PixelHD 插桩未破坏默认流程）。

### 3.2 手动 Desktop 验证（可选）

1. 准备一张测试地图（有敌方 core）：
   - 复制一张现有地图（例如 groundZero 的变体），命名为 `pixelhd_destroy_test`；
   - 在 map 的 tags 或加载流程中设置：`pixelhd.mode = "destroy_0_1"`。

2. 启动 Desktop 版本并加载该地图：
   - 观察日志中是否出现：

   ```
   [PixelHD] Destroy mission initialized at (x, y) block=core-... on map 'pixelhd_destroy_test'.
   ```

3. 正常游玩并摧毁敌方 waveTeam 的 core：
   - DestroyBlockObjective 的 `update()` 应完成，触发 flagsAdded(extraction_unlocked)；
   - 下一帧后 `Rules.missionRuntime.extractionUnlocked` 应为 true，phase 为 `extraction_unlocked`。

## 4. 风险 / TODO

- **Objective 生命周期与 MissionRuntime 重置顺序**
  - MapObjectives 会在 `WorldLoadEvent` / `ResetEvent` 时调用 `resetMissionRuntime`，清空所有 mission flags 并重置 `rules.missionRuntime`。
  - 当前 PixelHD 初始化逻辑位于 `World.loadMap` 内部，先于 WorldLoadEvent 的触发（`endMapLoad` 内 fire）。
  - 目前 Destroy 垂直切片的测试表明顺序是安全的，但后续增加更多 PixelHD 初始化（尤其依赖 WorldLoadEvent 时）需要再审视生命周期，必要时集中改到统一 hook 上。

- **目标选择目前硬编码为 waveTeam 第一个 core**
  - 对于更复杂的任务/地图，可能需要：
    - 从 map tags 或独立 mission 配置中读取目标坐标/类型；
    - 支持多目标（DestroyBlocksObjective）。
  - 现阶段为了垂直切片，先使用简单策略，后续在 Capture / 多任务阶段再抽象。

- **撤离阶段目前仅做到 extraction_unlocked**
  - Destroy 垂直切片实现的是「摧毁目标 → `extraction_unlocked`」，还未实现：
    - 提取 Extraction 区域（ExtractObjective）；
    - 撤离成功 → `mission_success` → GameOverEvent。
  - 这些部分计划在下一阶段（Capture or Evac 扩展）逐步补齐。

- **测试依赖 ApplicationTests 环境**
  - 集成测试使用了 `ApplicationTests.launchApplication(false)`，这意味着：
    - 测试必须在能访问完整 assets 的环境下运行；
    - 测试时间略长，但换来的是高保真度（真实世界 + rules + entities）。
  - 后续如果测试执行时间成为问题，可以考虑再拆一部分逻辑到纯 logic UT。