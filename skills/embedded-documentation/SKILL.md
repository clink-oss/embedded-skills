---
name: embedded-documentation
description: >
  用于创建或评审 STM32、FreeRTOS、无人机、机器人等嵌入式固件项目，
  确保文档完整、一致且便于维护。自动生成并检查 README、架构文档、模块设计、接口规范、
  RTOS 任务设计、ISR 文档、状态机、硬件引脚映射和数据流图。
  在新建项目、模块、RTOS 任务或 ISR，发现状态机或未编写文档的组件时使用。
---

# 嵌入式固件文档

## 目的

本技能定义嵌入式固件项目的文档规范，确保生成的每个项目都具有足够的说明，使其他工程师能够理解、构建、修改和维护系统。

目标：

- 可维护性
- 可追溯性
- 新成员上手效率
- 架构清晰可见
- 设计一致性

最有价值的嵌入式文档并非 Doxygen API 注释，而是能让新工程师在几分钟内了解整个系统的架构概览、RTOS 任务设计和硬件资源映射。

文档必须随代码同步更新，公开模块不得缺少文档。

## 文档层级概览

| 层级 | 内容 | 生成时机 |
|------|------|----------|
| L1 — 项目 | README、架构、构建、硬件、目录结构 | 新建项目时 |
| L2 — 模块 | 各模块的设计文档 | 新建模块时 |
| L3 — 接口 | 接口契约文档 | 新建接口时 |
| L4 — RTOS/ISR | TaskDesign、ISRDesign、EventFlow | 检测到 `osThreadNew` 或 `IRQHandler` 时 |
| L5 — 控制 | ControlLoop、StateMachine、Estimator | 检测到状态机枚举或控制结构时 |

## 项目级文档（L1）

创建嵌入式项目时，生成：

```text
README.md

docs/
├── Architecture.md
├── Build.md
├── Hardware.md
└── DirectoryStructure.md
```

### README 要求

每个项目的 README 必须包含：

```markdown
# 项目概览

说明固件的功能及目标硬件。

# 功能特性

主要能力和支持的外设。

# 硬件平台

MCU 系列、开发板版本、关键引脚分配。

# 构建说明

工具链、构建命令和可选功能。

# 烧录说明

调试器、调试探针或引导程序的使用方式。

# 项目结构

目录布局概览。

# 软件架构

分层图及依赖方向。

# 依赖项

需要的中间件、SDK 和工具链。

# 许可证
```

### 架构文档

必须使用 ASCII 依赖图说明分层模型：

```text
app/
 ↓
module/
 ↓
interface/
 ↓
bsp/
 ↓
platform/
```

解释每层的作用以及依赖方向的限制。

### 目录结构文档

根据实际项目目录自动生成：

```text
project/
├── app/              — 应用入口、任务、状态机
├── module/           — 业务逻辑模块
├── interface/        — 硬件抽象接口契约
├── bsp/              — 板级驱动实现
├── platform/         — 厂商 SDK、HAL、CMSIS、链接脚本
├── middleware/       — 第三方组件（FreeRTOS、lwIP、FatFS）
├── docs/             — 文档
├── tools/            — 构建和调试脚本
└── tests/            — 单元测试和集成测试
```

### 硬件文档

项目涉及 GPIO、UART、SPI、I2C、CAN、PWM 或 ADC 时，生成 `docs/Hardware.md`，并包含资源表：

```markdown
| 外设 | 接口 | 连接对象 | 用途 |
|------|------|----------|------|
| USART1 | uart_if | 遥测设备 | 115200 波特率 |
| SPI1 | spi_if | ICM-42688 | IMU 数据，1 MHz |
| TIM3 | pwm_if | 电机驱动器 | 50 Hz PWM |
| ADC1 | | 电池 | 电压监测 |
| PB0 | gpio_if | LED | 状态指示 |
```

## 模块级文档（L2）

创建新模块（如 `module/motor/`）时，生成 `docs/modules/motor.md`：

```markdown
# 电机模块

## 目的

控制电机输出：将转速指令转换为 PWM 信号，并监测故障状态。

## 职责

- 执行转速控制环
- 通过 pwm_if 生成 PWM
- 检测故障并安全停机
- 限制电流

## 公开 API

error_t motor_init(void)
error_t motor_set_speed(int32_t rpm)
void    motor_stop(void)
bool    motor_is_faulted(void)

## 依赖项

- pwm_if
- timer_if
- gpio_if

## 数据流

control_module → motor_module → pwm_if → platform_pwm

## 状态机

STOPPED → STARTING → RUNNING → FAULT → STOPPED
```

## 接口文档（L3）

`interface/` 中的每个接口必须说明：

```markdown
# UART 接口

## 目的

为模块提供不依赖具体硬件的 UART 访问能力。

## 公开 API

int32_t uart_init(uart_config_t *cfg)
int32_t uart_write(const uint8_t *data, uint32_t len)
int32_t uart_read(uint8_t *data, uint32_t len)

## 实现方与使用方

由 BSP 层实现，供 module/ 层使用。

## 线程安全

不保证线程安全，调用者必须使用互斥锁或临界区保护。
```

## RTOS 任务文档（L4）

代码包含 `osThreadNew()` 或 `xTaskCreate()` 时，生成 `docs/TaskDesign.md`：

```markdown
# 任务设计

| 任务 | 周期 | 优先级 | 栈大小 | 职责 |
|------|------|--------|--------|------|
| imu_task | 1 ms | 高 | 512 | 传感器采集 |
| control_task | 1 ms | 高 | 1024 | PID 控制环 |
| telemetry_task | 20 ms | 低 | 256 | UART 遥测输出 |
| usb_cdc_task | 事件触发 | 中 | 512 | USB 串口处理 |
```

同时记录同步机制：

```markdown
### 同步机制

| 资源 | 类型 | 用途 |
|------|------|------|
| imu_data_queue | 消息队列 | IMU → 控制任务 |
| control_mutex | 互斥锁 | 共享设定值访问 |
| telemetry_event | 事件标志 | 触发遥测 |
```

## ISR 文档（L4）

代码包含 `IRQHandler` 函数时，生成 `docs/ISRDesign.md`：

```markdown
# ISR 设计

| 中断 | 来源 | 优先级 | 执行时间 | 共享变量 |
|------|------|--------|----------|----------|
| USART1_IRQn | 接收 | 5 | < 5 µs | rx_buffer、rx_index |
| TIM3_IRQn | 更新事件 | 4 | < 2 µs | pwm_duty |

### 延后处理

| ISR | 延后处理机制 | 处理任务 |
|-----|--------------|----------|
| USART1_IRQn | osMessageQueuePut | usb_cdc_task |
| TIM3_IRQn | 直接在 ISR 中更新 | — |
```

## 状态机文档（L5）

代码包含状态枚举或 `switch(state)` 时，生成 `docs/StateMachine.md`：

```text
INIT
 ↓
IDLE
 ↓
RUNNING
 ↓
FAULT → IDLE
```

应包含状态转换条件：

```markdown
| 转换 | 起始状态 | 目标状态 | 条件 |
|------|----------|----------|------|
| start | IDLE | RUNNING | system_ready == true |
| fault | RUNNING | FAULT | motor_fault_detected |
| reset | FAULT | IDLE | 收到 fault_ack |
| error | INIT | IDLE | 初始化完成 |
```

## 图示生成规则

编写文档时，架构图和数据流图优先采用 ASCII 图示，以便任何 Markdown 阅读器都能直接显示，无需外部工具。

### 架构图

```text
app/
 ↕
module/
 ↕
interface/
 ↕
bsp/
 ↕
platform/
```

### 数据流图

```text
imu_sensor
    ↓（SPI）
imu_module
    ↓（队列）
estimator
    ↓（共享数据）
controller
    ↓（pwm_if）
motor_driver
```

### 任务交互图

```text
imu_task ──队列──→ control_task ──队列──→ telemetry_task
```

## 文档检查清单

生成代码后，应确认：

- [ ] README 存在，且包含所有必需章节
- [ ] 架构文档描述了分层模型
- [ ] 已说明构建方法
- [ ] 硬件资源已通过外设表记录
- [ ] 每个新模块都有设计文档
- [ ] 所有接口都说明了用途、API、实现方与使用方、线程安全性
- [ ] RTOS 任务已记录周期、优先级、栈大小和职责
- [ ] ISR 已记录中断来源、延迟、共享变量和延后处理方式
- [ ] 状态机已记录状态转换及条件
- [ ] 已记录依赖图和数据流
- [ ] 已说明目录结构

## 自动修正策略

缺少文档时：

1. **识别**未编写文档的组件，包括模块、接口、任务、ISR 和硬件
2. 按上述模板**生成**所需文档
3. 在 README 目录中添加新文档的**链接**
4. **更新**架构文档中的引用，保持文档之间的关联一致

文档应随代码同步更新，公开模块不得缺少文档。
