# Phase 3：HUD 联动 Destroy/撤离 + 接线支援 API（content-cli）

## Summary

- 已根据 P3 指令在 HUD 中加强了 PixelHD Destroy → Extraction 流程的反馈：
  - 顶部任务栏现在根据 `MissionPhase` 显示不同的撤离文案（解锁 / 进行中 / 成功 / 失败）。
  - 在无后端字段的前提下，HUD 本地维护一个简单的撤离倒计时 `pixelhdExtractionTimeLeft`，在 `extraction_unlocked / extraction_called` 阶段递减并展示 `Xs` 提示，Mission 结束后自动清零。
- 将移动端 3 个支援按钮从“打印日志”升级为调用 `PixelHDStratagems` 静态 API，使用玩家当前坐标作为支援落点：
  - 支援一 → `callSupport1(player.x, player.y)`；
  - 支援二 → `callSupport2(player.x, player.y)`；
  - 支援三 暂时复用 `callSupport1` 作为占位。

从玩家视角：
- 当 Destroy 目标完成后进入撤离阶段，顶部 HUD 将高亮显示撤离状态与剩余秒数，压迫感更明显；
- 在手机上，右上角支援按钮现在会真正触发支援逻辑，而不仅仅是打 log。

## Changed Files

### 1. HUD 逻辑与 PixelHD 判定

- `core/src/mindustry/ui/fragments/HudFragment.java`

主要改动：

1. 新增字段（在已有 PixelHD HUD 字段之后）：
   ```java
   private float pixelhdExtractionTimeLeft = -1f;
   ```

2. PixelHD 判定 & 可见性（已存在逻辑确认无须改动接口）：
   ```java
   private boolean isPixelHDMission(){
       return state.rules != null
           && state.map != null
           && PixelHDMissionLoader.isPixelHDMission(state.rules, state.map);
   }

   private boolean isPixelHDMissionActive(){
       if(!shown) return false;
       if(!isPixelHDMission()) return false;
       var mr = state.rules.missionRuntime;
       return mr != null && mr.phase != Rules.MissionPhase.idle;
   }
   ```

   - 顶部任务栏与支援按钮的 `visible` 条件均基于 `isPixelHDMissionActive()`（支援按钮额外要求 `mobile`）。

3. 新增撤离倒计时更新方法，并在 `build` 尾部调用：
   ```java
   private void updatePixelhdExtractionTimer(){
       if(!isPixelHDMissionActive()){
           pixelhdExtractionTimeLeft = -1f;
           return;
       }
       var mr = state.rules.missionRuntime;
       if(mr == null) return;

       switch(mr.phase){
           case extraction_unlocked, extraction_called -> {
               if(pixelhdExtractionTimeLeft >= 0f){
                   pixelhdExtractionTimeLeft = Math.max(0f, pixelhdExtractionTimeLeft - Time.delta / 60f);
               }
           }
           case mission_success, mission_failed, idle -> {
               pixelhdExtractionTimeLeft = -1f;
           }
           default -> {
               // no-op
           }
       }
   }

   // build(...) 末尾：
   shooterHud.build(parent);
   blockfrag.build(parent);

   // update pixelhd extraction timer every frame
   updatePixelhdExtractionTimer();
   ```

### 2. 顶部撤离 HUD 文案 + 倒计时占位

- 在 PixelHD 顶部任务栏的 `pixelhdExtractionLabel` 生成逻辑中细化 phase：

  ```java
  pixelhdExtractionLabel = c.label(() -> {
      var mr = state.rules != null ? state.rules.missionRuntime : null;
      if(mr == null) return "";

      switch(mr.phase){
          case extraction_unlocked -> {
              if(pixelhdExtractionTimeLeft < 0f) pixelhdExtractionTimeLeft = 20f; // 占位初值
              int seconds = Mathf.ceil(pixelhdExtractionTimeLeft);
              return Core.bundle.format("pixelhd.extraction.unlocked", "撤离已解锁，前往撤离点！（{0}s）", seconds);
          }
          case extraction_called -> {
              int seconds = Mathf.ceil(Math.max(pixelhdExtractionTimeLeft, 0f));
              return Core.bundle.format("pixelhd.extraction.called", "撤离进行中，坚守至登舰！（{0}s）", seconds);
          }
          case mission_success -> {
              return Core.bundle.get("pixelhd.extraction.success", "撤离成功！");
          }
          case mission_failed -> {
              return Core.bundle.get("pixelhd.extraction.failed", "撤离失败！");
          }
          default -> {
              return "";
          }
      }
  }).style(Styles.outlineLabel).get();
  ```

- 说明：
  - `extraction_unlocked` 阶段首次进入时初始化 `pixelhdExtractionTimeLeft = 20f`（占位的 20 秒）；
  - 在 `updatePixelhdExtractionTimer()` 中随时间递减；
  - `extraction_called` 阶段继续显示剩余秒数；
  - `mission_success / mission_failed / idle` 时重置为 `-1f`，只显示成功/失败文案，不再显示秒数。

### 3. 支援按钮接线 PixelHDStratagems

- 新增导入：
  ```java
  import mindustry.game.pixelhd.PixelHDMissionLoader;
  import mindustry.game.pixelhd.PixelHDStratagems;
  ```

- 将支援按钮的点击回调从打印 log 改为调用封装方法：

  ```java
  // PixelHD mobile stratagem buttons (0.1 skeleton)
  parent.fill(t -> {
      t.name = "pixelhd-stratagem-buttons";
      t.top().right();
      t.visible(() -> mobile && isPixelHDMissionActive());

      t.table(c -> {
          pixelhdStratagemTable = c;
          c.defaults().size(dsize * 1.1f, dsize * 0.8f).pad(2f);

          ImageButtonStyle style = Styles.cleari;

          c.button("支援一", style, () -> handlePixelhdSupport1()).name("pixelhd-stratagem-1");
          c.row();
          c.button("支援二", style, () -> handlePixelhdSupport2()).name("pixelhd-stratagem-2");
          c.row();
          c.button("支援三", style, () -> handlePixelhdSupport3()).name("pixelhd-stratagem-3");
      });
  });
  ```

- 新增支援处理方法（使用玩家当前世界坐标）：

  ```java
  private void handlePixelhdSupport1(){
      if(!isPixelHDMissionActive()) return;
      if(player == null || player.unit() == null) return;
      PixelHDStratagems.callSupport1(player.x, player.y);
  }

  private void handlePixelhdSupport2(){
      if(!isPixelHDMissionActive()) return;
      if(player == null || player.unit() == null) return;
      PixelHDStratagems.callSupport2(player.x, player.y);
  }

  private void handlePixelhdSupport3(){
      if(!isPixelHDMissionActive()) return;
      if(player == null || player.unit() == null) return;
      // 当前复用 support1 作为占位
      PixelHDStratagems.callSupport1(player.x, player.y);
  }
  ```

## Verification

> 下述步骤假定 game-cli 已实现 Destroy → extraction_unlocked 的流程，以及 `PixelHDStratagems` 的最小实现（可以是空实现或简单 log）。

### 1. 构建与基本运行

```bash
cd C:/Users/KAKA/pixelhd

./gradlew pixelhdDesktopDev
./gradlew pixelhdAndroidDev
```

预期：
- 两个任务均成功，无编译错误；
- `HudFragment` 中对 `PixelHDStratagems` 的引用能正常解析。

### 2. Desktop：Destroy → Extraction HUD 验证

1. 启动 PixelHD 桌面版（Dev 任务或 IDE run 配置）。
2. 加载一张 PixelHD Destroy 测试地图（确保 `PixelHDMissionLoader.isPixelHDMission(...) == true`）。
3. 开始任务后观察顶部 HUD：
   - Destroy 未完成时：
     - 顶栏显示阶段标签（“落地”或“任务”） + 当前 Destroy 目标文案；
     - 撤离文案行为空或不显示。
   - 完成 Destroy 目标、`MissionRuntime.extractionUnlocked == true` 且 phase == `extraction_unlocked` 时：
     - 顶栏第三行显示“撤离已解锁，前往撤离点！（Xs）”，其中 X 从 20 附近的整数递减；
   - 若后续 game-cli 将 phase 切为 `extraction_called`：
     - 文案变为“撤离进行中，坚守至登舰！（Xs）”，秒数继续递减；
   - 当任务被判定成功（phase == `mission_success`）：
     - 顶栏显示“撤离成功！”，不再显示秒数；
   - 若任务失败（phase == `mission_failed`）：
     - 顶栏显示“撤离失败！”，倒计时被清空。

### 3. Android / Mobile：支援按钮触发验证

> 如无法真实部署到设备，可用模拟器 + `Vars.mobile == true` 的配置做近似验证。

1. 在移动端环境启动同一 PixelHD Destroy 任务地图。
2. 在 `mission_started` 或 `objective_active` 阶段：
   - 右上角显示三枚按钮：“支援一 / 支援二 / 支援三”。
3. 点击行为预期：
   - 玩家存活且有单位（`player != null && player.unit() != null`）：
     - 点击“支援一” → 调用 `PixelHDStratagems.callSupport1(player.x, player.y)`；
     - 点击“支援二” → 调用 `callSupport2(player.x, player.y)`；
     - 点击“支援三” → 暂时复用 `callSupport1`；
     - 具体效果由 game-cli 的实现决定（例如空投补给 / 炮塔 / 轰炸在玩家附近落下）。
   - 玩家死亡或尚未生成单位：
     - 点击不会调用 Stratagem（因为会直接 return），避免 NPE。

4. 在非 PixelHD 地图或非任务模式：
   - 顶部 PixelHD 任务栏与右上支援按钮整体隐藏，原版 HUD 行为保持不变。

## Risks / TODO

1. 撤离倒计时与真实逻辑暂时脱节
   - 当前 `pixelhdExtractionTimeLeft` 完全由 HUD 本地维护，与服务器 / MissionRuntime 真实时间轴不一定一致；
   - 这在联机模式下理论上可能出现：服务器已结束 / 延长撤离阶段，而 HUD 仍在倒计时；
   - TODO：game-cli 后续在 `MissionRuntime` 或其他共享结构中提供标准倒计时字段，由 HUD 直接读取，去掉本地模拟逻辑。

2. 支援落点简化为玩家当前位置
   - 目前 `callSupportX` 统一使用 `player.x / player.y` 作为支援落地点，意味着：
     - 不支持“投掷到远处”/“点落点”的交互；
     - 有可能在高移动速度时导致支援与预期稍有偏差。
   - TODO：
     - 与 game-cli 协商是否要改为“准星位置”或“点击事件位置”；
     - 如需支持更精细交互，可在 HUD 中加入简单落点预览步骤（点击按钮 → 出现目标圈 → 第二次确认）。

3. 顶部 HUD 信息密度与移动端体验
   - PixelHD 顶栏在 Extraction 段同时展示阶段标签 + Destroy/Capture 目标文案 + 撤离倒计时文案，可能在小屏上略显“字多”；
   - TODO：
     - 根据后续测试，看是否需要在 Extraction 阶段隐藏原目标文案，只保留撤离相关信息；
     - 或缩短默认文案长度，更多依赖图标/颜色表达状态。

4. 支援按钮数量与布局
   - 固定的三枚按钮在一些分辨率下可能与 minimap / 其他 HUD 发生局部重叠；
   - TODO：
     - 后续在 portrait 模式下可考虑压缩为 2 个直出按钮 + 1 个“更多”轮盘入口；
     - 增加简单的间距/布局调节，避免遮挡射击/准星区域。
