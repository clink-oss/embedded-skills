---
name: embedded-rtos-design
description: >
  用于设计、生成或评审基于 FreeRTOS、CMSIS-RTOS2、RT-Thread、Zephyr、ThreadX 的嵌入式固件，
  适用于 STM32、GD32、ESP32、无人机飞控和机器人等项目。规定实时系统的任务划分、调度、
  优先级分配、任务间通信、ISR 协作、资源归属和看门狗监控规则。
  避免 AI 常见的错误模式：将多种职责混入单个庞大任务，以及让控制环依赖通信时序而产生抖动。
---

# 嵌入式 RTOS 设计

## 目的

本技能定义基于 RTOS 的嵌入式系统架构规则。

目标：

- 确定的调度行为
- 低延迟
- 任务隔离
- 可维护性
- 可扩展性
- 可预测的时序行为

适用于 FreeRTOS、CMSIS-RTOS2、RT-Thread、Zephyr、ThreadX 及类似 RTOS。

AI 最常见的错误是在一个庞大任务的同一循环中读取传感器、运行 PID、收发 UART、更新 OLED 和写入 Flash。这样会导致系统：

- 无法分析实时性
- 扩展时难以保证现有功能不被破坏
- 各项职责紧密耦合
- 受到无关操作引入的抖动影响

本技能将这种模式拆分为合理的任务架构。

## RTOS 设计原则

### 一个任务，一项职责

每个任务只负责一个功能领域。

正确的任务划分：

```text
sensor_task      — 读取传感器，发布数据
control_task     — 计算控制输出
telemetry_task   — 处理通信
logger_task      — 后台日志
```

错误做法：单个任务包办所有工作。

```text
main_task:
    读取 IMU
    运行 PID
    更新 OLED
    发送 UART 数据
    写入 Flash
    osDelay(1)
```

禁止这种模式，因为它耦合了无关职责，阻碍实时性分析，也无法扩展。

### 隔离控制环时序

控制环不得依赖通信时序。

允许：

```text
sensor_task（1 kHz）→ 队列 → control_task（1 kHz）→ 队列 → motor_task（1 kHz）
telemetry_task（50 Hz，独立运行）
```

禁止用通信事件触发控制环：

```text
UART 接收 → 触发控制环
```

将控制与通信耦合会引入抖动，使控制周期不稳定。这对飞控、平衡机器人和无人车尤其关键。

## 任务分类

智能体创建的每个任务都必须归入以下某个类别。

### 采集任务

示例：`imu_task`、`adc_task`、`encoder_task`、`gps_task`。

职责：

- 从硬件采集原始数据
- 通过队列或事件发布数据

不得包含：

- 控制算法
- 日志调用
- 网络或协议处理

### 控制任务

示例：`attitude_task`、`motor_control_task`、`balance_task`。

职责：

- 根据传感器输入计算控制输出
- 使用 `osDelayUntil` / `vTaskDelayUntil` 保证控制环时序

不得包含：

- SPI 或 I2C 寄存器访问
- 显示刷新
- 文件 I/O

### 通信任务

示例：`uart_task`、`can_task`、`usb_task`、`ethernet_task`。

职责：

- 协议编码和解码
- 管理通信缓冲区
- 将接收的数据转发到适当的队列

不得包含：

- 控制律计算
- 应用业务逻辑

### 服务任务

示例：`logger_task`、`storage_task`、`monitor_task`、`health_check_task`。

职责：

- 后台维护
- 健康状态监控
- 将日志数据写入非易失性存储

不得包含：

- 实时控制操作
- 传感器采集

## 调度规则

### 固定周期调度

周期任务必须使用固定周期调度，避免周期漂移。

禁止使用会产生漂移的循环：

```c
while (1) {
    work();
    osDelay(1);
}
```

`osDelay(1)` 从 `work()` 结束后才开始等待 1 ms，因此实际周期会随 `work()` 的执行时间发生漂移。

应使用无漂移调度：

```c
uint32_t tick = osKernelGetTickCount();

while (1) {
    work();
    osDelayUntil(tick, 1);
}
```

或：

```c
vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(1));
```

## 优先级规则

按实时性要求分配优先级：

```text
较高优先级
    ↑
    控制任务（最高，硬实时）
    ↑
    传感器任务（次高，保证数据新鲜度）
    ↑
    通信任务（软实时）
    ↑
    服务 / 日志任务（后台）
    ↑
    空闲任务（最低）
```

飞控优先级分配示例：

```text
control_task       优先级 6
imu_task           优先级 5
uart_task          优先级 4
logger_task        优先级 2
```

## 任务间通信规则

任务间共享数据**不得**直接使用没有保护的全局变量。

禁止：

```c
// 共享头文件
extern imu_data_t g_imu;   // 禁止
```

### 队列：传递数据流

使用消息队列将流式数据从生产者传递到消费者：

```text
imu_task → [队列] → control_task
```

```c
osMessageQueuePut(imu_queue_id, &data, 0, timeout);
osMessageQueueGet(imu_queue_id, &data, NULL, wait);
```

### 事件标志：通知状态变化

使用事件标志通知状态变化，例如：

- DMA 传输完成
- 数据就绪
- GPS 定位成功

```c
osEventFlagsSet(event_id, IMU_DATA_READY);
osEventFlagsWait(event_id, IMU_DATA_READY, flags, timeout);
```

### 互斥锁：保护共享资源

多个任务访问 Flash、SD 卡或共享内存缓冲区等资源时，使用互斥锁：

```c
osMutexAcquire(flash_mutex_id, timeout);
// 访问共享 Flash
osMutexRelease(flash_mutex_id);
```

**不要**将互斥锁用作消息传递机制。

## ISR 协作规则

ISR 必须尽量精简，仅允许以下操作：

- 设置事件标志
- 通过 ISR 安全 API 向队列发送数据
- 通过 ISR 安全 API 释放信号量
- 启动 DMA 传输

示例：

```c
void USART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    xQueueSendFromISR(rx_queue_id, &rx_byte, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

ISR 内禁止：

```c
printf()                  // 阻塞，执行时间不确定
malloc() / free()         // 延迟不确定
浮点运算                  // 栈空间和周期开销过大
协议解析                  // 应在任务上下文中执行
控制算法计算              // 应在任务上下文中执行
```

### DMA 延后处理模式

正确：

```text
DMA ISR → [事件] → communication_task → protocol_parser
```

错误：全部工作都在 ISR 上下文中完成。

```text
DMA ISR → protocol_parser → control_algorithm
```

## 资源归属规则

每个硬件资源必须且只能归属于一个任务。

示例：

```text
UART1 → 所属任务：telemetry_task
SPI1  → 所属任务：imu_task
I2C1  → 所属任务：sensor_task
```

禁止多个任务直接写入同一个 UART：

```text
imu_task       ──┐
logger_task    ──┤→ UART1（冲突）
control_task   ──┘
```

正确做法：由单一任务负责，通过队列供其他任务使用。

```text
所有任务 → [队列] → telemetry_task → UART1
```

## 看门狗规则

每个 RTOS 项目必须包含看门狗或健康监控机制：

```c
health_monitor_task:
    - 检查所有任务的周期性心跳
    - 监测队列占用情况
    - 通过空闲任务钩子跟踪 CPU 负载
    - 报告异常或触发安全状态
```

看门狗至少覆盖：

- 任务存活状态：每个任务在自身周期内切换一次存活标志
- 队列深度：检查是否接近溢出
- 栈高水位：使用 configASSERT 或 uxTaskGetStackHighWaterMark

## 时序文档

创建周期任务时，应在任务设计文档中记录以下参数：

```text
+----------------+------+--------+--------+--------+
| 任务           | Hz   | 优先级 | 栈大小 | WCET   |
+----------------+------+--------+--------+--------+
| imu_task       | 1000 | 5      | 1024   | 150 µs |
| control_task   | 1000 | 6      | 1024   | 200 µs |
| telemetry_task | 50   | 3      | 512    | 50 µs  |
+----------------+------+--------+--------+--------+
```

每个硬实时任务都应估算并记录 WCET（Worst-Case Execution Time，最坏情况执行时间）。

## 检查清单

生成 RTOS 代码后，应确认：

- [ ] 每个任务具有单一且明确的职责
- [ ] 没有混合采集、控制和通信的庞大任务
- [ ] 周期任务使用 `osDelayUntil` 或 `vTaskDelayUntil`，而非 `osDelay`
- [ ] 按实时性要求分配优先级
- [ ] ISR 中没有 `printf`、`malloc`、浮点运算、协议解析或控制逻辑
- [ ] ISR 通过队列或事件向任务交接，不直接共享全局变量
- [ ] 任务间通过队列、事件或互斥锁通信，不直接使用未保护的 `extern` 全局变量
- [ ] 控制环不依赖通信时序
- [ ] DMA 后处理在任务上下文中执行，而非 ISR
- [ ] 每个硬件资源仅归属于一个任务
- [ ] 存在看门狗或健康监控，并覆盖任务存活检查
- [ ] 已记录周期、优先级、栈大小和 WCET 等时序参数

## 自动修正策略

发现违反规则的情况时：

1. **识别**任务职责重叠或 ISR 承担过多工作的情况
2. 将庞大任务**拆分**为职责单一的任务
3. 用队列、事件或互斥锁**替换**直接共享全局变量的方式
4. 通过延后处理，将协议解析、控制计算等 ISR 繁重工作**移入**任务上下文
5. **引入**资源归属模型，每个外设只由一个任务负责
6. 根据实时性要求**重新分配**优先级
7. **重新生成** RTOS 架构和任务文档
