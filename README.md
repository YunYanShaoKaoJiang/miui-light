# miui-light

小米（HyperOS）背部灯环控制链路的逆向与实机验证记录。

目标：搞清楚 MIUI 究竟怎样驱动这颗灯，再决定任何上层功能怎么做。

| 项 | 值 |
|---|---|
| 设备 | Xiaomi M098FE（`songyuan`） |
| 系统 | Android 17 / HyperOS 4.0.7.0.XGNCNXM |
| 灯 | 后置相机模组旁单色橙色灯环 |
| 驱动 | `aw21024_led`（Awinic AW21024，24 通道，i2c-3 @0x30） |
| HAL | `/odm/bin/hw/vendor.qti.hardware.lights.service` |
| 系统服务 | `com.android.server.lights.HyperLightsService`（system_server 内） |
| 权限 | KernelSU root；应用调用 `su` 实测 8 ms |

## 当前最重要的结论

**`hwen` 是硬件总使能。** `hwen=0` 时，写 `effect` / `brightness` / `color` 全部无效，但读回值会正常变化——表象是"写进去了、灯不亮"。

实测：`hwen=1` + `effect=1` + `brightness=150` → 灯常亮（肉眼确认）；仅 `effect`/`brightness` 而 `hwen=0` → 全黑。

MIUI 的 HAL 会在场景结束时把 `hwen` 写回 0。任何外部控制必须先确认并保持 `hwen=1`。

## 文档

| 文件 | 内容 |
|---|---|
| [`docs/HARDWARE_aw21024.md`](docs/HARDWARE_aw21024.md) | sysfs 全属性、写入延迟实测、禁用项、寄存器快照 |
| [`docs/MIUI_STACK.md`](docs/MIUI_STACK.md) | 设置键 → HyperLightsService → HAL → sysfs 的完整调用链 |
| [`docs/EXPERIMENTS.md`](docs/EXPERIMENTS.md) | 已做的实验、命令、结果（含无效实验） |
| [`docs/OPEN_QUESTIONS.md`](docs/OPEN_QUESTIONS.md) | 尚未验证的假设与验证方法 |

## 方法论

1. 先读代码（`system_ext/framework/miui-services.jar`、`miui-framework.jar`）确定调用链，再用实机采样验证
2. 采样粒度 50~70 ms，同时记录 `hwen` / `brightness` / `effect current cfg` / `reg`
3. 任何修改系统设置或状态的实验，先快照原值，实验后立即恢复
4. 结论区分「已验证」与「推断」
