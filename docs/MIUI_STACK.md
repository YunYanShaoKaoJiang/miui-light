# MIUI 灯光控制链

## 分层

```
设置界面  com.android.settings.special.backstrap.*
   │  读写 Settings.Global
   ▼
HyperLightsService   (system_server，class com.android.server.lights.HyperLightsService)
   │  LightsService.LightImpl#setLightLocked(color, mode, onMS, offMS, brightnessMode)
   ▼
android.hardware.light.ILights /default
   │
   ▼
vendor.qti.hardware.lights.service（/odm/bin/hw/）
   │  直接写 sysfs
   ▼
/sys/class/leds/aw21024_led/{hwen,color,brightness,period,repeat,run,…}
```

## 灯 id

`HyperLightsService` 里的私有灯编号：

| id | 常量 | 用途 | 优先级 |
|---|---|---|---|
| 18 | `LIGHT_ID_LED_STRIP` | 背灯主体 | 2（通知） |
| 19 | `LIGHT_ID_LED_STRIP_MUSIC` | 音乐律动 | 4 |
| 20 | `LIGHT_ID_LED_STRIP_URGENT` | 紧急/游戏 | 1 |
| 3 | — | 电池灯 | 3 |
| 4 | — | 通知灯 | 2 |

另有 `MiuiLightsService` 中的 `LIGHT_ID_COLORFUL=8`、`MUSIC=9`、`PRIVACY=10`、`LED=11`，与背灯不是同一路。

## 设置键

| 键 | 取值 | 作用 |
|---|---|---|
| `settings_strip_light_enable` | 0 / 1 / 2 | 关闭 / 仅倒扣 / 常亮 |
| `lamp_effect_brightness` | 0–255 | 自定义亮度（本机 255） |
| `back_strap_app_notification` | `pkg=true;pkg2=false` | 哪些应用的通知点灯 |
| `back_strap_app_notification_color` | 颜色 | 通知颜色 |
| `back_strap_call_reminder` / `_color` | — | 来电提醒 |
| `call_strip_light_mode` | 0 | 通话灯模式 |

本机实测：`back_strap_app_notification`、`back_strap_call_reminder` 均为 `null`（未配置）。

## 亮灭判定

`LightImpl#updateLight()`

```java
if (shouldSkipUpdate()) return;
if (!isLedStripEnable()) handleDisabledCases();   // → updateState(0,…) + turnOffLightLocked()
else                     handleEnabledCases();    // → setLedStrip(lightState, styleType, false)
```

`isLedStripEnable()`（`HyperLightsService`）：

| `settings_strip_light_enable` | 条件 |
|---|---|
| 0 | 永不亮 |
| 1 | 需要倒扣（厂商传感器 type `33171037`，`values[1]==2`）且息屏、无相机/手电 |
| 2 | 除相机、手电外都亮 |

`handleDisabledCases()` 最终走到 `turnOffLightLocked()` → `setPriorityLightLocked(priority, 0, 0, 0, 0, 0)` → `realSetLightLocked(...)`，**color 为 0**。

## 优先级仲裁

`setLedStripLocked()` 按 `mPriority` 比较：

- 当前灯优先级更高且未熄灭 → 跳过
- 同优先级时按 `styleType` 与 `mlastLight` 决定是否覆盖
- 通过后调用 `realSetLightLocked(priority, color, mode, onMS, offMS, brightnessMode)`

`realSetLightLocked` 内部 `printLogIfNeeded` 会打日志：

```
I/HyperLightsService: realSetLightLocked, # mId: 18, curPriority: 2 color: fbffffff
```

`color` 非 0 表示点亮，0 表示熄灭。这条日志是外部观测 MIUI 是否在驱动背灯的最直接信号（仅状态翻转时打印）。

## 场景定义

`/product/etc/lights/` 下为 XML，字段：`colorARGB / flashMode / onMS / offMS / brightnessMode`。

| 文件 | colorARGB | flashMode |
|---|---|---|
| `lightstyle_notification.xml` | `#ffffff` | 0 |
| `lightstyle_alarm.xml` | `#ffffff` | 0 |
| `lightstyle_phone.xml` | `#ffffff` | 0 |
| `lightstyle_game.xml` | 多段 | — |
| `lightstyle_game_colorful.xml` | 多段 | — |

本机这些文件里的 `onMS/offMS` 数值很大（如 39321600），推断是占位值，实际节奏由 HAL 的 `period/repeat` 决定。

## HAL 层（关键）

从 `/odm/bin/hw/vendor.qti.hardware.lights.service` 提取的字符串显示，它直接写以下路径：

```
/sys/class/leds/aw21024_led/hwen
/sys/class/leds/aw21024_led/color
/sys/class/leds/aw21024_led/brightness
/sys/class/leds/aw21024_led/period
/sys/class/leds/aw21024_led/repeat
/sys/class/leds/aw21024_led/run
/sys/class/leds/aw21024_led/open_detect
/sys/class/leds/aw21024_led/short_detect
```

另有通用模板路径（供其他 LED）：

```
/sys/class/leds/%s/{breath,brightness,delay_off,delay_on,hw_pattern,repeat,trigger}
```

注意：HAL **不写 `effect`**。`effect` 是驱动自带的 cfg 选择器，MIUI 走的是 `color + brightness + period/repeat/run` 这套。

## 已知推论（待验证）

1. HAL 收到 `color==0` 时写 `hwen=0`，收到非 0 时写 `hwen=1` 再写 `color`/`brightness` —— 与本机观察到的"场景结束后灯全黑且 `hwen=0`"一致
2. 呼吸/闪烁由 `period` + `repeat` + `run` 组合实现，而非 `effect=6`
3. 单色机型下 `color` 仅用于表达"亮/灭 + 亮度权重"，颜色本身被忽略

见 `OPEN_QUESTIONS.md` 的验证方法。
