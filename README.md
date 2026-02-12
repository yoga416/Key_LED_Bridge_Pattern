# STM32 FreeRTOS 任务抢占与互斥量实验 (Homework_07)

## 1. 项目背景
本项目是基于 **STM32F411CE (Black Pill)** 开发板的嵌入式系统课程实验。实验通过模拟多任务并发环境，深入探究了 RTOS 的调度机制及共享资源访问冲突的解决方法。

## 2. 硬件环境
* **核心芯片**: STM32F411CEU6 (Cortex-M4, 主频 100MHz)
* **调试工具**: J-Link V9 (SWD 模式)
* **关键引脚**:
    * **USART1 (PA9/PA10)**: 用于串口日志输出，波特率 115200。
    * **PC13**: 用于控制板载 LED 灯。

## 3. 软件架构
项目基于 **CMSIS-RTOS V2** 接口，配置了三个具有不同优先级的任务：

| 任务名称 | 优先级 | 功能描述 |
| :--- | :--- | :--- |
| **DefaultTask** | `osPriorityNormal` | 每 5ms 唤醒一次，模拟高优先级“插嘴”打印提示信息。 |
| **Task02** | `osPriorityLow` | 连续循环打印长字符串 `02-START-12345...02-END`。 |
| **Task03** | `osPriorityLow` | 备用打印任务，用于观察同优先级任务的时间片轮转。 |



## 4. 实验核心：资源竞争 (Race Condition)
在未引入保护机制时，串口发送是一段非原子操作。当低优先级的 `Task02` 正在发送字符时，高优先级的 `DefaultTask` 会强制抢占 CPU，导致输出流中断。

### 实验捕获的典型乱码 Log：
> `02-START-1234567!!! [NORMAL PRIORITY] INTERRUPTED !!!890-END`
*(注：可以看到输出字符串被暴力截断，这就是典型的非线程安全现象。)*

## 5. 解决方案：Mutex (互斥量)
为了保证打印的完整性，项目引入了 `osMutexId_t uartMutexHandle`。

**代码实现逻辑：**
```c
if (osMutexAcquire(uartMutexHandle, osWaitForever) == osOK) {
    // 进入临界区：独占串口资源
    printf("02-START-1234567890-02-END\r\n");
    // 退出临界区：释放串口资源
    osMutexRelease(uartMutexHandle);
}
