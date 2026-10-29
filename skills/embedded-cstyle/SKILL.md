---
name: embedded-cstyle
description: >
  用于生成、修改、评审或重构嵌入式固件项目的 C 代码，适用于 STM32、GD32、ESP32、
  FreeRTOS、裸机、驱动开发和机器人等场景。规定命名、类型、函数、变量、宏、头文件、
  内存、并发、错误处理和文档方面的工程规范。检查 AI 常见错误，包括全局变量、魔法数字、
  缺少 static/const、滥用 volatile、忽略返回值、未检查空指针，以及业务逻辑直接调用 HAL。
---

# 嵌入式 C 编码规范

## 目的

本技能定义嵌入式固件项目必须遵循的编码规范。

目标：

- 可读性
- 可维护性
- 可移植性
- 安全性
- 确定的执行行为

本规范参考 MISRA-C、Linux 内核编码风格、AUTOSAR C 指南和 STM32 固件开发实践，并针对中小型嵌入式项目的实际使用进行调整，不以严格满足这些标准为目标。

生成、修改、评审或重构 C 代码时均适用本技能。

需要权衡时，优先级为：**安全性 > 确定性 > 可维护性 > 可读性**。

## 接口边界规则

应用层和业务模块不得直接调用 HAL API。

`app/` 和 `module/` 中禁止以下调用：

```c
HAL_GPIO_WritePin()
HAL_UART_Transmit()
HAL_SPI_Transmit()
HAL_I2C_Master_Transmit()
```

只有 `interface/` 和 `bsp/` 层可以访问厂商 HAL API。

## 命名规则

### 类型命名

格式：`typename_t`。

应使用：

```c
motor_t
imu_t
pid_t
uart_config_t
```

禁止：

```c
Motor
IMU
PIDStruct
UART_CONFIG
```

### 函数命名

格式：`module_action()`，使用小写字母和下划线。

应使用：

```c
motor_init()
motor_set_speed()
imu_read()
pid_update()
```

禁止：

```c
InitMotor()
ReadIMU()
PID()
```

文件内部函数必须声明为 `static`：

```c
static void motor_update(void);
```

### 宏命名

格式：`MODULE_DESCRIPTION`，使用大写字母和下划线。

应使用：

```c
MOTOR_MAX_SPEED_RPM
UART_RX_BUFFER_SIZE
PID_DEFAULT_KP
```

禁止：

```c
MaxSpeed
bufferSize
```

## 类型规则

始终使用 `<stdint.h>` 中的定宽类型。

禁止：

```c
int
short
long
unsigned
```

应使用：

```c
uint8_t
uint16_t
uint32_t
int8_t
int16_t
int32_t
```

布尔值必须使用 `<stdbool.h>`：

```c
#include <stdbool.h>

bool ready;
```

禁止用以下类型代替布尔类型：

```c
u8 flag;
uint8_t status;
```

## 变量规则

### 全局变量

禁止直接暴露普通全局变量：

```c
int speed;  // 禁止
```

允许使用文件私有的静态变量：

```c
static int32_t speed;  // 允许
```

跨模块访问必须通过接口函数：

```c
motor_get_speed()  // 正确
```

不得使用：

```c
extern int speed;  // 禁止
```

### 优先使用 `const`

不可修改的数据必须声明为 `const`：

```c
static const uint8_t crc_table[256];
static const float pid_gains[3] = {1.0f, 0.1f, 0.05f};
```

### `volatile` 使用约束

`volatile` 只允许用于：

- 与 ISR 共享的变量
- DMA 状态变量
- 内存映射外设寄存器

```c
volatile bool uart_rx_done;  // 正确
```

禁止用于普通变量：

```c
volatile int count;  // 禁止
```

## 函数规则

### 参数检查

解引用指针参数前必须先检查：

```c
if (ptr == NULL) {
    return ERR_INVALID_PARAM;
}
```

禁止：

```c
ptr->data = 1;  // 未检查空指针
```

### 返回值检查

调用者必须检查返回值：

```c
ret = uart_write(...);
if (ret != ERR_OK) {
    return ret;
}
```

禁止：

```c
uart_write(...);  // 忽略返回值
```

## 宏规则

### 禁止魔法数字

禁止：

```c
if (speed > 123)
```

应使用：

```c
#define MOTOR_MAX_SPEED_RPM (123U)

if (speed > MOTOR_MAX_SPEED_RPM)
```

### 宏体加括号

禁止：

```c
#define MAX_SPEED 100 + 20
```

应使用：

```c
#define MAX_SPEED (120U)
```

## 头文件规则

### 防止重复包含

每个头文件必须采用以下方式之一：

```c
#ifndef MOTOR_H
#define MOTOR_H

// ...

#endif
```

或：

```c
#pragma once
```

### 减少头文件依赖

头文件中优先使用前向声明，减少 `#include`：

```c
struct motor;  // 前向声明
```

避免：

```c
#include "a.h"
#include "b.h"
#include "c.h"
#include "d.h"
```

## 内存规则

### 禁止动态分配

禁止运行时堆内存分配：

```c
malloc()
calloc()
realloc()
free()
```

改用静态分配：

```c
static uint8_t rx_buffer[256];
```

这样可避免堆碎片和不可预测的分配延迟，这对实时嵌入式系统至关重要。

## 并发规则

### 临界区

多个线程或 ISR 访问的共享资源必须受到保护：

```c
osMutexAcquire(mutex_id, timeout);
// ... 访问共享资源 ...
osMutexRelease(mutex_id);
```

或：

```c
taskENTER_CRITICAL();
// ... 访问共享资源 ...
taskEXIT_CRITICAL();
```

## 错误处理规则

### 统一错误码

使用项目统一的错误枚举，避免没有明确含义的返回值：

```c
typedef enum {
    ERR_OK = 0,
    ERR_TIMEOUT,
    ERR_BUSY,
    ERR_INVALID_PARAM,
    ERR_NOT_INITIALIZED,
    ERR_OVERFLOW,
    ERR_UNDERFLOW,
} error_t;
```

禁止：

```c
return -1;
return -2;
return -3;
```

## 文档规则

所有公开接口都必须具有 Doxygen 风格的注释块：

```c
/**
 * @brief 设置电机转速
 *
 * @param rpm 目标转速，单位为转/分钟
 *
 * @return error_t 成功时返回 ERR_OK
 */
error_t motor_set_speed(int32_t rpm);
```

## 检查清单

生成或评审代码后，应确认：

- [ ] 使用 `<stdint.h>` 类型，没有直接使用 `int`、`short`、`long`、`unsigned`
- [ ] 布尔值使用 `<stdbool.h>`
- [ ] 内部函数声明为 `static`
- [ ] 解引用指针前已检查
- [ ] 没有忽略返回值
- [ ] 运行时不调用 `malloc`/`free`
- [ ] 没有魔法数字，所有常量均通过具名宏表示
- [ ] 宏体已加括号
- [ ] 尽量减少全局变量，跨模块通过函数访问
- [ ] 不可修改的数据均使用 `const`
- [ ] `volatile` 只用于 ISR、DMA 或寄存器相关变量
- [ ] 命名遵循 `module_action`、`typename_t`、`MACRO_NAME` 约定
- [ ] 公开接口具有 Doxygen 注释块
- [ ] `app/` 和 `module/` 中没有 HAL 调用

## 自动修正策略

发现违反规则的情况时：

1. **报告**问题所在的文件和行号
2. **解释**对嵌入式系统的影响，包括安全性、确定性或可移植性风险
3. **给出**修正后的代码
4. 对机械性修改，自动**重构**

修正优先级：

```text
安全性 > 确定性 > 可维护性 > 可读性
```
