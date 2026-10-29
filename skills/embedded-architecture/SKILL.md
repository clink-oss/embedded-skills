---
name: embedded-architecture
description: >
  用于新建嵌入式固件项目、添加模块、重构固件、评审架构或生成项目目录结构。
  规定面向 STM32、GD32、ESP32、RP2040、NXP、Nordic 等 MCU 平台的架构规则，
  以构建可维护、可移植、可测试且不依赖具体硬件的固件。
  涵盖项目结构、依赖管理、模块边界、平台抽象和架构检查。
---

# 嵌入式软件架构

## 目的

定义嵌入式固件项目必须遵循的架构规则。

目标是生成可维护、可移植、可测试且不依赖具体硬件的固件，适用于 STM32、GD32、ESP32、RP2040、NXP、Nordic 等 MCU 平台。

本技能重点关注：

- 项目结构
- 依赖管理
- 模块边界
- 平台抽象
- 代码归属
- 架构检查

## 架构原则

生成的固件必须遵循：

1. 单一职责
2. 低耦合
3. 高内聚
4. 明确的接口
5. 硬件抽象
6. 平台可移植性
7. 可测试性
8. 确定的执行行为

应用层不得直接依赖 MCU 厂商库。

业务逻辑必须独立于 STM32 HAL、CMSIS、FreeRTOS 内部实现以及外设寄存器。

## 必须采用的项目结构

固件项目必须使用以下目录布局：

```text
project/
├── app/
├── module/
├── interface/
├── bsp/
├── platform/
├── middleware/
├── docs/
├── tools/
└── tests/
```

### `app/` — 应用入口

包含：

- 状态机
- 应用流程编排
- 系统启动顺序
- 任务创建

不得包含：

- 寄存器访问
- HAL 调用
- 外设驱动

### `module/` — 业务逻辑

示例：`motor/`、`imu/`、`control/`、`telemetry/`、`navigation/`、`power/`。

模块只能依赖 `interface/`。

模块不得直接访问 HAL、CMSIS、GPIO 寄存器、UART 寄存器或 SPI 寄存器。

### `interface/` — 硬件抽象接口契约

示例：`gpio_if.h`、`uart_if.h`、`spi_if.h`、`pwm_if.h`、`timer_if.h`。

只定义接口契约，不得包含实现。

### `bsp/` — 板级实现

示例：`led`、`button`、`buzzer`、`oled`、`imu_board`。

负责将接口映射到具体硬件，可以使用平台层。

### `platform/` — 厂商相关代码

```text
platform/
└── stm32f407/
    ├── core/
    ├── hal/
    ├── cmsis/
    ├── startup/
    └── linker/
```

包含 CubeMX 生成的代码、HAL 驱动、CMSIS、启动文件和链接脚本。

不得包含业务逻辑。

### `middleware/` — 第三方中间件

示例：`freertos/`、`lwip/`、`fatfs/`、`usb/`。

应与应用逻辑隔离。

## 依赖规则

允许的依赖方向只能自上而下：

```text
APP → MODULE → INTERFACE → BSP → PLATFORM
```

禁止以下依赖：

- `APP → HAL`
- `APP → STM32_GPIO / STM32_UART`
- `MODULE → HAL`
- `MODULE → 寄存器访问`
- `MODULE → CMSIS`
- `MODULE → FreeRTOS 内部 API`
- `PLATFORM → MODULE`
- `PLATFORM → APP`
- `BSP → APP`

依赖必须始终向下，禁止循环依赖。

## 平台隔离规则

所有 MCU 厂商文件必须放在 `platform/<target>/` 中：

```text
platform/stm32f407/
platform/stm32h743/
platform/gd32f450/
platform/esp32s3/
```

不得把厂商生成的代码直接放在项目根目录。

仓库根目录不得出现 `/Core`、`/Drivers`，应将它们移入平台目录。

## 接口规则

每项硬件服务都必须通过接口对外提供。

示例：

```c
typedef struct {
    int32_t (*write)(const uint8_t *data, uint32_t len);
    int32_t (*read)(uint8_t *data, uint32_t len);
} uart_if_t;
```

模块只能通过接口交互，不得感知 UART 实例、GPIO 端口、DMA 通道或 MCU 系列。

## 模块规则

每个模块应只承担一项职责。

推荐：`motor`、`imu`、`control`、`telemetry`、`storage`。

不推荐：`system_manager`、`all_in_one`、`utility`、`misc`。

避免把各种职责集中到一个无所不包的模块中。

## RTOS 职责分离

RTOS 任务创建属于 `app/`，业务逻辑属于 `module/`。

错误做法：在同一个任务函数中混合传感器读取、控制算法、日志和通信。

正确做法：任务调用模块 API，模块调用接口，接口再调用驱动：

```text
TASK → MODULE API → INTERFACE → DRIVER
```

## 架构检查清单

接受生成的代码前，应确认：

- [ ] 项目符合规定的目录布局
- [ ] `app/` 中没有 HAL 调用
- [ ] `module/` 中没有 HAL 调用
- [ ] `platform/` 之外没有寄存器访问
- [ ] `platform/` 之外没有 CubeMX 代码
- [ ] 所有外设都通过接口抽象
- [ ] 没有循环依赖
- [ ] BSP 中没有业务逻辑
- [ ] 平台层中没有业务逻辑
- [ ] 模块职责单一
- [ ] 接口只包含声明
- [ ] 依赖方向正确

发现违反规则的情况时：

1. 报告具体问题
2. 解释其危害
3. 提出修正后的架构
4. 据此重构生成的代码

架构正确性优先于实现方便。
