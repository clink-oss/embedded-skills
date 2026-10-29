---
name: embedded-driver-design
description: >
  用于开发或评审 STM32、GD32、ESP32、FreeRTOS、无人机飞控和机器人等嵌入式固件的外设驱动。
  规定分层驱动架构，包括总线抽象、设备上下文对象、寄存器封装，以及总线驱动、设备驱动、
  BSP 和业务模块之间的严格职责分离。防止设备驱动直接依赖 HAL 或 MCU 寄存器，
  并将 PID、卡尔曼滤波、姿态估计等控制算法排除在驱动代码之外。
---

# 嵌入式驱动设计

## 目的

使用分层架构设计和评审嵌入式驱动。

目标是确保：

- 可在不同 MCU 平台间移植
- 可通过模拟总线接口进行测试
- 通过稳定的接口契约提高可维护性
- 通过硬件抽象隐藏 MCU 细节
- 不依赖具体总线：设备驱动面向接口，而非具体外设
- 不依赖具体平台：无需修改驱动即可在 STM32、GD32 或 PC 测试环境中编译

驱动必须通过稳定接口隐藏硬件细节，业务逻辑不得直接访问 MCU 外设。

## 分层架构

完整驱动栈：

```text
APP
 ↓
MODULE（业务逻辑：估计器、控制器、导航）
 ↓
INTERFACE（只定义契约：uart_if.h、spi_if.h）
 ↓
DEVICE DRIVER（设备驱动：icm42688、w25q64、bmp280）
 ↓
BUS DRIVER（总线驱动：spi、i2c、uart、can）
 ↓
HAL / BSP（厂商 HAL → 平台抽象）
 ↓
STM32 HAL / CMSIS
```

禁止的模式：IMU 驱动直接调用 HAL。

```text
APP
 ↓
ICM42688  ← 内含 HAL_SPI_Transmit()
 ↓
HAL_SPI_Transmit()
```

这会将设备驱动与某个 MCU 系列绑定，使其无法移植、测试和复用。

## 驱动分类规则

生成驱动代码前，必须将其归入以下某一个类别。

### 总线驱动 — `driver/bus/`

负责特定 MCU 上的总线协议事务。

示例：`SPI`、`I2C`、`UART`、`CAN`、`USB`、`ETH`。

目录：`driver/bus/spi/`、`driver/bus/i2c/`。

### 设备驱动 — `driver/device/`

负责某个具体外设芯片。

示例：`ICM42688`、`MPU6050`、`W25Q64`、`BMP280`、`SX1278`。

目录：`driver/device/icm42688/`。

### BSP 驱动 — `bsp/`

负责映射到具体硬件的板级资源。

示例：`LED`、`Button`、`Buzzer`、`OLED`。

目录：`bsp/`，不放在 `driver/`。

许多 AI 工具会错误地将 LED、按键等板级组件放入 `driver/`。这些 BSP 组件应放在 `bsp/` 中。

### 驱动层职责边界

设备驱动中**不得**包含：

- PID 控制器逻辑
- 卡尔曼滤波实现
- 姿态估计算法
- 导航算法
- 应用状态机

在 AI 生成的飞控和机器人固件中，驱动代码与控制算法混杂是常见架构错误。IMU 驱动负责读取和转换传感器数据，姿态估计器负责处理这些数据，两者必须分层：

```text
driver/device/imu/          ← 只负责寄存器读取和数据转换
module/estimator/           ← 卡尔曼滤波、姿态融合
module/controller/          ← PID、控制律
```

## 必须采用的目录结构

```text
driver/
├── bus/
│   ├── spi/
│   ├── i2c/
│   ├── uart/
│   └── can/
├── device/
│   ├── icm42688/
│   │   ├── icm42688.c
│   │   ├── icm42688.h
│   │   └── icm42688_reg.h
│   ├── w25qxx/
│   │   ├── w25qxx.c
│   │   ├── w25qxx.h
│   │   └── w25qxx_reg.h
│   └── bmp280/
└── common/
    ├── error.h
    └── types.h
```

## 总线抽象规则

**每种总线都必须提供抽象接口。** 设备驱动只能依赖这些接口，不得依赖厂商 HAL 类型。

SPI 总线接口：

```c
typedef struct {
    int32_t (*transfer)(const uint8_t *tx, uint8_t *rx, uint32_t len);
    int32_t (*transfer_async)(const uint8_t *tx, uint8_t *rx, uint32_t len,
                              void (*callback)(int32_t status));
} spi_if_t;
```

I2C 总线接口：

```c
typedef struct {
    int32_t (*write)(uint8_t addr, const uint8_t *data, uint32_t len);
    int32_t (*read)(uint8_t addr, uint8_t *data, uint32_t len);
    int32_t (*write_read)(uint8_t addr, const uint8_t *cmd, uint32_t cmd_len,
                          uint8_t *rx, uint32_t rx_len);
} i2c_if_t;
```

设备驱动只能通过 `spi_if_t` 或 `i2c_if_t` 访问总线，不得直接使用 `SPI_HandleTypeDef` 或 `HAL_SPI_Transmit()`。

任何设备驱动中都禁止：

```c
SPI_HandleTypeDef hspi1;       // 禁止：MCU 专用类型
HAL_SPI_Transmit(&hspi1, ...); // 禁止：厂商 HAL API
```

必须通过总线接口：

```c
spi->transfer(tx, rx, len);    // 正确
```

## 设备上下文规则

驱动状态必须保存在设备上下文结构体中，不得使用全局变量。

禁止使用全局设备状态：

```c
static float gyro_x;
static float gyro_y;
static uint8_t initialized;  // 禁止
```

应使用设备上下文：

```c
typedef struct {
    spi_if_t      *spi;
    float          gyro[3];
    float          accel[3];
    uint8_t        initialized;
    uint32_t       timeout_ms;
} icm42688_t;
```

所有状态均属于设备对象，从而支持在不同总线通道上使用同类设备的多个实例。

## 初始化规则

统一初始化接口，所有设备驱动必须遵循以下签名：

```c
error_t xxx_init(xxx_t *dev);
```

示例：

```c
error_t icm42688_init(icm42688_t *imu);
```

禁止：

```c
void imu_init(void);  // 隐藏了全局状态，无法确定初始化哪个实例
```

## 寄存器封装规则

寄存器地址和位域定义必须集中放在 `_reg.h` 头文件中。

禁止在驱动中散布没有明确含义的寄存器地址：

```c
spi_write(0x1A, 0x80);   // 0x1A 和 0x80 分别代表什么？
```

应使用具名寄存器常量：

```c
// icm42688_reg.h
#define ICM42688_REG_PWR_MGMT0    (0x4EU)
#define ICM42688_BIT_TEMP_EN      (0x01U << 4)

// icm42688.c
ret = icm42688_write_reg(imu, ICM42688_REG_PWR_MGMT0, ICM42688_BIT_TEMP_EN);
```

## 同步与 DMA 规则

设备驱动不应感知 DMA 数据流、DMA 通道或 IRQ 编号，这些属于总线驱动层。

总线接口同时提供同步和异步传输函数：

```c
spi->transfer(tx, rx, len);           // 同步：阻塞至传输完成
spi->transfer_async(tx, rx, len, cb); // 异步：完成后回调
```

设备驱动可以选择传输模式，但不得直接配置 DMA 硬件。

设备驱动中禁止：

```c
DMA_HandleTypeDef hdma_spi1_tx;  // 禁止：DMA 属于总线层职责
__HAL_DMA_ENABLE(...);           // 禁止
```

## RTOS 隔离规则

设备驱动必须独立于 RTOS，不得调用：

```c
osDelay();          // 禁止：引入 RTOS 依赖
vTaskDelay();       // 禁止
osMutexAcquire();   // 禁止
```

设备驱动不得忙等：

```c
while (!(reg & BIT_READY));  // 禁止：自旋等待会阻塞整个系统
```

应支持超时、回调或事件机制：

```c
ret = spi->transfer(tx, rx, len);
if (ret == ERR_TIMEOUT) {
    // 处理超时，由调用者决定重试或恢复
}
```

阻塞、让出 CPU 和互斥保护等 RTOS 事务由设备驱动的**上层**处理。多个任务共享设备时，由模块层或应用层协调访问。

## 错误处理规则

所有驱动必须使用统一的错误码类型：

```c
typedef enum {
    ERR_OK = 0,
    ERR_TIMEOUT,
    ERR_BUSY,
    ERR_INVALID_PARAM,
    ERR_NOT_READY,
    ERR_COMMUNICATION,
} error_t;
```

禁止：

```c
return -1;  // 没有明确含义，无法判断失败原因
return -2;
return -3;
```

## 模拟测试规则

每个总线接口都必须支持模拟实现。总线抽象（`spi_if_t`、`i2c_if_t`）天然支持这一点：同一接口既可指向真实硬件，也可指向测试桩。

```c
// 真实硬件
spi_if_t spi = { .transfer = stm32_spi_transfer };

// 测试模拟实现
spi_if_t mock_spi = { .transfer = mock_spi_transfer };
```

这样无需 MCU 硬件也能在 PC 上测试设备驱动：

- 面向主机平台（x86/ARM64）编译
- 注入返回已知数据的 `mock_spi`
- 验证寄存器操作顺序和数据转换逻辑

## 检查清单

生成驱动代码后，应确认：

- [ ] 设备驱动没有 HAL API 调用
- [ ] 设备驱动没有 MCU 专用类型
- [ ] 设备驱动依赖总线接口，而非 MCU 总线外设
- [ ] 没有散布的魔法寄存器地址，均集中在 `_reg.h`
- [ ] 寄存器位定义采用具名常量
- [ ] 设备状态位于上下文结构体中，而非全局变量
- [ ] 没有文件作用域的 `static` 设备状态
- [ ] 初始化接口符合 `error_t xxx_init(xxx_t *dev)` 模式
- [ ] 错误码使用 `error_t` 类型，而非含义不明的 `-1`/`-2`
- [ ] 设备驱动中没有 DMA 配置
- [ ] 设备驱动中没有 IRQ 编号
- [ ] 驱动代码中没有 `osDelay`、`vTaskDelay` 或互斥锁等 RTOS 调用
- [ ] 没有忙等循环
- [ ] 总线接口支持注入模拟实现进行测试
- [ ] 驱动中没有 PID、卡尔曼滤波或控制算法逻辑

## 自动修正策略

发现违反规则的情况时：

1. **识别**设备驱动中直接依赖 HAL 或寄存器的部分
2. 将总线操作**提取**为 `spi_if_t` / `i2c_if_t` 接口
3. 将寄存器常量**移入**独立的 `_reg.h` 头文件
4. 将全局设备状态**封装**为上下文结构体
5. 将含义不明的错误码**替换**为 `error_t` 枚举值
6. 从设备层**移除** DMA、IRQ 和 RTOS 依赖
7. 将控制算法**分离**到 `module/`：驱动读取数据，模块处理数据
