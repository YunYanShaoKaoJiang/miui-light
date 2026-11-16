# 待验证清单

按「验证价值 / 成本」排序。每条都写明方法，可直接执行。

## Q1　HAL 何时写 `hwen`

假设：`color==0` → `hwen=0`；`color!=0` → `hwen=1`。

**验证方法**：用真实应用触发背灯场景（见 Q2），同时以 50 ms 粒度采样 `hwen`。
**预期**：场景开始出现 `hwen 0→1`，场景结束出现 `1→0`。

## Q2　哪些事件真正触发背灯

已排除：`settings_strip_light_enable` 变更、shell 通知。
**待测**：真实应用通知（需在设置里勾选）、来电、充电、游戏、音乐律动。

**验证方法**：在设置页勾选一个常用应用，让该应用发通知，同时采样 sysfs；或播放音乐测试 id 19 的音乐灯。

## Q3　`period` / `repeat` / `run` 的语义

假设：HAL 用这三个实现闪烁与呼吸，`period` 为周期（ms），`repeat` 为次数，`run` 为起动开关。

**验证方法**：在受控状态下写 `period=500`、`repeat=3`、`run=1`，观察灯的节奏；**注意会锁定，需准备重启**（见 EXPERIMENTS E4）。

## Q4　`effect` 与 HAL 路径的关系

HAL 不写 `effect`，但 `effect` 读回值会随 MIUI 动作变化（曾观察到 `breath_leds_on`）。
**待测**：`effect` 的 `current cfg` 是否由驱动的 `run/period/repeat` 反推得出。

**验证方法**：写 `period/repeat/run` 后立即读 `effect`，比对 cfg 名。

## Q5　单色机型上 `color` 的作用

假设：`color` 只用来表达亮灭与亮度权重，色值被忽略。

**验证方法**：写不同 `color`（如 `0xff0000`、`0x00ff00`），同时观察 `brightness` 与肉眼亮度；若亮度随 color 的 alpha 变化则假设成立。

## Q6　能否通过 `reg` 直接控芯片

`reg` 可读写，但语义未知，写错可能导致芯片进入异常状态。
**风险高，暂不执行**；若要做，先完整备份 `reg` 快照。

## Q7　`miui.lights.ILightsManager` 的第三方可用性

`miui-framework.jar` 中有该 AIDL，`HyperLightsService` 声明权限 `com.miui.permission.ACCESS_SET_LIGHTS`。
**待验证**：该权限的 protectionLevel、第三方应用能否通过系统签名外的途径获得、`setCustomLight` 是否受 `isLedStripEnable()` 门控。

**验证方法**：反编译 `miui-framework.jar` 的 `ILightsManager$Stub` 取事务码与方法签名，再从普通应用侧尝试调用（预期被权限拒绝，用于确认门禁位置）。
