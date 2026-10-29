---
name: embedded-isr-design
description: >
  用于生成或评审 STM32、GD32、ESP32、FreeRTOS、裸机、无人机飞控和机器人等嵌入式固件的
  中断服务程序（ISR）。遵循“快进快出”原则：ISR 捕获事件，将处理延后至任务上下文。
  检查 AI 常见错误，包括在 ISR 内解析协议、执行控制算法、调用 printf/malloc、使用阻塞调用、
  忘记清除中断标志，以及与 RTOS 交互时未使用 FromISR API。
---

# 嵌入式中断服务程序设计

## 目的

设计和评审嵌入式系统的中断服务程序。

目标：

- 低中断延迟
- 确定的执行过程
- 尽量减少 ISR 工作量
- 安全地与 RTOS 集成
- 可预测的系统行为

核心原则：**ISR 捕获事件，任务处理业务逻辑。** ISR 不应被当作普通函数看待；它属于硬件触发的上下文切换，必须尽快执行并返回。

AI 最常见且危险的错误，是在 ISR 中加入协议解析、传感器融合、PID 计算或 `printf` 调用。这种写法会：

1. 增加其他中断的响应延迟，因为它们被阻塞或抢占
2. 干扰 RTOS 调度，因为调度器只在 ISR 之外运行
3. 造成 DMA 缓冲区溢出，因为数据到达速度超过臃肿 ISR 的处理速度
4. 引入抖动，破坏控制环稳定性
5. 在 ISR 尝试获取被抢占任务持有的互斥锁时，产生死锁风险

## ISR 核心原则

**快进快出（Fast In, Fast Out）**：快速进入，只做必要操作，立即退出。

ISR 的执行时间应以微秒衡量，而不是毫秒。

## ISR 职责边界

### ISR 中允许的操作：仅捕获事件

- 读取硬件状态寄存器
- 清除中断标志
- 记录时间戳
- 将少量且有明确上限的数据复制到缓冲区或寄存器
- 设置事件标志（`xTaskNotifyGiveFromISR`、`osEventFlagsSet`）
- 向队列发送数据（`xQueueSendFromISR`、`osMessageQueuePut`）
- 释放信号量（`xSemaphoreGiveFromISR`）
- 启动 DMA 传输
- 唤醒任务（`vTaskNotifyGiveFromISR`）
- 递增有界计数器

### ISR 中禁止的操作：业务处理

- 协议解析，包括 JSON、protobuf、NMEA、MAVLink 和自定义二进制协议
- 姿态估计，如 Mahony、Madgwick
- PID 或其他控制律计算
- 卡尔曼滤波更新
- 路径规划或导航逻辑
- 文件系统操作
- Flash 写入或擦除
- 网络协议处理
- 显示刷新，如 OLED、LCD
- 日志输出
- 循环次数不固定的复杂循环
- 任何阻塞调用

## ISR 执行时间预算

每个 ISR 的执行时间都必须有明确上限且可预测。

| ISR 类型 | 建议的最大执行时间 |
|----------|--------------------|
| GPIO | < 5 µs |
| UART | < 10 µs |
| SPI | < 10 µs |
| DMA | < 10 µs |
| I2C | < 15 µs |
| CAN | < 20 µs |
| 定时器 | < 20 µs |

禁止在 ISR 中使用无明确上限的循环：

```c
for (int i = 0; i < large_variable; i++) {  // 禁止：没有明确上限
    process(data[i]);
}
```

## ISR 中的数据处理

允许进行最少量的数据搬运：

```c
rx_buf[index++] = data;       // 允许：有明确边界的环形缓冲区写入
```

禁止繁重处理：

```c
json_parse(&parser, byte);       // 禁止
protobuf_decode(&msg, buf, len); // 禁止
kalman_update(&kf, measurement); // 禁止
```

## `printf` 规则

ISR 上下文中严格禁止调用 `printf` 或同系列函数，因为这些函数：

- 可能阻塞，等待标准输出，而标准输出可能使用 UART、半主机或 ITM
- 执行时间不确定
- 通常不可重入
- 延迟很高，单次 `printf` 可能耗时数百微秒到数毫秒

禁止：

```c
void USART1_IRQHandler(void) {
    printf("rx byte: %02x\r\n", data);  // 禁止
}
```

## 动态内存规则

ISR 内严格禁止调用堆分配函数：

```c
malloc(size);     // 禁止
calloc(n, size);  // 禁止
realloc(ptr, sz); // 禁止
free(ptr);        // 禁止
```

堆操作的延迟不确定，可能涉及合并、分割，部分分配器还涉及垃圾回收，因此违反 ISR 执行时间必须有界的要求。

## 浮点运算规则

ISR 中禁止大量浮点计算：

```c
float angle = atan2f(y, x);  // 禁止：FPU 运算开销大、栈占用高
```

在带 FPU 的 Cortex-M4F/M7F 上，单次 `atan2f` 可能需要 50–150 个周期。在不带 FPU 的 Cortex-M3/M0 上，软件浮点模拟会增加数千个周期，并占用大量栈空间。

## HAL 调用规则

允许调用处理中断分发和硬件状态清除的函数：

```c
HAL_UART_IRQHandler(&huart1);  // 允许：分发至 HAL 接收、发送或错误回调
HAL_DMA_IRQHandler(&hdma);     // 允许
```

ISR 中禁止阻塞式 HAL 操作：

```c
HAL_UART_Transmit(&huart1, data, len, timeout);  // 禁止：阻塞、轮询
HAL_SPI_Transmit(&hspi1, data, len, timeout);    // 禁止
HAL_Delay(ms);                                    // 禁止：基于 SysTick 的阻塞
```

## DMA 协同设计

正确做法：DMA 完成后通知任务处理数据。

```text
DMA 完成中断 → [事件标志 / 任务通知] → 任务 → 数据处理
```

错误做法：直接在 DMA ISR 中处理数据。

```text
DMA 完成中断 → 协议解析 → 控制逻辑（禁止）
```

## RTOS 集成规则

使用 FreeRTOS、CMSIS-RTOS2、RT-Thread、Zephyr 等 RTOS 时，ISR 与任务通信**必须**使用 RTOS API 的 `FromISR` 版本。

FreeRTOS 应使用：

```c
xQueueSendFromISR(queue, &data, &higherPriorityTaskWoken);
vTaskNotifyGiveFromISR(task, &higherPriorityTaskWoken);
xSemaphoreGiveFromISR(semaphore, &higherPriorityTaskWoken);
portYIELD_FROM_ISR(higherPriorityTaskWoken);
```

CMSIS-RTOS2 应使用：

```c
osMessageQueuePut(queue, &data, 0, timeout);      // 允许在 ISR 中使用
osEventFlagsSet(flags, mask);                     // 允许在 ISR 中使用
```

禁止在 ISR 中使用非 `FromISR` API：

```c
xQueueSend(queue, &data, timeout);     // 禁止
osMessageQueuePut(queue, &data, ...);  // timeout 非零时禁止
xSemaphoreGive(semaphore);             // 禁止
```

始终检查 `higherPriorityTaskWoken`，为真时调用 `portYIELD_FROM_ISR()`，以便在退出最外层中断前触发调度。

## 中断优先级规则

中断优先级必须按实时性要求分配，而非图省事：

```text
最高优先级
    ↑
    定时器控制（控制环节拍，最高，硬实时）
    ↑
    DMA（数据吞吐、缓冲区管理）
    ↑
    SPI / I2C（传感器数据流）
    ↑
    UART（通信，优先级低于控制）
    ↑
    EXTI（用户按键、低速事件）
    ↑
最低优先级
```

飞控示例：

```text
TIM 更新中断        优先级 0（最高，控制环节拍）
IMU 数据就绪中断    优先级 1
SPI DMA 中断        优先级 2
UART 中断           优先级 3
EXTI（按键）        优先级 4
```

**控制相关中断的优先级必须高于通信中断。** 避免 UART 突发数据延误控制环。

## 共享数据规则

ISR 和任务上下文共享的普通全局变量必须受到保护。优先采用以下机制，按推荐顺序排列：

1. **队列**：ISR 入队，任务出队。

   ```c
   // ISR
   xQueueSendFromISR(rx_queue, &byte, &woken);

   // 任务
   xQueueReceive(rx_queue, &byte, portMAX_DELAY);
   ```

2. **双缓冲**：ISR 写入一个缓冲区，任务读取另一个。

   ```c
   static uint8_t buf[2][256];
   static uint8_t active_buf = 0;
   static uint16_t index = 0;

   // ISR：填充当前缓冲区
   buf[active_buf][index++] = byte;

   // 任务：缓冲区填满后交换并处理
   ```

3. **事件标志 + 单一读取者**：适用于简单状态共享。

   ```c
   // ISR
   vTaskNotifyGiveFromISR(consumer_task, &woken);

   // 任务
   ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
   ```

禁止在没有同步机制的情况下随意共享全局变量：

```c
volatile imu_t imu_data;  // ISR 写入，任务读取，没有同步

// ISR
imu_data = sensor_read();     // 存在问题

// 任务
process(&imu_data);           // 可能读到只更新了一部分的数据
```

### UART 接收使用环形缓冲区

UART 接收应使用环形缓冲区，不要在 ISR 中解析：

```text
UART 中断 → 环形缓冲区 → UART 任务 → 协议解析器
```

```c
// ISR
ringbuf_put(&uart_rx_ring, rx_byte);

// 任务
uint8_t byte;
ringbuf_get(&uart_rx_ring, &byte);
// 在这里解析协议，运行于任务上下文
```

## 定时器 ISR 规则

定时器 ISR 应只负责：

- 生成系统节拍（`osKernelGetTickCount`）
- 触发周期性事件标志
- 记录输入捕获时间戳
- 递增计时用计数器

定时器 ISR 中禁止：

```c
void TIM_IRQHandler(void) {
    HAL_TIM_IRQHandler(&htim);

    pid_update();           // 禁止：控制算法属于任务职责
    telemetry_send();       // 禁止：通信属于任务职责
    sensor_fusion_update(); // 禁止：估计运算属于任务职责
}
```

## ISR 设计模式

### 标志模式：单个二值事件

```c
void EXTI_IRQHandler(void) {
    data_ready = true;     // 简单标志，由任务轮询检查
    EXTI->PR1 |= (1U << 0);
}
```

### 队列模式：数据流

```c
void USART1_IRQHandler(void) {
    uint8_t byte;
    byte = (uint8_t)(USART1->DR & 0xFFU);
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(rx_queue, &byte, &woken);
    portYIELD_FROM_ISR(woken);
}
```

### 事件模式：任务通知

```c
void DMA2_IRQHandler(void) {
    DMA2->IFCR |= (1U << 0);
    BaseType_t woken = pdFALSE;
    vTaskNotifyGiveFromISR(processor_task, &woken);
    portYIELD_FROM_ISR(woken);
}
```

### 环形缓冲区模式：缓冲 UART 接收数据

```c
void USART1_IRQHandler(void) {
    uint8_t byte = (uint8_t)(USART1->DR & 0xFFU);
    ringbuf_put(&uart_rx_ring, byte);
    USART1->SR &= ~(USART_SR_RXNE);
}
```

## 飞控专项规则

飞控和机器人应用尤其需要遵守以下要求。

禁止在 IMU 数据就绪 ISR 中运行 Mahony 或 PID：

```text
IMU 中断 → Mahony → PID → PWM（全部在 ISR 内，禁止）
```

应由 ISR 通知控制任务：

```text
IMU 中断 → [任务通知]
             ↓
         控制任务（1 kHz）
             ↓
         Mahony → PID → PWM
```

采用这种模式，控制环周期由 RTOS 调度器决定，而非 ISR 执行时间。ISR 延迟会累积为控制环抖动，尽量缩短 ISR 才能保证控制时序一致、可预测。

## 检查清单

生成 ISR 代码后，应确认：

- [ ] ISR 具有单一且明确的职责
- [ ] ISR 执行时间有明确上限且可预测
- [ ] ISR 中没有循环次数可变的复杂循环
- [ ] ISR 中没有协议解析
- [ ] ISR 中没有控制算法计算
- [ ] ISR 中没有 `printf` 或日志输出
- [ ] ISR 中没有 `malloc`、`free` 或其他堆操作
- [ ] ISR 中没有 `HAL_Delay`、轮询等待等阻塞调用
- [ ] 在适用场景使用 RTOS 的 `FromISR` API
- [ ] 正确清除中断标志
- [ ] 通过事件通知或队列实现 ISR 到任务的交接
- [ ] ISR 职责与任务职责分离
- [ ] ISR 中没有业务逻辑
- [ ] DMA ISR 将数据处理延后至任务上下文
- [ ] 使用 RTOS 时，请求唤醒任务后调用了 `portYIELD_FROM_ISR()`

## 自动修正策略

发现 ISR 违反规则时：

1. **识别** ISR 中的繁重操作，包括解析、数学计算、循环和阻塞调用
2. 将这些操作**移入**专门的任务上下文
3. 用队列、事件或环形缓冲区**替换** ISR 与任务间直接共享变量的通信方式
4. 将阻塞 API **替换**为非阻塞或 `FromISR` 版本
5. 必要时**引入**环形缓冲区以积累串口数据
6. 修改后**重新计算**最坏情况下的中断延迟
7. 按检查清单**重新验证**完整的 ISR 设计
