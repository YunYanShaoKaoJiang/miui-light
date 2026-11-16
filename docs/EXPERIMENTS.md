# 实验记录

所有实验均在 root shell 下执行；涉及系统状态的实验先快照、事后恢复。

## E1　点灯最小序列

```sh
L=/sys/class/leds/aw21024_led
echo 1 > $L/hwen
echo 1 > $L/effect
echo 150 > $L/brightness
```

结果：灯常亮（肉眼确认）。`effect` 读回 `all_leds_on`，`hwen` 稳定为 1，连续 6 秒采样无漂移。

## E2　`hwen=0` 时写入的假象

清理环境后读取：

```
hwen=hwen=0
effect=cfg_led_off
```

此时写 `effect=1`、`brightness=150`，读回一致且保持——但灯不亮。

结论：**读回正常 ≠ 硬件输出**。`hwen` 是总闸，必须先确认。

## E3　写 `effect` 的耗时

```sh
S=$(date +%s%N); for i in 1 2 3 4 5; do echo 6 > $L/effect; done; E=$(date +%s%N)
```

| 写入值 | 5 次总耗时 |
|---|---|
| `effect=6` | 1136 ms |
| `effect=0` | 6233 ms |
| `brightness` ×10 | 1 ms |

结论：动画必须走 `brightness`。

## E4　图案引擎锁定（故障复现）

依次写入 `run 1`、`period 800`、`repeat 3`、`effect=6` 后：

- 任意 `effect` 值（0~6，含配置名）写入后约 100 ms 内回退为 `breath_leds_on`
- 同时 `hwen` 仍为 1
- 排除项：停止用户态进程、停止 `HyperLightsService`、`kill -9` lights HAL、解绑/重绑 i2c 驱动 `3-0030`、写 `run 0..3`、`hwen 0→1` 循环
- 唯一恢复方式：重启设备

结论：**禁止写 `effect=6` / `run` / `period` / `repeat`**。

## E5　切换系统开关时的 HAL 行为

快照 `settings_strip_light_enable=2`，改为 0 后 3 秒改回 2，期间以 70 ms 粒度采样 `hwen/brightness/effect`。

结果：采样全程 `hwen=1`、`brightness=150`、`cfg=cfg_led_off`，**无任何变化**。

结论：`settings_strip_light_enable` 的变更本身不直接写 LED；它只改变 `isLedStripEnable()` 的返回值，实际写入发生在场景（通知/来电/游戏）触发时。

## E6　shell 通知是否触发背灯

```sh
settings put global back_strap_app_notification "com.android.shell=true"   # 原值 null
cmd notification post -S bigtext -t "灯场景测试" tag_b "hello"
# 采样 4 秒
settings delete global back_strap_app_notification
```

结果：采样全程无变化（`hwen=1`、`cfg=off`）。

结论：shell 来源的通知不会进入背灯场景；需要真实应用的通知，或场景并非由通知触发。

## E7　idle 状态寄存器

```
reg:0x00=0x91   reg:0x01..0x09=0x00
```

结论：熄灭时除芯片 ID 外寄存器清零；`hwen` 与寄存器状态相互独立。

## E8　`su` 提权开销

```sh
S=$(date +%s%N); su -c true; E=$(date +%s%N)
S=$(date +%s%N); su -c 'echo 40 > /sys/class/leds/aw21024_led/brightness'; E=$(date +%s%N)
```

| 操作 | 耗时 |
|---|---|
| `su -c true` | 8 ms |
| `su -c` 写一次 sysfs | 7 ms |

结论：应用侧以 10 Hz 通过 `su` 驱动灯是可行的（约 7% 单核）。KernelSU 的授权是常驻的，无弹窗。

## E9　抓取 MIUI 点灯全过程（定时通知）

用 50 ms 粒度采样 `hwen/brightness/effect/reg`，同时 `logcat -s HyperLightsService` 落盘，覆盖定时通知触发窗口。

两次事件结果完全一致：

```
17:20:00.114  hwen=1  bri=255  cfg=cfg_led_off  reg:0x00=0x91 0x01=0x00
17:20:03.107  hwen=0  bri=255  cfg=cfg_led_off  reg:0x00=0x00 0x01=0x00
17:21:00.059  hwen=1  bri=255  cfg=cfg_led_off  reg:0x00=0x91 0x01=0x00
17:21:03.097  hwen=0  bri=255  cfg=cfg_led_off  reg:0x00=0x00 0x01=0x00

MIUI 日志：
17:20:00.056  realSetLightLocked, # mId: 4, curPriority: 2 color: ffffffff
17:20:03.059  realSetLightLocked, # mId: 4, curPriority: 2 color: 0
17:21:00.022  realSetLightLocked, # mId: 4, curPriority: 2 color: ffffffff
17:21:03.023  realSetLightLocked, # mId: 4, curPriority: 2 color: 0
```

观察：

- 走的是 **灯 id 4（通知灯）**，不是 18（背灯）
- 只写 `hwen` 与 `brightness`，**从未写 `effect`**
- 通道使能寄存器 `0x01` 全程为 `0x00`
- 持续 3 秒（与 `NOTIFICATION_DURATION_DEFAULT = 3000` 一致）
- 结束后 `hwen=0`，`reg:0x00` 由 `0x91` 变 `0x00`（芯片断电，I2C 不可读）

## E10　`effect` 是否必需（对照实验）

固定 `hwen=1`、`brightness=200`，改变 `effect`：

```
effect=0  → reg:0x01=0x00, 0xa7~0xab=0x00  → 灯不亮（肉眼确认）
effect=1  → reg:0x01=0x60, 0xa7~0xab=0xff  → 灯亮
```

结论：**`effect` 是必需的**。`brightness` 只调光，不负责使能通道。

## E11　写 `color` 能否替代 `effect`

依次写入 `0xffffff`、`16777215`、`255,255,255`、`ffffff`、`255`，`effect` 读回始终为 `cfg_led_off`，`reg:0x01` 始终 `0x00`。

结论：至少这些格式下 `color` 不会使能通道；`color` 的作用待定（可能只在通道已使能时参与调色）。
