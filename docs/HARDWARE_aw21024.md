# aw21024 硬件层

## sysfs 节点

`/sys/class/leds/aw21024_led/`，属主 `system:system`，root 可写，SELinux 不拦截。

| 属性 | 权限 | 读回 | 实测写入耗时 | 备注 |
|---|---|---|---|---|
| `hwen` | rw | `hwen=0/1` | — | **硬件总使能**，0 时灯全黑 |
| `brightness` | rw | 0–255 | < 1 ms | 实际亮度 |
| `effect` | rw | 全量 cfg 表 + `current cfg` | 0.2~1.2 s | 慢操作 |
| `color` | w | I/O error | — | 单色机型无可见效果 |
| `rgbcolor` | w | I/O error | — | 同上 |
| `run` | w | I/O error | — | 图案引擎，禁用 |
| `period` | w | I/O error | — | 图案引擎，禁用 |
| `repeat` | w | I/O error | — | 图案引擎，禁用 |
| `reg` | rw | 寄存器快照 | — | 如 `reg:0x00=0x91` |
| `open_detect` | r | 开路检测 | — | 本机正常 |
| `short_detect` | r | 短路检测 | — | 本机正常 |
| `max_brightness` | r | 255 | — | — |

### effect 取值

```
0  aw21024_cfg_led_off       全灭
1  aw21024_all_leds_on       全亮
2  aw21024_left_leds_on      左半亮
3  aw21024_left_leds_off     左半灭
4  aw21024_right_leds_on     右半亮
5  aw21024_right_leds_off    右半灭
6  aw21024_breath_leds_on    硬件呼吸
```

写入用序号（`echo 1 > effect`），写配置名无效。

## 实测数据

写入延迟（连续写入 5~10 次取平均）：

```
effect=6    1136 ms / 5 次   ≈ 227 ms 每次
effect=0    6233 ms / 5 次   ≈ 1246 ms 每次
brightness     1 ms / 10 次  ≈ 0.1 ms 每次
```

结论：动画只能用 `brightness`，`effect` 仅用于状态切换。

寄存器快照（熄灭状态）：

```
reg:0x00=0x91   ← 芯片 ID
reg:0x01=0x00 … reg:0x09=0x00
```

## 禁用项

写入 `effect=6`、`run`、`period`、`repeat` 会启动 AW21024 内部图案引擎，此后：

- 所有 `effect` 写入在约 100 ms 内被回退为 `breath_leds_on`
- 该配置在本机型不产生可见亮度
- 停用户态进程、停 `HyperLightsService`、`kill -9` lights HAL、解绑/重绑 i2c 驱动均无法恢复
- 只有断电（重启设备）能复位

## 已验证的最小可用序列

```sh
L=/sys/class/leds/aw21024_led
echo 1 > $L/hwen        # 必须
echo 1 > $L/effect
echo 150 > $L/brightness
```

结果：灯常亮，肉眼确认。

熄灭：

```sh
echo 0 > $L/effect
```

`hwen` 保持 1 不影响熄灭效果；若要让芯片彻底关闭，另写 `hwen=0`（MIUI 会这么做）。

## 待确认

- `hwen` 是否为电源使能、写 0/1 的时序是否有最小间隔要求
- `brightness` 与 `color` 的寄存器映射关系（单色机型下 `color` 是否真的被忽略）
- `reg` 写入是否可用来直接控制通道（尚未尝试，风险未知）
