# Task: P3-extraction-and-stratagems

## Summary

- 在 Destroy 任务 0.1 垂直切片基础上，补齐了：
  - **撤离成功 → mission_success → GameOver** 的最小闭环；
  - 一个极简的 PixelHD 支援 API（PixelHDStratagems），供 HUD 调用，提供可见的支援效果。
- 现在单局 Destroy 流程在 PixelHD 模式下是：
  - Destroy 目标核心 → 解锁撤离（extraction_unlocked）→ 进入撤离区域 → 任务成功（mission_success）→ GameOverEvent(defaultTeam)。

## Changed Files

### 1. 撤离成功链路

- `core/src/mindustry/game/MapObjectives.java`
  - **修改 `ExtractObjective.done()`：**

    ```java
    @Override
    public void done(){
        if(successOnComplete){
            state.rules.objectiveFlags.add(Rules.MissionRuntime.flagMissionSuccess);
            // 新增：撤离成功即立即 GameOver，胜方为默认队伍
            Events.fire(new GameOverEvent(state.rules.defaultTeam));
        }
        super.done();
    }
    ```

  - 含义：任何使用 `ExtractObjective` 且 `successOnComplete=true` 的任务，在撤离完成时：
    - 写入 `mission_success` flag；
    - 触发 `GameOverEvent(defaultTeam)`，交由 `Control` 现有逻辑收尾。

- `core/src/mindustry/game/pixelhd/PixelHDMissionLoader.java`
  - 在 Destroy 目标注入之后，为 PixelHD 模式增加一个挂在 Destroy 之后的 `ExtractObjective`：

    ```java
    DestroyMissionConfig config = ...;

    MapObjectives.DestroyBlockObjective objective =
        (MapObjectives.DestroyBlockObjective) new MapObjectives.DestroyBlockObjective(
            config.targetBlock,
            config.targetX,
            config.targetY,
            rules.waveTeam
        ).flagsAdded(Rules.MissionRuntime.flagExtractionUnlocked);

    // PixelHD：在默认队伍核心周围生成撤离区域
    if (!state.teams.cores(rules.defaultTeam).isEmpty()) {
        Tile evacTile = state.teams.cores(rules.defaultTeam).first().tile;
        MapObjectives.ExtractObjective extract = new MapObjectives.ExtractObjective(
            evacTile.x,
            evacTile.y,
            10f, // 撤离圈半径（tile）
            rules.defaultTeam,
            "Reach evacuation zone",
            Rules.MissionRuntime.flagExtractionUnlocked
        );
        objective.child(extract);
    }

    rules.objectives.add(objective);
    ```

  - 行为：
    - DestroyBlockObjective 完成后，挂在其下的 ExtractObjective 才会开始生效；
    - ExtractObjective 依赖 `flagExtractionUnlocked`，Destroy 完成时写入该 flag；
    - 玩家进入默认队伍核心附近一定半径并满足条件时，撤离完成 → `mission_success` + GameOver。

### 2. 简易支援 API

- `core/src/mindustry/game/pixelhd/PixelHDStratagems.java`

  ```java
  public class PixelHDStratagems {

      /** Simple visible shockwave support. */
      public static void callSupport1(float worldX, float worldY) {
          if (world == null) return;
          Fx.spawnShockwave.at(worldX, worldY, 40f);
      }

      /** Secondary support with a smaller radius. */
      public static void callSupport2(float worldX, float worldY) {
          if (world == null) return;
          Fx.spawnShockwave.at(worldX, worldY, 20f);
      }
  }
  ```

  - 作用：
    - 为 HUD/content 提供两个静态支援入口：`callSupport1` / `callSupport2`；
    - 当前实现为在 world 中指定坐标生成不同半径的冲击波特效，作为“支援到达”的视觉反馈；
    - 不做复杂数值/逻辑，只要求调用安全、效果可见。

### 3. 测试

- `tests/src/test/java/mindustry/game/pixelhd/PixelHDMissionLoaderTest.java`

  新增测试用例：

  ```java
  @Test
  void extractObjective_setsMissionSuccessAndFiresGameOver() {
      ApplicationTests.launchApplication(false);

      Rules rules = new Rules();
      Map map = ApplicationTests.testMap.copy();
      map.tags.put("pixelhd.mode", "destroy_0_1");

      world.loadMap(map, rules);
      MapObjectives objectives = rules.objectives;
      assertNotNull(objectives);

      // 1. 先摧毁 Destroy 目标 → 解锁撤离
      CoreBlock.CoreBuild targetCore = state.teams.cores(rules.waveTeam).first();
      Tile targetTile = targetCore.tile;
      targetTile.removeNet();
      objectives.update();

      // 2. 查找 PixelHD 注入的 ExtractObjective
      MapObjectives.ExtractObjective extract = null;
      for (var obj : objectives.all) {
          if (obj instanceof MapObjectives.ExtractObjective eo) {
              extract = eo;
              break;
          }
      }
      assertNotNull(extract);

      final boolean[] gameOverFired = {false};
      Events.on(GameOverEvent.class, e -> gameOverFired[0] = true);

      // 3. 模拟撤离成功
      extract.done();
      objectives.syncMissionRuntime();

      assertNotNull(rules.missionRuntime);
      assertTrue(rules.missionRuntime.success);
      assertEquals(Rules.MissionPhase.mission_success, rules.missionRuntime.phase);
      assertTrue(gameOverFired[0]);
  }
  ```

- `tests/src/test/java/mindustry/game/pixelhd/PixelHDStratagemsTest.java`

  ```java
  @Test
  void supportCalls_doNotThrow() {
      ApplicationTests.launchApplication(false);
      assertDoesNotThrow(() -> PixelHDStratagems.callSupport1(10f, 10f));
      assertDoesNotThrow(() -> PixelHDStratagems.callSupport2(20f, 20f));
  }
  ```

  - 保证支援 API 在初始化好的 world 上调用不会抛异常。

## Verification

### 自动化

在 pixelhd 仓库根目录执行（命令按实际构建脚本调整）：

```bash
cd C:/Users/KAKA/pixelhd
./gradlew :tests:test --tests "mindustry.game.pixelhd.PixelHDMissionLoaderTest.*"
./gradlew :tests:test --tests "mindustry.game.pixelhd.PixelHDStratagemsTest.*"
```

期望：
- Destroy → extraction_unlocked 测试保持通过；
- `extractObjective_setsMissionSuccessAndFiresGameOver` 通过，证明 Destroy → Extraction → MissionSuccess → GameOver 链路在 headless 环境可重放；
- `PixelHDStratagemsTest.supportCalls_doNotThrow` 通过，证明支援 API 调用安全。

### 手动（Desktop）

1. 选择/制作一张 PixelHD 测试地图：
   - 有玩家 core（defaultTeam）和敌方 core（waveTeam）；
   - 在 map tags 或 Rules.tags 中设置：`pixelhd.mode = "destroy_0_1"`。

2. 启动 Desktop 版本并加载该地图：
   - 在日志中确认 Destroy 任务初始化：
     - `[PixelHD] Destroy mission initialized at (x, y) block=core-... on map '...'`。

3. 游戏内验证 Destroy → Extraction：
   - 摧毁敌方 waveTeam core：
     - 观察 HUD/MissionRuntime：
       - `phase` 从 `objective_active` 切到 `extraction_unlocked`；
       - 地图上应存在默认队伍 core 附近的撤离区域（具体呈现由 HUD/content 决定）。

4. 撤离成功：
   - 操作玩家进入撤离区域并满足 ExtractObjective 条件（根据 content/HUD 对接实现，可能是停留若干时间或交互）：
   - 期望行为：
     - `MissionRuntime.success == true` 且 `phase == mission_success`；
     - 游戏触发 GameOver，胜方为 defaultTeam。

5. 支援调用（配合 HUD）：
   - 在 PixelHD 模式下，通过 HUD 支援按钮触发 `PixelHDStratagems.callSupport1/2`；
   - 观察玩家附近是否出现明显的冲击波特效，确认 API 有可见效果且不会崩溃。

## Risks / TODO

- **ExtractObjective 的 GameOver 行为全局生效**
  - 当前在 `ExtractObjective.done()` 中统一触发 `GameOverEvent(defaultTeam)`：
    - 对 PixelHD 任务非常符合预期；
    - 但若原版/其他模式也使用 ExtractObjective，可能会引入“撤离完成立即 GameOver”的新语义。
  - TODO：
    - 后续可以考虑通过 Rules.tags 或 objective.flags 控制该行为仅在 PixelHD 模式生效，避免影响原有地图/模组。

- **撤离区域选择目前简化为默认队伍 core 附近**
  - 方便快速打通竖切，但不够灵活：
    - 未来任务可能需要在地图任意位置定义撤离点；
    - 多撤离点/移动撤离载具等高级需求无法满足。
  - TODO：
    - 从 map tags / mission 配置中读取撤离区域坐标 + 半径；
    - 或在编辑器中支持为 PixelHD 模式标记撤离点。

- **支援 API 目前仅有视觉效果**
  - `PixelHDStratagems` 目前只做冲击波特效，未加入真实战斗效应：
    - 没有伤害、没有 Buff、没有掉落物；
    - 仅作为 HUD 按钮“有反应”的原型。
  - TODO：
    - 为支援引入简单伤害/防御逻辑（例如在范围内对 waveTeam 敌人造成伤害）；
    - 设计基础冷却/次数限制，避免滥用。

- **测试仍依赖 ApplicationTests 完整环境**
  - Mission/支援相关测试继续依赖 `ApplicationTests.launchApplication(false)`：
    - 优点：高度贴近真实运行环境；
    - 风险：tests 模块运行时间略长，对 CI 时长有一定影响。
  - TODO：
    - 将 Mission/Stratagem 的部分逻辑抽到纯逻辑层，增加不依赖完整引擎初始化的 UT，以缩短回归时间。
