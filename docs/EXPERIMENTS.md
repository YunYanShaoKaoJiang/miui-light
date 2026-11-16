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
