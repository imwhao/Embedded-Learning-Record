# RTOS 入门
## 裸机与 RTOS 特点
裸机：裸机又称为前后台系统，前台系统指的中断服务函数，后台系统指的大循环，即应用程序。 

裸机特点：
1. 实时性差：应用程序轮流执行
2. delay：空等待，CPU 不执行其他代码
3. 结构臃肿：实现功能都放在无限循环

RTOS：RTOS 全称为：Real Time OS，就是实时操作系统，强调的是实时性。

RTOS 特点：
1. 分而治之：实现功能划分为多个任务
2. 延时函数：任务调度（高优先级任务进入延时后可把 CPU 使用权交给低优先级任务，待高优先级任务延时结束后收回使用权）
3. 抢占式：高优先级任务抢占低优先级任务
4. 任务堆栈：每个任务都有自己的栈空间，用于保存局部变量以及任务的上下文信息
注意：中断可以打断任意任务。任务可以同等优先级。

## FreeRTOS 简介/基础知识
FreeRTOS 是一个免费的嵌入式实时操作系统。

特点：
1. 免费开源：可商用
2. 可裁剪：FreeRTOS 的核心代码 9000+行，包含在 3 个 `.c` 文件中
3. 简单：简单易用，可移植性非常好
4. 优先级不限：任务优先级分配没有限制，多任务可同一优先级（若所使用最高优先级算法为硬件，则受限于 MCU 架构）
5. 任务不限：可创建的实时任务数量没有软件限制（硬件限制取决于堆栈大小）
6. 抢占/协程/时间片：支持抢占式，协程式、时间片流转任务调度

### 任务调度
FreeRTOS 支持三种任务调度方式：
1. 抢占式调度：主要是针对优先级不同的任务，每个任务都有一个优先级，优先级高的任务可以抢占优先级低的任务（数值越大优先级越高）
2. 时间片调度：主要针对优先级相同的任务，当多个任务的优先级相同时，任务调度器会在每一次系统时钟节拍到的时候切换任务，时间片的大小取决于滴答定时器时钟频率
3. 协程式调度：当前执行任务将会一直运行，同时高优先级的任务不会抢占低优先级任务（官方不再更新）

### 任务状态
FreeRTOS 中任务共存在 4 种状态：
1. 运行态：正在执行的任务处于运行态（在 stm32 中同一时间仅有一个任务处于运行态）
2. 就绪态：如果该任务已经能够被执行，但当前还未被执行，那么该任务处于就绪态
3. 阻塞态：如果一个任务因延时或等待外部事件发生，那么这个任务就处于阻塞态
4. 挂起态：类似暂停，调用函数 `vTaskSuspend()` 进入挂起态，需要调用解挂函数 `vTaskResume()` 才可以进入就绪态
注：只有就绪态能转变成运行态。这四种状态中，除了运行态，其他三种任务状态的任务都有其对应的任务状态列表。调度器总是在所有处于就绪列表的任务中选择具有最高优先级的任务来执行。

# FreeRTOS 移植
移植步骤：
1. 添加 FreeRTOS 源码：将 FreeRTOS 源码添加至基础工程、头文件路径等
2. 添加 `FreeRTOSConfig.h` 配置文件
3. 修改 SYSTEM 文件：修改 SYSTEM 文件中的 `sys.c`、`delay.c`、`usart.c`
4. 修改中断相关文件：修改 Systick 中断、SVC 中断、PendSV 中断
5. 添加应用程序：验证移植是否成功

系统配置文件：
- 『INCLUDE』：配置 FreeRTOS 中可选的 API 函数
- 『config』：完成 FreeRTOS 的功能配置和裁剪
- 其他配置项：PendSV 宏定义、SVC 宏定义

# 任务创建和删除
## 任务创建和删除的 API 函数
任务创建和删除的本质就是调用 FreeRTOS 的 API 函数

| API 函数              | 描述             |
| --------------------- | ---------------- |
| `xTaskCreate()`       | 动态方式创建任务 |
| `xTaskCreateStatic()` | 静态方式创建任务 |
| `vTaskDelete()`       | 删除任务         |

- 动态创建任务：任务的任务控制块以及任务的栈空间所需的内存，均由 FreeRTOS 从 FreeRTOS 管理的堆中分配
- 静态创建任务：任务的任务控制块以及任务的栈空间所需的内存，需用户分配提供


## 任务创建和删除
实现动态创建任务流程：
1. 将宏 `configSUPPORT_DYNAMIC_ALLOCATION` 配置为 1
2. 定义函数入口参数
3. 编写任务函数
用起来只需要以上三步。此函数创建的任务会立刻进入就绪态，由任务调度器调度运行

静态创建任务使用流程：
1. 需将宏 `configSUPPORT_STATIC_ALLOCATION` 配置为 1
2. 定义空闲任务 & 定时器任务的任务堆栈及 TCB
3. 实现两个接口函数
	1. `vApplicationGetIdleTaskMemory()`
	2. `vApplicationGetTimerTaskMemory()`
4. 定义函数入口参数
5. 编写任务函数
注：静态创建任务的任务句柄是由 `xTaskCreateStatic()` 返回的，动态创建任务直接指定句柄。

动态方式创建任务与静态方式创建任务比较：
1. 在实际应用中，动态方式创建任务相对简单，比较常用，除非有特殊需求，一般都会用动态方式创建任务
2. 静态创建可将任务堆栈放置在指定的内存位置，并且无需关心对内存分配失败的处理

删除任务流程：
1. 使用删除任务函数，需将宏 `INCLUDE_vTaskDelete` 配置为 1
2. 入口参数输入需要删除的任务句柄 **（NULL 代表删除本身）**
注：静态创建的任务申请的内存需要用户在任务被删除前提前释放，否则将导致内存泄漏。 

# 任务挂起和恢复
任务的挂起与恢复的 API 函数

| API 函数             | 描述             |
| -------------------- | ---------------- |
| `VTaskSuspend()`     | 挂起任务         |
| `vTaskResume()`      | 恢复被挂起的任务 |
| `vTaskResumeFromISR` | 在中断中恢复被挂起的任务                 |
注：中断服务程序中要调用 FreeRTOS 的 API 函数则中断优先级不能高于 FreeRTOS 所管理的最高优先级（5~15）且中断要全部设置为抢占式。

# FreeRTOS 中断管理
对于 STM32 中断的介绍和中断优先级分组的配置见 [STM32 学习笔记中断部分](https://github.com/imwhao/Embedded-Learning-Record/blob/main/stm32/Notes.md#%E4%B8%AD%E6%96%AD)

注意：
1. 低于 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 优先级的中断里才允许调用 FreeRTOS 的 API 函数
2. FreeRTOS 为了方便配置，将中断优先级分组配置为 `NVIC_PriorityGroup_4`，0~15 级抢占优先级，0 级子优先级。（在 `HAL_Init()` 中调用函数 `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)` 即可完成设置）
3. 中断优先级数值越小越优先，任务优先级数值越大越优先

FreeRTOS 将 PendSV 和 SysTick 设置为最低优先级，保证系统任务切换不会阻塞其他中断的响应。

FreeRTOS 所使用的中断管理利用的是 `BASEPRI` 寄存器，屏蔽优先级低于某一个阈值的中断。比如：`BASEPRI` 设置为 0x50（低 4 位不被使用），代表中断优先级在 5~15 内的均被屏蔽，0~4 级的中断优先级正常执行。

# FreeRTOS 临界段代码保护及调度器挂起与恢复
## 临界区代码保护
临界段代码也叫临界区，是指那些必须完整运行，不能被打断的代码段。

适用场合如：
1. 外设：需严格按照时序初始化的外设：IIC、SPI 等等
2. 系统：系统自身需求
3. 用户：用户需求

临界段代码保护函数：

| 函数                            | 描述             |
| ------------------------------- | ---------------- |
| `taskENTER_CRITICAL()`          | 任务级进入临界段 |
| `taskEXIT_CRITICAL()`           | 任务级退出临界段 |
| `taskENTER_CRITICAL_FROM_ISR()` | 中断级进入临界段 |
| `taskEXIT_CRITICAL_FROM_ISR()`  | 中断级退出临界段                 |
中断级进入临界段函数有返回值，需保存下来在退出临界段时作为参数传入 `taskEXIT_CRITICAL_FROM_ISR()`

特点：
1. 成对使用
2. 支持嵌套（多次进入临界区，需要多次退出才会开中断）
3. 尽量保持临界段耗时短

## 任务调度器的挂起与恢复
挂起任务调度器，调用此函数不需要关闭中断

| 函数                | 描述           |
| ------------------- | -------------- |
| `vTaskSuspendAll()` | 挂起任务调度器 |
| `xTaskResumeAll()`  | 恢复任务调度器 |

特点：
1. 与临界区不一样的是，挂起任务调度器，未关闭中断
2. 它仅仅是防止了任务之间的资源争夺，中断照样可以直接响应
3. 挂起调度器的方式，适用于临界区位于任务与任务之间，既不用去延时中断，又可以做到临界区的安全

# 列表和列表项
## 介绍
列表是 FreeRTOS 中的一个数据结构，概念上和链表有点类似，列表被用来跟踪 FreeRTOS 中的任务。列表项就是存放在列表中的项目。

列表相当于链表，列表项相当于节点，FreeRTOS 中的列表是一个**双向环形链表**。

列表结构体：
```c
typedef struct xLIST
{

      listFIRST_LIST_INTEGRITY_CHECK_VALUE  /* 校验值 */
      volatile UBaseType_t uxNumberOfItems;  /* 列表中的列表项数量（不包括末尾列表项） */
      ListItem_t * configLIST_VOLATILE pxIndex  /* 用于遍历列表项的指针 */
      MiniListItem_t xListEnd  /* 末尾列表项 */
      listSECOND_LIST_INTEGRITY_CHECK_VALUE  /* 校验值 */

} List_t;
```

列表项结构体：
```c
struct xLIST_ITEM
{
	listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE  /* 用于检测列表项的数据完整性 */
	configLIST_VOLATILE TickType_t xItemValue  /* 列表项的值 */
	struct xLIST_ITEM * configLIST_VOLATILE pxNext  /* 下一个列表项 */
	struct xLIST_ITEM * configLIST_VOLATILE pxPrevious  /* 上一个列表项 */
	void * pvOwner  /* 列表项的拥有者（通常是任务控制块） */
	struct xLIST * configLIST_VOLATILE pxContainer;   /* 列表项所在列表 */
	listSECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE  /* 用于检测列表项的数据完整性 */
};
typedef struct xLIST_ITEM ListItem_t;
```

迷你列表项（仅用于标记列表的末尾和挂载其他插入列表中的列表项）：
```c
struct xMINI_LIST_ITEM
{
	listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE   /* 用于检测数据完整性 */
	configLIST_VOLATILE TickType_t xItemValue;  /* 列表项的值 */
	struct xLIST_ITEM * configLIST_VOLATILE pxNext;  /* 上一个列表项 */
	struct xLIST_ITEM * configLIST_VOLATILE pxPrevious; /* 下一个列表项 */
};
typedef struct xMINI_LIST_ITEM MiniListItem_t;
```

## 列表相关 API 函数
| 函数                    | 描述               |
| ----------------------- | ------------------ |
| `vListInitialise()`     | 初始化列表         |
| `vListInitialiseItem()` | 初始化列表项       |
| `vListInsertEnd()`      | 列表插入列表项（插入到列表 `pxIndex` 指针指向的列表项前面，一种无序的插入方法） |
| `vListInsert()`         | 列表插入列表项（列表项值升序插入）    |
| `uxListRemove()`        | 列表移除列表项                   |






