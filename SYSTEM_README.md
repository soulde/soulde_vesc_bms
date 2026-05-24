# VESC BMS 固件系统说明

本文从代码角度说明本仓库固件的整体结构、启动流程和主要模块职责。该项目是基于 STM32L476 与 ChibiOS 的 VESC BMS 固件，面向电池管理、充电控制、均衡、通信和固件升级。

## 1. 系统总体介绍

系统运行在 ChibiOS RTOS 上，主控为 STM32L476。固件围绕三类核心任务展开：

1. 电芯监测：通过 LTC6813 读取串联电芯电压、芯片温度和 GPIO 模拟量。
2. BMS 控制：根据配置、温度、电压、电流和 CAN 总线状态执行充电允许、充电关断、均衡控制、SoC/电量计数等逻辑。
3. 对外通信：通过 CAN、USB CDC、可选 UART 与 VESC Tool、VESC 控制器或其他 BMS 节点交互。

全局运行状态和用户配置保存在 `backup` 结构中。该结构放在 `.ram4` 段，低功耗 Standby 时通过 SRAM4 保持；断电或关键事件时会写入 Flash 备份区。

## 2. 启动流程

入口在 `main.c`：

1. 调用 `halInit()` 和 `chSysInit()` 初始化 ChibiOS HAL 与内核。
2. 检查 `backup` 中的初始化标志；无效时从 Flash 备份区恢复。
3. 对 Ah/Wh 计数、电流零偏、CAN ID、CAN 波特率、主配置等字段设置默认值。
4. 执行 `HW_INIT_HOOK()`，并通过 `conf_general_apply_hw_limits()` 套用硬件限制。
5. 初始化命令处理、USB、电源 ADC、LTC6813、BMS 逻辑、CAN、可选 UART、睡眠、环境传感器、看门狗和自检。
6. 发送 CAN 启动通知帧。
7. 主循环每 1 ms 同步部分配置到 `backup`，实际业务由各线程完成。

## 3. 线程组成

系统采用多线程分工：

| 线程 | 来源文件 | 作用 |
| --- | --- | --- |
| `LTC` | `drivers/ltc6813.c` | 周期读取电芯电压、GPIO、电池包电压和芯片温度，并写入均衡开关状态。 |
| `ADC` | `pwr.c` | 读取 MCU ADC，包括充电口电压、熔丝电压、输入电流和 NTC 温度。 |
| `IfThd` | `bms_if.c` | 滤波电流、计算 Ah/Wh、估算 SoC、更新 LED 状态。 |
| `Charge` | `bms_if.c` | 判断充电条件，控制充电 MOS/BQ 输出，处理过温、过流和充电器断开。 |
| `Balance` | `bms_if.c` | 根据电芯压差、均衡模式、温度和最大均衡通道数控制 LTC 放电均衡。 |
| `CAN read/process/status` | `comm_can.c` | 接收 CAN 帧、解析协议、周期广播 BMS 状态。 |
| `USB read/process` | `comm_usb.c` | 处理 USB CDC 字节流，并交给 packet/commands 协议层。 |
| `comm_uart` | `comm_uart.c` | 在硬件启用 UART 时处理串口协议。 |
| `Sleep` | `sleep.c` | 管理休眠倒计时、绿色 LED 状态和 Standby 进入流程。 |
| `Timeout` | `timeout.c` | 监控关键线程喂狗情况，驱动独立看门狗。 |

## 4. 核心模块说明

### 4.1 BMS 业务模块：`bms_if.c`

这是系统的业务核心。主要职责包括：

- 维护输入电流滤波值：`m_i_in_filter` 和 `m_i_in_filter_ic`。
- 根据电芯电压和温度判断 `bms_if_charge_ok()`。
- 控制充电开关：满足充电口电压、温度、电压、电流和远端 CAN BMS 状态时允许充电。
- 控制均衡：支持禁用、仅充电时、充电中和充电后、始终均衡等模式。
- 支持分布式均衡：当 `dist_bal` 启用时，会参考 CAN 上其他 BMS 的最低电芯电压。
- 统计 Ah/Wh 充放电量，并在电流超过配置阈值时阻止睡眠。
- 提供上层读取接口，如总电压、单体电压、温度、SoC、充电状态、均衡状态等。

### 4.2 电芯采样与均衡驱动：`drivers/ltc6813.c`

该模块通过软件 SPI 与 LTC6813 通信：

- 周期配置 LTC6813 寄存器。
- 读取 18 路电芯电压。
- 读取 GPIO 辅助电压和芯片温度。
- 根据 `ltc_set_dsc()` 设置的状态控制对应电芯放电均衡。
- 提供 `ltc_self_test()` 用于 LTC 内部自检。

### 4.3 电源与 MCU ADC：`pwr.c`

该模块负责主控 ADC 侧的模拟量：

- 充电口电压 `pwr_get_vcharge()`。
- 熔丝/电源侧电压 `pwr_get_vfuse()`。
- 输入电流 `pwr_get_iin()`。
- 多路 NTC 温度 `pwr_get_temp()`。
- 初始化温度测量使能、当前测量使能、CAN 电源使能和可选蜂鸣器 PWM。

### 4.4 通信协议层：`commands.c`

`commands.c` 是 USB、CAN、UART 共用的命令处理中心。它根据 `COMM_*` 命令执行操作，包括：

- 获取固件版本、硬件名、UUID 和固件信息。
- 获取 BMS 实时值、电池类型、扩展温湿度。
- 修改充电允许、均衡覆盖、重置计数、强制均衡。
- 读写自定义配置和压缩 XML 配置。
- 执行固件升级相关操作：擦除新 App、写入新 App、跳转 Bootloader。
- 处理终端命令、Blackmagic 调试转发、自检和 CAN 波特率更新。

### 4.5 CAN 通信：`comm_can.c`

CAN 模块同时承担接收、转发和周期广播：

- `cancom_read_thread` 从 CAN 外设读取帧并放入环形缓冲。
- `cancom_process_thread` 解析扩展帧或调用标准帧回调。
- `cancom_status_thread` 周期发送 BMS 状态，包括总压、充电电压、电流、Ah/Wh、单体电压、均衡位图、温度、湿度、SoC/SoH/状态位。
- 支持 VESC 状态帧缓存、其他 BMS 状态缓存、CAN packet 分片组包、Ping/Pong、远程命令转发和 CAN 波特率切换。

### 4.6 USB 与 UART 通信：`comm_usb.c`、`comm_uart.c`

USB CDC 和 UART 都使用 `packet.c` 做字节流封包/解包，解出的完整命令包交给 `commands_process_packet()`。USB 使用两个线程分别读串口数据和处理环形缓冲；UART 在硬件定义 `HW_UART_DEV` 时编译启用。

### 4.7 配置系统：`conf_general.*`、`config/`

`conf_general.h` 选择默认硬件配置，当前默认包含 `hw_lb.h` 和 `hw_lb.c`。配置默认值来自 `config/conf_default.h`，配置结构与序列化由 `config/confparser.*` 管理，VESC Tool 使用的配置描述 XML 来自 `config/settings.xml`，并通过 `config/confxml.*` 编译进固件。

### 4.8 硬件抽象层：`hwconf/`

`hwconf/` 中每套硬件提供引脚、ADC 通道、传感器数量、电阻分压、MOS 控制、温度宏和 Hook。公共入口是 `hwconf/hw.h`，它通过 `HW_HEADER` 包含具体硬件头文件，并为未定义的硬件能力提供默认宏。这样同一套业务代码可以适配多种 BMS 硬件。

### 4.9 低功耗管理：`sleep.c`

系统通过 `sleep_reset()` 延长唤醒时间。充电、均衡、CAN/USB 活动、电流超过阈值等事件会重置休眠倒计时。倒计时归零后：

- 关闭电流测量、温度测量、CAN、电源输出和 LED。
- 调用 `bms_if_sleep()` 与 `HW_SLEEP_HOOK()`。
- 配置 RTC 周期唤醒。
- 设置 SRAM4 保持并进入 Standby。

### 4.10 Flash 与 Bootloader：`flash_helper.c`

Flash 布局按固定地址划分：

- `0x08000000`：主程序。
- `0x0801E000`：参数备份区。
- `0x08020000`：新固件暂存区。
- `0x0803E000`：Bootloader。

模块提供备份数据擦写、固件写入、Bootloader 擦除和跳转 Bootloader 功能。长时间 Flash 操作前会把看门狗调到最慢。

### 4.11 看门狗与故障保护：`timeout.c`

系统使用独立看门狗 IWDG。`Timeout` 线程每 100 ms 检查关键线程是否喂狗，目前监控均衡、CAN、睡眠线程。若关键线程未运行，看门狗不会被刷新，MCU 会复位。

### 4.12 工具与辅助模块

- `buffer.c`：二进制协议的整数、浮点打包/解包。
- `packet.c`：通信帧封包、CRC 和流式解析。
- `crc.c`：CRC 计算。
- `utils.c`：滤波、映射、电池容量估算等通用函数。
- `terminal.c`：终端命令处理。
- `mempools.c`：简单内存池。
- `selftest.c`：BMS 自检入口。
- `blackmagic/`：可选 Blackmagic 调试/烧录支持。
- `compression/`：LZO 压缩支持，用于配置 XML 或烧录数据。

## 5. 数据流概览

```text
LTC6813 / MCU ADC / 环境传感器
        ↓
ltc6813.c / pwr.c / hdc1080,sht30,shtc3,bme280_if
        ↓
bms_if.c 计算充电、均衡、电流、SoC、Ah/Wh、状态
        ↓
comm_can.c 周期广播状态
commands.c 响应 USB/CAN/UART 请求
        ↓
VESC Tool / VESC 控制器 / 其他 BMS 节点
```

## 6. 构建与硬件选择

默认构建：

```bash
make
```

指定硬件构建：

```bash
make HW_SRC=hw_lb.c HW_HEADER=hw_lb.h
```

烧录：

```bash
make upload
```

调试 OpenOCD 服务：

```bash
make server
```

构建依赖 `arm-none-eabi-gcc`、ChibiOS Makefile 规则和 OpenOCD。新增硬件时通常需要在 `hwconf/` 添加 `.c/.h`，并通过 `HW_SRC`、`HW_HEADER` 或 `conf_general.h` 选择。

## 7. 维护注意事项

- 修改充电、均衡、电压、电流、温度阈值时，应同时检查 `config/settings.xml`、`conf_default.h` 和对应 `hwconf/` 限制。
- 修改 CAN 状态帧时要保持 VESC Tool 和 VESC 固件端协议兼容。
- Flash 布局、Bootloader 地址和备份区地址属于高风险区域，修改前必须确认目标芯片容量和启动流程。
- 新增线程建议接入 `timeout.c` 的喂狗监控，避免死锁后系统继续运行。
- 涉及电池安全的改动必须在目标硬件上验证充电断开、均衡温升、过流和过温路径。
