# 学习准备
实验基于正点原子 `stm32f103精英开发板V1`。学习教程主要参考[手把手教你学STM32 HAL库开发全集（新版视频）](https://www.bilibili.com/video/BV1bv4y1R7dp)。

学习资源列表：
1. [正点原子论坛](http://www.openedv.com/forum.php)
2. [stm32f103精英开发板V2资料下载中心](http://www.openedv.com/docs/boards/stm32/zdyz_stm32f103_jingyingV2.html)
3. [stm32f103精英开发板V1资料下载中心](http://www.openedv.com/docs/boards/stm32/zdyz_stm32f103_jingying.html)
4. [手把手教你学STM32 HAL库开发全集（新版视频）](https://www.bilibili.com/video/BV1bv4y1R7dp)
5. [手把手教你学STM32入门教学视频（老版视频）](https://www.bilibili.com/video/BV1Lx411Z7Qa/) 

开发板使用说明：
1. 连接外设时可使用资料中 `精英板 IO引脚分配表.xlsx` 文件查询引脚分配情况。
2. 例程测试时确保开发板 B0，B1 都接在 GND

# 基础篇
## 程序下载
[下载接口](https://www.bilibili.com/video/BV1bv4y1R7dp?t=1425.2&p=4)：
1. JTAG：占用 5 个引脚，可以下载、仿真、调试
2. SWD：占用 2 个引脚，可以下载、仿真、调试（推荐）
3. 串口：占用 2 个引脚，只能下载程序，不能调试

### 串口下载
使用工具：FlyMcu
1. 搜索并选择 CH340 虚拟串口并设置波特率。
	1. F4 系列波特率最大可以使用 76800
	2. F1 系列的波特率最大可以使用 460800
2. 选择 hex 文件
3. 勾选『编程前重装文件』
4. 勾选『校验』和『编程后执行』
5. **不要**勾选『编程到 FLASH 时写选项字节』
6. 选择『DTR 的低电平复位，RTS 高电平进 BootLoader』

### JLINK/STLINK 下载
具体配置见[程序下载方法2：JLINK程序下载](https://www.bilibili.com/video/BV1Lx411Z7Qa/?p=9)。

## 程序调试
1. JTAG 调试需要占用 5 个引脚。
2. SWD 调试需要占用 2 个引脚。

JTAG/SWD 调试原理：Cortex-M 内核含有内核硬件调试模块，该模块可在取指（指令断点）或访问数据（数据断点）时停止。内核停止时，可以查询内核的内部状态和系统的外部状态。完成查询后，可恢复程序运行。

为了在调试期间可以使用更多 GPIOs，通过设置复用重映射和调试 I/O 配置寄存器 (AFIO_MAPR) 的 SWJ_CFG[2:0]位，可以改变重映像配置。具体配置参考《STM32F10xxx 参考手册》 。

## 存储器映射
给存储器分配地址的过程叫存储器映射。

STM32 将 4G (2^32) 空间平均分为 8 个块。

## 寄存器映射
寄存器是单片机内部一种特殊的内存，可以实现对单片机各个功能的控制。给寄存器的地址命名的过程就叫寄存器映射。

### 地址计算
寄存器地址计算过程（以 GPIOA_ODR 寄存器为例）：
1. 获取外设挂在哪个总线上面？查系统结构图。
2. 获取总线基地址，APB2 总线基地址：0X4001 0000。
3. 获取外设地址偏移量，GPIOA 相对 APB2 总线偏移量是：0X800。
4. 获取寄存器地址偏移量，ODR 相对 GPIOA 外设基地址的偏移量是：0X0C。
5. 寄存器地址=BUS_BASE_ADDR+PERIPH_OFFSET+REG_OFFSET
   GPIOA_ODR = 0X4001 0000+0X800+0X0C=0X4001080C

### 映射方法
1. 宏定义直接映射，将寄存器名字映射到寄存器地址。
2. 结构体映射，将同一个外设的寄存器作为成员放在一个结构体中，成员数据类型位数与寄存器位数相同，又因为同一个结构体内存连续分配，所以只需要映射外设基地址，外设中的所有寄存器就能完成映射。
结构体映射例子：`#define GPIOA ((GPIO_TypeDef*)GPIOA_BASE)`

## HAL 库工程建立
裁剪的两种方式：
1. 在 `stm32f1xxx_hal_conf.h` 中注释掉对应外设
2. 不将驱动文件包含到 `Drivers/STM32F1xx_HAL_Driver` 中。

HAL 库用户配置文件 `stm32f1xxx_hal_conf.h` 使用：
1. 裁剪 HAL 库外设驱动源码（条件编译使对应需要裁剪的驱动源码不编译）
2. 设置外部高速晶振频率
3. 设置外部低速晶振频率

## stm32 启动模式
### STM32 启动模式（F1)
在系统复位后，SYSCLK 的第 4 个上升沿，BOOT 引脚的值会被锁存。BOOT0 和 BOOT1 为启动模式选择引脚。不同启动模式（主闪存存储器、系统存储器、内置 SRAM）将 0x00000000（堆栈指针 MSP 初始值，即栈顶地址）和 0x00000004（程序计数器 PC 初始值，即复位中断服务函数地址）所映射的地址不同。

### STM32CubeMX
ST 开发的一款图形配置工具，可通过配置自动生成初始化代码。

## stm32 时钟树
### 系统时钟树配置步骤
1. 配置 HSE_VALUE：告诉 HAL 库外部晶振频率 `stm32xxxx_hal_conf.h`
2. 调用 `SystemInit()` 函数（可选）：在启动文件中调用，在 `system_stm32xxxx.c` 定义
3. 选择时钟源，配置 PLL：通过 `HAL_RCC_OscConfig()` 函数设置
4. 选择系统时钟源，配置总线分频器：通过 `HAL_RCC_ClockConfig()` 函数设置
5. 配置扩展外设时钟（可选）：通过 `HAL_RCCEx_PeriphCLKConfig()` 函数设置
正点原子中调用 3+4+5= `sys_stm32_clock_init()`

### 外设时钟使能和失能
**我们要使用某个外设，必须先使能该外设时钟！**（为降低功耗，外设时钟默认失能）

```c
__HAL_RCC_GPIOA_CLK_ENABLE(); /*使能GPIOA时钟*/
__HAL_RCC_GPIOA_CLK_DISABLE(); /*禁止GPIOA时钟*/ 
```

## SYSTEM 文件夹介绍
正点原子提供的常用函数。其中子文件夹包括：
### sys 
包括中断类函数、低功耗类函数、设置栈顶指针函数、**系统时钟初始化函数**、Cache 配置函数（F7/H7）
### delay 
初始化系统滴答定时器、微妙/毫秒延时函数：
	1. 初始化滴答定时器时声明了一个全局变量 `g_fac_us=sysclk/分频数`，代表 1 $\mu s$ 计时器的自减值。（参数 `sysclk` 单位为 MHz）
由系统滴答计时器实现微秒延时函数。（不超频情况下最大能够实现约 1.8s 的微妙级延时）
用微秒延时函数实现毫秒延时函数。
### usart
printf 函数输出流程：
1. printf () 用户调用。
2. printf 函数由编译器提供的 stdio. h 解析。
3. fputc () 最终实现输出（用户需要根据最终输出的硬件重新定义该函数，此过程称为 printf 重定向）。

printf 函数支持：
1. 避免使用半主机模式（仿真器在电脑上输入输出）。两种方法：
	1. 微库法：在魔术棒 target 中勾选 Use MicroLIB，实现简单，速度慢，兼容性差。
	2. 代码法：需要实现一个预处理、两个定义、三个函数，实现复杂，速度快，兼容性好。（`usart.c` 文件中实现，直接复制即可）
2. 实现 fputc 函数。

# 入门篇
## GPIO
### GPIO 需要熟悉的知识
GPIO (General Purpose Input Output) 作用：
1. 输入（采集外部器件信息）
2. 输出（控制外部器件工作）

GPIO 电气特性：
1. 电压工作范围 2V 至 3.6V
2. GPIO 识别电压范围：CMOS 端口（不带 FT）：-0.3 至 1.164V 0, 1.833 至 3.6V 1。
3. GPIO 输出电流：单个 IO 最大 25mA

STM32 引脚类型：电源引脚、晶振引脚、复位引脚、下载引脚、BOOT 引脚、GPIO 引脚。

GPIO 的 8 种工作模式：
1. 输入浮空
2. 输入上拉
3. 输入下拉
4. 模拟输入
5. 开漏输出
	1. 不能输出高电平，只能输出 0 或高阻态，除非有内部或外部的上拉
6. 推挽输出
	1. 可以输出 0 和 1
	2. 驱动能力强
7. 开漏式复用功能
8. 推挽式复用功能

GPIO 寄存器介绍：
1. 端口配置低寄存器 (GPIOx_CRL) (x=A..E)
2. 端口配置高寄存器 (GPIOx_CRH) (x=A..E)
3. 端口输入数据寄存器 (GPIOx_IDR) (x=A..E)
4. 端口输出数据寄存器 (GPIOx_ODR) (x=A..E)
5. 端口位设置/清除寄存器 (GPIOx_BSRR) (x=A..E)
6. 端口位清除寄存器 (GPIOx_BRR) (x=A..E)
7. 端口配置锁定寄存器 (GPIOx_LCKR) (x=A..E)

ODR 寄存器控制输出和 BSRR 寄存器控制输出的区别：ODR 可读可写，寄存器赋值的步骤是读、改、写，BSRR 只可写，控制输出直接写。使用 ODR 在读和修改访问之间产生中断时可能发生风险，BSRR 无风险。

### 通用外设驱动模型
四步法：
1. 初始化：时钟设置、参数设置、IO 设置、中断设置（开中断、设 NVIC）（可选）
2. 读函数（可选）
3. 写函数（可选）
4. 中断服务函数（可选）

#### GPIO 配置步骤：
1. 初始化
	1. 使能时钟 `__HAL_RCC_GPIOx_CLK_ENABLE()`
	2. 设置工作模式 `HAL_GPIO_Init()`
2. 设置输出状态（可选）`HAL_GPIO_WritePin()`、`HAL_GPIO_TogglePin()`
3. 读取输入状态（可选）`HAL_GPIO_ReadPin()`

## 中断
中断的作用：
1. 实时控制：在确定时间内对相应事件作出响应。
2. 故障处理：检测到故障需要第一时间处理。
3. 数据传输：不知道数据什么时候来，如串口数据接收。
高效处理紧急程序，不会一直占用 CPU 资源。

### NVIC
Nested vectored interrupt controller，嵌套向量中断控制器，属于内核。
NVIC 支持：256 个中断（16 内核+240 外部），支持 256 个优先级，允许裁剪。（ST 公司裁剪为 16 个）

stm32 中断优先级基本概念：
1. 抢占优先级（pre）：高抢占优先级可以打断正在执行的低抢占优先级中断。
2. 响应优先级（sub）：当抢占优先级相同时，响应优先级高的先执行，但是不能互相打断。
3. 自然优先级：抢占和响应都相同的情况下，自然优先级（中断向量表的优先级）越高的，先执行。

stm32 中断优先级分组：通过配置 `AIRCR[10:8]` 对 `IPRx bit[7:4]` 的四个位进行抢占优先级数目和响应优先级分配。

STM32 NVIC 的使用：
1. 设置中断分组 `AIRCR[10:8], HAL_NVIC_SetPriorityGrouping`
2. 设置中断优先级 `IPRx bit[7:4], HAL_NVIC_SetPriority`
3. 使能中断 `ISERx, HAL_NVIC_EnableIRQ`

### EXTI
External (Extended) interrupt/event Controller，外部（扩展）中断事件控制器。包含 20 个产生事件/中断请求的边沿检测器，即总共：20 条 EXTI 线（F1）。

EXTI 寄存器：
1. 中断屏蔽寄存器 (EXTI_IMR)
2. 事件屏蔽寄存器 (EXTI_EMR)
3. 上升沿触发选择寄存器 (EXTI_RTSR)
4. 下降沿触发选择寄存器 (EXTI_FTSR)
5. 软件中断事件寄存器 (EXTI_SWIER)
6. 挂起寄存器 (EXTI_PR) 

AFIO：Alternate Function IO，即复用功能 IO，主要用于重映射和外部中断映射配置（F1）：
1. 调试 IO 配置：AFIO_MAPR[26:24]，配置 JTAG/SWD 的开关状态
2. 重映射配置：AFIO_MAPR，部分外设 IO 重映射配置
3. 外部中断配置：AFIO_EXTICR1~4，配置 EXTI 中断线 0~15 对应到哪个具体 IO 口。
特别注意：配置 AFIO 寄存器之前要使能 AFIO 时钟，方法如下：`__HAL_RCC_AFIO_CLK_ENABLE();对应RCC_APB2ENR寄存器 位0`

中断的产生：
1. EXTI 中断
	1. GPIO 设置输入模式。
	2. AFIO（F1）或 SYSCFG（其他）设置 EXTI 和 IO 映射关系。
	3. EXTI。
	4. NVIC：设置中断分组、优先级、使能。
	5. CPU：按优先级顺序，依次处理中断。
2. 外设中断
	1. USART/TIM/SPI...
	2. NVIC：设置中断分组、优先级、使能。
	3. CPU：按优先级顺序，依次处理中断。

STM32 EXTI 的配置步骤（GPIO 外部中断）：
1. 使能 GPIO 时钟
2. 设置 GPIO 输入模式
3. 使能 AFIO/SYSCFG 时钟
4. 设置 EXTI 和 IO 对应关系
5. 设置 EXTI 屏蔽，上/下沿
6. 设置 NVIC
7. 设计中断服务函数
注：HAL 库步骤 2~5 使用 `HAL_GPIO_Init` 一步到位。

STM32 EXTI 的 HAL 库配置步骤（GPIO 外部中断）：
1. 使能 GPIO 时钟 `__HAL_RCC_GPIOx_CLK_ENABLE`
2. GPIO/AFIO (SYSCFG)/EXTI `HAL_GPIO_Init`
3. 设置中断分组 `HAL_NVIC_SetPriorityGrouping`，**此函数只需要设置一次**
4. 设置中断优先级 `HAL_NVIC_SetPriority`
5. 使能中断 `HAL_NVIC_EnableIRQ`
6. 设计中断服务函数 `EXTIx_IRQHandler`，中断服务函数，清中断标志
   STM32 仅有 EXTI0~4、EXTI9_5、EXTI15_10，7 个外部中断服务函数

HAL 库中断回调处理机制介绍：
1. 中断服务函数
2. HAL 库中断处理公用函数
3. HAL 库数据处理回调函数（函数名以 Callback 结尾）

## 串口通信
串口：即串行通信接口，指按位发送和接收的接口。如：RS-232/422/485 等。

常见的串行通信接口：

| 通信接口                     | 接口引脚                                                     | 数据同步方式 | 数据传输方向 |
| ---------------------------- | ------------------------------------------------------------ | ------------ | ------------ |
| UART<br />（通用异步收发器） | TXD: 公共端<br />RXD：接收端<br />GND：公共地                 | 异步通信     | 全双工       |
| 1-wire                       | DQ: 发送/接收端                                               | 异步通信     | 半双工       |
| IIC                          | SCL: 同步时钟<br />SDA: 数据输入/输出端                        | 同步通信     | 半双工       |
| SPI                          | SCK: 同步时钟<br />MISO: 主机输入，从机输出<br />MOSI: 主机输出，从机输入<br />CS: 片选信号 | 同步通信     | 全双工       |

### RS232
RS-232 电平与 CMOS/TTL 电平对比：

| 类型       | 逻辑 1     | 逻辑 0     |
| ---------- | --------- | --------- |
| RS232      | -15V 至-3V | +3V 至+15V |
| CMOS (3.3V) | 3.3V      | 0V        |
| TTL (5V)    | 5V        | 0V        |
结论：CMOS/TTL 电平不能与 RS232 电平直接交换信息。


### USART
Universal synchronous asynchronous receiver transmitter，通用同步异步收发器。
Universal asynchronous receiver transmitter，通用异步收发器，USART 裁剪调同步功能。

F1 波特率计算公式：$$baud=\frac{f_{ck}}{16*USARTDIV}$$

USART/UART 异步通信配置步骤：
1. 配置串口工作参数 `HAL_UART_Init()`
2. 串口底层初始化 `HAL_UART_MspInit()` 配置 GPIO、NVIC、CLOCK 等
3. 开启串口异步接收中断 `HAL_UART_Receive_IT()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 编写中断服务函数 `USARTx_IRQHandler()`、`UARTx_IRQHandler()`
6. 串口数据发送 寄存器 `USART_DR` HAL 库 `HAL_UART_Transmit()`

## IWDG
IWDG 简介：
- IWDG 全称：Independent watchdog，即独立看门狗。
- IWDG 本质：能产生**系统复位信号**的计数器。
- IWDG 特性：独立的计数器，时钟由独立的 RC 振荡器提供（可在待机和停止模式运行），看门狗被激活后，当递减计数器计数到 0x000 时产生复位。
- 喂狗：计数器计到 0 之前，重装载计数器的值，防止复位。

IWDG 作用：
1. 异常：外界电磁干扰或自身系统（硬件或软件）异常，造成程序跑飞，如：陷入某个不正常的死循环，打断正常的程序运行。
2. IWDG 作用：主要用于检测外界电磁干扰，或硬件异常导致的程序跑飞问题。
3. 应用：在一些需要高稳定性的产品中，并且对时间精度要求较低（RC 振荡器精度较低）的场合。
**注意：独立看门狗是异常处理的最后手段，不可依赖，应在设计时尽量避免异常。**

IWDG 溢出时间计算
IWDG 溢出时间计算公式：$$T_{out}=\frac{psc*rlr}{f_{IWDG}}$$
其中，$T_{out}$ 是看门狗溢出时间，$f_{IWDG}$ 是看门狗的时钟源频率，$psc$ 是看门狗预分频系数，$rlr$ 是看门狗重装载值。

IWDG 配置步骤：
1. 取消 PR/RLR 寄存器写保护，设置 IWDG 预分频系数和重装载值，启动 IWDG `HAL_IWDG_Init()`
2. 及时喂狗，即写入 0xAAAA 到 IWDG_KR `HAL_IWDG_Refresh()`

## WWDG
Window watchdog，即窗口看门狗。
WWDG 简介：
- WWDG 本质：能产生**系统复位信号**和**提前唤醒中断**的计数器。
- WWDG 特性：
	- 递减的计数器。
	- 当递减计数器值从 0x40 减到 0x3F 时复位。
	- 计数器的值大于 W[6:0]值时喂狗会复位。
	- 提前唤醒中断（EWI）：当递减计数器等于 0x40 时可产生。
- 喂狗：在窗口期内重装载计数器的值，防止复位。
	- $W[6:0]\ge窗口期>0x3F]$

WWDG 作用：监测单片机运行时效是否精准，主要检测**软件异常**。
WWDG 应用：需要精准监测程序运行时间的场合。

WWDG 超时时间计算：$$T_{out}\frac{4096*2^{WDGTB}*(T[5:0]+1)}{F_{wwdg}}$$
其中，$T_{out}$ 是 WWDG 超时时间（没喂狗）；$F_{wwdg}$ 是 WWDG 的时钟源频率；4096 是 WWDG 固定的预分频系数；$2^{WDGTB}$ 是 WWDG_CFR 寄存器设置的预分频系数值；T[5:0]是 WWDG 计数器低 6 位。

WWDG 配置步骤：
1. WWDG 工作参数初始化：`HAL_WWDG_Init()`
2. WWDG Msp 初始化：`HAL_WWDG_MspInit()` 配置 NVIC、CLOCK 等
3. 设置优先级，使能中断：`HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
4. 编写中断服务函数：`WWDG_IRQHandler()` -> `HAL_WWDG_IRQHandler`
5. 重定义提前唤醒回调函数：`HAL_WWDG_EarlyWakeupCallback()`
6. 在窗口期喂狗：`HAL_WWDG_Refresh()`

IWDG 和 WWDG 对比：

| 对比点         | 独立看门狗                   | 窗口看门狗                       |
| -------------- | ---------------------------- | -------------------------------- |
| 时钟源         | LSI（40KHz 或 32KHz）          | PCLK1 或 PCLK3                     |
| 复位条件       | 递减计数到 0                  | 计数值大于 W[6:0]值喂狗或减到 0x3F |
| 中断           | 没有中断                     | 计数值减到 0x40 可产生中断         |
| 递减计数器位数 | 12 位（最大计数范围：4096~0） | 7 位（最大计数范围：127~63）      |
| 应用场合       | 防止程序跑飞，死循环，死机   | 检测程序时效，防止软件异常                                 |

## TIMER
### 定时器概述
软件定时原理：使用纯软件（CPU 死等）方式实现定时（延时）功能。

软件延时缺点：
1. 延时不精准
2. CPU 死等

定时器定时：使用精准的时基，通过硬件的方式，实现定时功能。

stm32 定时器分类：
1. 常规定时器
	1. 基本定时器 TIM6/TIM7
	2. 通用定时器 TIM2/TIM3/TIM4/TIM5
	3. 高级定时器 TIM1/TIM8
2. 专用定时器
	1. 独立看门狗
	2. 窗口看门狗
	3. 实时时钟
	4. 低功耗定时器
3. 内核定时器 SysTick 定时器

stm32 基本、通用、高级定时器功能整体的区别：

| 定时器类型 | 主要功能                                                |
| ---------- | ------------------------------------------------------- |
| 基本定时器 | 没有输入输出通道，常用作时基，即定时功能                |
| 通用定时器 | 具有多路独立通道，可用于输入捕获/输出比较，也可用作时基 |
| 高级定时器 | 除具备通用定时器所有功能外，还具备带死区控制的互补信号输出、刹车输入等功能（可用于电机控制、数字电源设计等）                                                        |

### 基本定时器
基本定时器简介：
1. 基本定时器：TIM6/TIM7
2. 主要特性：
	1. 16 位递增计数器（计数值：0~65535）
	2. 16 位预分频器（分频系数：1~65536），分频系数为寄存器值+1
	3. 可用于触发 DAC

定时器中断实验相关寄存器（F1）：
1. TIM6 和 TIM7 控制寄存器 1（TIMx_CR1）
	1. 位 7 ARPE（Auto-reload preload enable)
		1. 0：TIMx_ARR 寄存器没有缓冲（对寄存器写入后直接生效）。
		2. 1：TIMx_ARR 寄存器具有缓冲（更新事件发生后才写入影子寄存器生效）。
		3. 作用：在微妙级计时前后计时时间发生变化时，等待更新事件发生后对寄存器的写入会耗费一定时间，当具有缓冲时可以在更新时间发生前写入寄存器，增加计时精度。
	2. 位 0 CEN（Counter Enable）
2. TIM6 和 TIM7 DMA/中断使能寄存器（TIMx_DIER）
3. TIM6 和 TIM7 状态寄存器（TIMx_SR） 用于判断是否发生了更新中断，由硬件置 1，软件清零
4. TIM6 和 TIM7 计数器（TIMx_CNT）计数器值可读可写
5. TIM6 和 TIM7 预分频器（TIMx_PSC）分频系数为 PSC 值+1

定时器溢出时间的计算方法：$$T_{out}=\frac{(ARR+1)*(PSC+1)}{F_t}$$
其中，$T_{out}$ 是定时器溢出时间，$F_t$ 是定时器的时钟源频率，$ARR$ 是自动重装载寄存器的值，$PSC$ 是预分频器寄存器的值。（ARR+1 是因为当设置 ARR 为 0 时也要计一个数）

定时器中断实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_Base_Init()`
2. 定时器基础 MSP 初始化 `HAL_TIM_Base_MspInit()` 配置 NVIC、CLOCK 等
3. 使能更新中断并启动计数器 `HAL_TIM_Base_Start_IT()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 编写中断服务函数 `TIMx_IRQHandler()` 等-> `HAL_TIM_IRQHandler()`
6. 编写定时器更新中断回调函数 `HAL_TIM_PeriodElapsedCallback()`

### 通用定时器
#### 通用定时器简介（F1）
1. 通用定时器 TIM2/TIM3/TIM4/TIM5
2. 主要特性（加粗为相对基本定时器的新特性）
	1. 16 位递增、**递减、中心对齐**计数器（计数值：0~65535）
		1. 递增、递减计数器又称边沿对齐计数器
	2. 16 位预分频器（分频系数：1~65536）
	3. 可用于触发 DAC、**ADC**
	4. 在更新时间、**触发事件、输入捕获、输出比较**时，会产生中断/DMA 请求
	5. **4 个独立通道，可用于：输入捕获、输出比较、输出 PWM、单脉冲模式**
	6. **使用外部信号控制定时器且可实现多个定时器互连的同步电路**（定时器的级联）
	7. **支持编码器和霍尔传感器等**

计数器时钟源：
1. 内部时钟（CK_INT），来自外部总线 APB 提供的时钟（是否 x2 需要看预分频器）
2. 外部时钟模式 1：外部输入引脚（TIx），来自定时器通道 1 或者通道 2 引脚的信号
3. 外部时钟模式 2：外部触发输入（ETR），来自可以复用为 TIMx_ETR 的 IO 引脚
4. 内部触发输入（ITRx），用于与芯片内部其它通用/高级定时器级联

#### 通用定时器 PWM 输出实验
通用定时器输出 PWM 原理：
假设采用递增技术模式，ARR 为自动重装载寄存器的值，CCRx 为捕获/比较寄存器 x 的值，当 CNT<CCRx，IO输出0，当CNT>=CCRx，IO 输出 1。此时，PWM 波周期或频率由 ARR 决定，PWM 波占空比由 CCRx 决定。

通用定时器 PWM 输出实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_PWM_Init()`
2. 定时器 PWM 输出 MSP 初始化 `HAL_TIM_PWM_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置 PWM 模式/比较值等 `HAL_TIM_PWM_ConfigChannel()`
4. 使能输出并启动计数器 `HAL_TIM_PWM_Start()`
5. 修改比较值控制占空比（可选） `__HAL_TIM_SET_COMPARE()`
6. 使能通道预装载（可选）`__HAL_TIM_ENABLE_OCxPRELOAD()`

PWM 周期/频率的计算方法同定时器溢出时间的计算方法：$$T_{out}=\frac{(ARR+1)*(PSC+1)}{F_t}$$
其中，$T_{out}$ 是定时器溢出时间，$F_t$ 是定时器的时钟源频率，$ARR$ 是自动重装载寄存器的值，$PSC$ 是预分频器寄存器的值。（ARR+1 是因为当设置 ARR 为 0 时也要计一个数）

#### 通用定时器输入捕获实验
通用定时器输入捕获脉宽测量原理：
以捕获测量**高电平脉宽**为例，假设计数器为递增技术模式，ARR 为自动重装载寄存器的值，CCRx1 为 t1 时间点 CCRx1 的值，CCRx2 为 t2 时间点 CCRx 的值。捕获上升沿后将 CCRx 清零，上升沿触发改为下降沿触发。高电平期间，计时器计数的个数为：`N*(ARR+1) + CCRx2`

通用定时器输入捕获实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_IC_Init()`
2. 定时器输入捕获 MSP 初始化 `HAL_TIM_IC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置输入通道映射、捕获边沿等 `HAL_TIM_IC_ConfigChannel()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 使能定时器更新中断 `__HAL_TIM_ENABLE_IT()`
6. 使能捕获、捕获中断及计数器 `HAL_TIM_IC_Start_IT()`
7. 编写中断服务函数 `TIMx_IRQHandler()` 等-> `HAL_TIM_IRQHandler()`
8. 编写更新中断和捕获回调函数 `HAL_TIM_PeriodElapsedCallback()`、`HAL_TIM_IC_CaptureCallback()`

#### 通用定时器脉冲计数实验
通用定时器脉冲计数实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_IC_Init()`
2. 定时器输入捕获 MSP 初始化 `HAL_TIM_IC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置定时器从模式等 `HAL_TIM_SlaveConfigSynchro()`
4. 使能输入捕获并启动计数器 `HAL_TIM_IC_Start()`
5. 获取计数器的值 `__HAL_TIM_GET_COUNTER()`
6. 设置计数器的值 `__HAL_TIM_SET_COUNTER()`

### 高级定时器
高级定时器简介：
1. 高级定时器 TIM1/TIM8
2. 主要特性：除通用定时器特性外还具有
	1. 重复计数器
	2. 死区时间带可编程的互补输出
	3. 断路输入，用于将定时器的输出信号置于用户可选的安全配置中

重复计数器特性：计数器每次上溢或下溢都能使重复计数器减 1，减到 0 时，再发生一次溢出就会产生更新事件。**如果设置 RCR 为 N，更新事件将在 N+1 次溢出时发生。**

#### 高级定时器输出指定个数 PWM 实验
高级定时器输出指定个数 PWM 原理：
1. 配置边沿对齐模式输出 PWM（同通用定时器 PWM 输出）
2. 指定输出 N 个 PWM，则把 N-1 写入 PCR
3. 在更新中断内，关闭计数器
**注意：高级定时器通道输出必须把 MOE 位置 1**

高级定时器输出指定个数 PWM 实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_PWM_Init()`
2. 定时器 PWM 输出 MSP 初始化 `HAL_TIM_PWM_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置 PWM 模式/比较值等 `HAL_TIM_PWM_ConfigChannel()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 使能定时器更新中断 `__HAL_TIM_ENABLE_IT()`
6. 使能输出、主输出、计数器 `HAL_TIM_PWM_Start()`
7. 编写中断服务函数 `TIMx_IRQHandler()` 等 -> `HAL_TIM_IRQHandler()`
8. 编写更新中断回调函数 `HAL_TIM_PeriodElapsedCallback()`

#### 高级定时器输出比较模式实验
输出比较模式下翻转功能作用是：当计数器的值等于捕获/比较寄存器影子寄存器的值时，OC1REF 发生翻转，进而控制通道输出（OCx）翻转。

通过翻转功能实现输出 PWM 的具体原理如下：PWM 频率由自动重载寄存器（TIMx_ARR）的值决定，在这个过程中，只要自动重载寄存器的值不变，那么 PWM 占空比就固定为 50%。我们可以通过捕获/比较寄存器（TIMx_CCRx）的值改变 PWM 的相位。

高级定时器输出比较模式实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_OC_Init()`
2. 定时器输出比较 MSP 初始化 `HAL_TIM_OC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置输出比较模式等 `HAL_TIM_OC_ConfigChannel()`
4. 使能通道预装载 `__HAL_TIM_ENABLE_OCxPRELOAD()`
5. 使能输出、主输出、计数器 `HAL_TIM_OC_Start()`
6. 修改捕获/比较寄存器的值 `__HAL_TIM_SET_COMPARE()`

#### 高级定时器互补输出带死区控制实验
带死区的互补输出应用：H 桥控制电机正反转。

死区时间计算：
1. 确定 $t_{DTS}$ 的值：$f_{DTS}=\frac{F_t}{2^{CKD[1:0]}}$，其中 CKD[1:0]为时钟分频因子，这 2 位定义在定时器时钟（CK\_INT）频率、死区时间和由死区发生器与数字滤波器（ETR, Tlx）所用的采样时钟（$t_{DTS}$）之间的分频比例。
2. 判断 DTG[7:5]，选择计算公式。
3. 代入选择的公式计算。

高级定时器互补输出带死区控制实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_PWM_Init ()`
2. 定时器 PWM 输出 MSP 初始化 `HAL_TIM_PWM_MspInit ()`
3. 配置 PWM 模式/比较值等 `HAL_TIM_PWM_ConfigChannel()`
4. 配置刹车功能、死区时间等 `HAL_TIMx_ConfigBreakDeadTime()`
5. 使能输出、主输出、计数器 `HAL_TIM_PWM_Start()`
6. 使能互补输出、主输出、计数器 `HAL_TIMEx_PWMN_Start()`

#### 高级定时器 PWM 输入模式实验
PWM 输入模式只能用 CH1 或 CH2（CH3、CH4 从模式无法选择复位模式）。

高级定时器 PWM 输入模式实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_IC_Init()`
2. 定时器捕获输入 MSP 初始化 `HAL_TIM_IC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置 IC1/2 映射、捕获边沿等 `HAL_TIM_IC_ConfigChannel()`
4. 配置从模式，触发源等 `HAL_TIM_SlaveConfigSynchro()`
5. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
6. 使能捕获、捕获中断及计数器 `HAL_TIM_IC_Start_IT()`、`HAL_TIM_IC_Start()`
7. 编写中断服务函数 `TIMx_IRQHandler()` 等-> `HAL_TIM_IRQHandler()`
8. 编写输入捕获回调函数 `HAL_TIM_IC_CaptureCallback()`

## OLED
### OLED 简介
常用显示屏：LCD 显示屏，点阵显示屏，OLED（Organic Light-Emitting Diode）显示屏

OLED 简介
1. 优点：
	1. 自发光，不需要背光
	2. 功耗更低，节能
	3. 对比度高，色彩艳丽
2. 缺点：
	1. 烧屏
	2. 价格昂贵
	3. 低频频闪
3. 应用场景：
	1. OLED 电视
	2. 手机/平板
	3. 手表/手环

### OLED 驱动原理
怎么驱动屏幕？
1. 选择驱动芯片时序，根据时序实现数据写入/读取。
2. 初始化序列，由厂家提供，初始化屏幕。
3. 实现画点函数、读点函数（可选），基于这两个函数可以实现各种绘图功能。

### OLED 驱动芯片简介
什么是页地址模式？
对 GRAM 进行操作时，列地址指针会自动递增。当列地址指针到达列结束地址时，重置为开始地址，但页地址指针保持不变。用户必须设置新的页面和列地址，以便访问下一页 GRAM 内容。（每一页对应 8 行，页地址可用 `y/8` 得到）

GRAM 和 OLED 屏幕坐标对应关系表：一个通用的点 (x, y) 置 1 表达式为：`OLED_GRAM[x][y/8] |= 1 << y % 8`

### OLED 基本驱动步骤
目标：用最简单代码，点亮 OLED 屏，实现任意位置画点。
1. 确定 IO 连接关系：开发板 OLED 接口原理图
2. 初始化 IO 口：初始化连接 OLED 的各个 IO 口
3. 编写 8080 接口函数：`oled_wr_byte`
4. 编写 OLED 初始化函数：编写 `oled_init` 函数，完成初始化序列配置
5. 编写 OLED 画点函数：编写 `oled_draw_point` 函数，实现 OLED 任意位置画点

## LCD
### 简介
LCD 基本组成：
1. 玻璃基板
2. 背光
3. 驱动 IC

LCD 接口分类

1. MCU：分辨率 $\le800*400$ 带 SRAM，无需频繁刷新，无需大内存，驱动简单
2. RGB：分辨率 $\le1200*800$ 不带 SRAM，需要实时刷新，需要大内存，驱动稍微复杂
3. MIPI：分辨率 4k 不带 SRAM，支持分辨率高，省电，大部分手机屏用此接口

三基色原理：无法通过其它颜色混合得到的颜色称之为基本色，通过三基色的混合可以得到自然界中绝大部分颜色。

电脑一般用 32 位来表示一个颜色（ARGB888）：透明度[31:24]，R[23:16], G[15:8]，B[7:0]。

单片机一般用 16/24 位表示一个颜色（RGB565/RGB888）。

### LCD 驱动原理
LCD 驱动的一般过程：
1. 8080 底层操作函数：写数据、写命令、读数据
2. 初始化 LCD：主要是发送初始化序列/数组
3. 实现画点函数：有了画点函数，就可以实现各种操作函数了
4. 实现读点函数：用于读取屏幕颜色，一般上了 GUI 采用，可不用

8080 时序简介：并口总线时序，常用于 MCU 屏驱动 IC 的访问，由 Intel 提出，也叫英特尔总线。

| 信号    | 名称      | 控制状态  | 作用                                  |
| ------- | --------- |----| ------------------------------------- |
| CS      |片选| 低电平    | 选中器件，低电平有效，先选中，后操作  |
| WR      | 写        | ↑         | 写信号，上升沿有效，用于数据/命令写入 |
| RD      |读|↑|读信号，上升沿有效，用于数据/命令读取|
| RS      | 数据/命令 | 0=命/1=数 |表示当前是读写数据还是命令，也叫 DC 信号|
| D[15:0] | 数据线    | 无        |双向数据线，可以写入/读取驱动 IC 数据|

### LCD 基本驱动实现
目标：用最简单代码，点亮开发板 LCD 屏，实现任意位置画点和读点
1. 确定 IO 连接关系
2. 初始化 IO 口
3. 初始化 FSMC 外设（可选，某些芯片没有 FSMC 外设）
4. 编写读写接口函数：`lcd_wr_data`、`lcd_wr_regno`、`lcd_write_reg`、`lcd_rd_data`
5. 编写 LCD 初始化函数：编写 `lcd_init` 函数，完成初始化序列配置，点亮背光等
6. 编写 LCD 画点和读点函数

### FSMC（F1）
FSMC，Flexible Static Memory Controller，灵活的静态存储控制器。

作用：用于驱动 SRAM，NOR FLASH，NAND FLASH 及 PC 卡类型的存储器。配置好 FSMC，定义一个指向这些地址的指针，通过对指针操作就可以直接修改存储单元的内容，FSMC 自动完成读写命令和数据访问操作，不需要程序去实现时序。FSMC 外设配置好就可以模拟出时序。可用于驱动 LCD 时实现 8080 通讯接口时序。

FMC（F4/F7/H7） 和 LTDC 精英版 Stm32f103 不具备相应硬件资源。

## USMART
USMART 是一个串口调试组件，可以直接通过串口调用用户编写的函数，随意修改函数参数，提高代码调试效率。

USMART 主要特点：
1. 可以调用绝大部分用户直接编写的函数
2. 占用资源少（最小：4KB FLASH，72B SRAM）
3. 支持参数类型多（整数（10/16）、字符串、函数指针等）
	1. 注：**不支持**浮点数参数
4. 支持函数返回值显示且可对格式进行设置
5. 支持函数执行时间计算

修改 `usmart_port. c/. h` 即可完成移植，修改 `usmart_config. c` 即可添加自己想要调用的函数。

USMART 移植步骤：
1. 获取 USMART 组件
2. 添加到工程：添加全部组件到工程，4 个文件，设置好路径关联
3. 适配硬件：修改调试串口和定时器，以适配自己的硬件
4. 添加执行函数：在 `usmart_config. c` 中添加自己需要的执行函数
5. 通过串口交互：烧录移植好的 USMART 组件，可以通过串口反复测试目标函数

开启后可以通过串口发送 `?` 获取帮助信息

## RTC
实时时钟（Real Time Clock，RTC），本质是一个计数器，计数频率常为秒，专门用来记录时间。

### 特性
1. 能提供时间（秒钟数）
2. 能在 MCU **掉电后运行**
3. 低功耗

### RTC 基本配置步骤（寄存器）
1. 使能对 RTC 的访问：使能 PWR&&BKP 时钟（RCC_APB1ENR）、使能对后备寄存器和 RTC 的访问权限（PWR_CR）
2. 设置 RTC 时钟源：激活 LSE，设置 RTC 的计数时钟源为 LSE（RCC_BDCR）
3. 进入配置模式：等待 RTOFF 位为 1，设置 CNF 位为 1（RTC_CRL）
4. 设置 RTC 寄存器：设置分频值（RTC_PRL）、计数值等，一般先只设置分频值，CNT 的设置（RTC_CNT）独立
5. 退出配置模式：清除 CNF 位（RTC_CRL），等待 RTOFF 为 1 即配置完成

### RTC 相关的 HAL 库驱动介绍（F1）

| 驱动函数                       | 关联寄存器        | 功能描述              |
| ------------------------------ | ----------------- | --------------------- |
| HAL_RTC_Init (...)              | CRL/CRH/PRLH/PRLL | 初始化 RTC             |
| HAL_RTC_MspInit (...)           | 初始化回调        | 使能 RTC 时钟           |
| HAL_RCC_OscConfig (...)         | RCC_CR/PWR_CR     | 开启 LSE 时钟源         |
| HAL_RCCEx_PeriphCLKConfig (...) | RCC_BDCR          | 设置 RTC 时钟源为 LSE    |
| HAL_PWR_EnableBkUpAccess (...)  | PWR_CR            | 使能备份域的访问权限  |
| HAL_RTCEx_BKUPWirte/Read ()    | BKP_DRx           | 读/写备份与数据寄存器 |

需要开启的时钟源：
1. `__HAL_RCC_RTC_ENABLE()`
2. `__HAL_RCC_PWR_CLK_ENABLE()`
3. `__HAL_RCC_BKP_CLK_ENABLE()`

### RTC 基本驱动步骤（F1）
1. 使能电源时钟并使能后备域访问
	1. `__HAL_RCC_PWR_CLK_ENABLE`
	2. `__HAL_RCC_BKP_CLK_ENABLE`
	3. `HAL_PWR_EnableBkUpAccess()`
2. 开启 LSE/选择 RTC 时钟源/使能 RTC 时钟
	1. `HAL_RCC_OscConfig()`
	2. `HAL_RCCEx_PeriphCLKConfig()`
	3. `__HAL_RCC_RTC_ENABLE()`
3. 初始化 RTC，设置分频值以及工作参数
	1. `HAL_RTC_Init()`
	2. `HAL_RTC_MspInit()`
4. 设置 RTC 的日期和时间：操作寄存器方式实现 `rtc_set_time()`
5. 获取 RTC 当前日期和时间：定义 `rtc_get_time()` 

## RNG
随机数发生器（ Random Number Generators，RNG ），用于生成随机数的程序或硬件。

F1 系列只能生成伪随机数。F1 系列不具备随机数发生器 RNG。

RNG 基本驱动步骤（H7）：
1. 使能 RNG 时钟：`__HAL_RCC_RNG_CLK_ENABLE`
2. 初始化随机数发生器：
	1. `HAL_RNG_Init`
	2. `HAL_RNG_MspInit`
	3. `HAL_RCCEx_PeriphCLKConfig`
3. 判断 DRDY 位，读取随机数值 `HAL_RNG_GenerateRandomNumber`

## LOW POWER
低功耗，即降低集成电路的能量消耗。

低功耗特性（对用电池供电的产品）：
1. 更小的电池体积（大小和成本↓）
2. 延长电池寿命
3. 电磁干扰更小，提高无线通信质量
4. 电源设计更简单，无需过多考虑散热问题

### 低功耗模式
STM32 具有运行、睡眠、停止和待机四种工作模式。上电后默认是在运行模式，当内核不需要继续运行时，可以选择后面三种低功耗模式。同等条件下（T=25℃，VDD=3.3V，系统时钟 72MHz）

| 模式     | 主要影响                                           | 唤醒时间 | 供应电流（典型值） |
| -------- | -------------------------------------------------- | -------- | ------------------ |
| 正常模式 | 所有外设正常工作                                   | 0        | 51mA               |
| 睡眠模式 | CPU 时钟关闭                                       | 1.8us    | 29.5mA             |
| 停止模式 |1.8V 时钟区域关闭，电压调节器低功耗 <br> 存储器供电，程序不会复位| 5.4us    | 35uA               |
|待机模式|1.8V 时钟区域关闭，电压调节器关闭 <br>存储器断电，程序复位|50us|3.8uA|

睡眠模式配置步骤：
1. 初始化 WKUP 为中断触发源：参考外部中断引脚初始化
2. 外设低功耗处理（可选）：设置 MCU 外围外设进入低功耗
3. 进入睡眠模式：`HAL_PWR_EnterSLEEPMode`
4. 等待 WKUP 外部中断唤醒

停止模式配置步骤：
1. 初始化 WKUP 为中断触发源：参考外部中断引脚初始化
2. 外设低功耗处理（可选）：设置 MCU 外围外设进入低功耗
3. 进入睡眠模式：`HAL_PWR_EnterSTOPMode`
4. 等待 WKUP 外部中断唤醒
5. 重新设置时钟、重新选择滴答时钟源、失能 systick 中断

待机模式配置步骤：
1. 初始化 WKUP 为中断触发源（可选）：参考外部中断引脚初始化
2. 外设低功耗处理（可选）：设置 MCU 外围外设进入低功耗
3. 使能电源时钟：`__HAL_RCC_PWR_CLK_ENABLE`
4. 使能 WKUP 的唤醒功能：`HAL_PWR_EnableWakeUpPin`
5. 清除唤醒标记 WUF：`__HAL_PWR_CLEAR_FLAG`
6. 进入待机模式：`HAL_PWR_EnterSTANDBYMode`

待机模式下，所有 I/O 引脚处于高阻态，除了复位引脚、被使能的唤醒引脚等->待机模式下不能下载程序。

### STM32 电源监控
电源监控即对某些电源电压（VDD/VDDA/VBAT）进行监控。最基本的包含：
- POR/PDR（power on/down reset）：上电/掉电复位
- PVD（programmable voltage detector）：监控 VDD 电压

PVD 的使用步骤：
1. 使能电源时钟：`__HAL_RCC_PWR_CLK_ENABLE`
2. 配置 PVD 检测：通过 `HAL_PWR_ConfigPVD` 配置电压级别、中断线边沿触发
3. 使能 PVD 检测： `HAL_PWR_EnablePVD`
4. 设置 PVD 中断优先级
	1. `HAL_NVIC_SetPriority`
	2. `HAL_NVIC_EnableIRQ`
5. 编写中断服务函数
	1. `PVD_IRQHandler`
	2. `HAL_PWR_PVD_IRQHandler`
	3. `HAL_PWR_PVDCallback`

## DMA
DMA（Direct Memory Access），即直接存储器访问，DMA 传输将数据从一个地址空间复制到另一个地址空间。

DMA 传输无需 CPU 直接控制传输，也没有中断处理方式那样保留现场和恢复现场过程。通过硬件为 RAM 和 IO 设备开辟一条直接传输数据的通道，使得 CPU 的效率大大提高。

以 DMA 方式传输串口数据配置步骤：
1. 使能 DMA 时钟：`__HAL_RCC_DMA1_CLK_ENABLE`
2. 初始化 DMA： `HAL_DMA_Init` 函数初始化 DMA 相关参数，`__HAL_LINKDMA` 函数连接 DMA 和外设
3. 使能串口的 DMA 发送，启动传输：`HAL_UART_Transmit_DMA`
4. 查询传输状态
	1. `__HAL_DMA_GET_FLAG` 查询通道传输状态
	2. `__HAL_DMA_GET_COUNTER` 获取当前传输剩余数据量
5. DMA 中断使用：`HAL_NVIC_EnableIRQ`、`HAL_NVIC_SetPriority`，编写中断服务函数 `xxx_IRQHandler`

## ADC
ADC（Analog-to-Digital Converter）

常见的 ADC 类型

| ADC 电路类型 | 优点             | 缺点                     |
| ------------ | ---------------- | ------------------------ |
| 并联比较型   | 转换速度最快     | 成本高、功耗高、分辨率低 |
| 逐次逼近型   | 结构简单，功耗低 |转换速度较慢（分辨率越高，采样速率越低）|

### ADC 工作原理
#### 参考电压/模拟部分电压
- ADC 供电电源：$V_{SSA}$、$V_{DDA}$ $(2.4V \le V_{DDA} \le 3.6V)$
- ADC 输入电压范围：$V_{REF-} \le V_{IN} \le V_{REF+}$，（即 $0V \le V_{IN} \le 3.3V$）

#### 转换序列（F1 为例）
A/D 转换被组织为两组：规则组（常规转换组）和注入组（注入转换组）。规则组最多可以有 16 个转换，注入组最多有 4 个转换。注入组可以插入规则组的序列（类似中断）。

#### 触发源 (F1)
触发转换的方法有两种：
1. ADON 位触发转换（仅限 F1 系列）：当 ADC\_CR2 寄存器的 ADON 位为 1 时，再**单独**给 ADON 位写 1（只能启动规则组转换）。
2. 外部事件触发转换
	1. 规则组外部触发
	2. 注入组外部触发

#### 转换时间
如何设置 ADC 时钟？ 
PCLK2（APB2 总线上的时钟）-> ADCPRE[1:0]（RCC_CFGR 寄存器，分频系数：2/4/6/8） -> ADCCLK（ADC 最大时钟频率为 14MHz）

如何设置转换时间？
ADC 转换时间：$T_{CONV}=采样时间 + 12.5 个周期$（采样时间可通过 SMPx[2:0]位设置，x=0~17）

#### 数据寄存器
规则组（16 个）共用一个寄存器，转换结果按顺序输出至规则数据寄存器 `ADCx_DR`（32 位），独立模式只用到低 16 位。
注入组（4 个）刚好对应四个寄存器，转换结果输出至注入数据寄存器 `ADCx_JDRy (y=1~4)`（16 位）
由 `ADCx_CR2` 寄存器的 ALIGN 位设置数据对齐方式，可选择右对齐或左对齐。

#### 中断
F1/F4/F7 系列 ADC 中断时间汇总表：

| 中断事件               | 事件标志 | 使能控制位 |
| ---------------------- | -------- | ---------- |
| 规则通道转换结束       | EOC      | EOCIE      |
| 注入通道转换结束       | JEOC     | JEOCIE     |
| 设置了模拟看门狗状态位 | AWD      | AWDIE      |
| 溢出（F1 没有）        | OVR      | OVRIE           |

DMA 请求（只适用于规则组）：规则组每个通道转换结束后，除了可以产生中断外，还可以产生 DMA 请求，我们利用 DMA 及时把转换好的数据传输到指定的内存里，防止数据被覆盖。

#### 单次转换模式/连续转换模式和扫描模式
不同模式的组合作用：

| 模式组合     | 功能 |
| ------------ | ---- |
| 单次转换模式（不扫描） |只转换一个通道，而且是一次，需等待下一次触发|
|单次转换模式（扫描）|ADC_SQRx 和 ADC_JSQR 选中的所有通道都转换一次 |
|连续转换模式（不扫描）|只会转换一个通道，转换完后会自动执行下一次转换|
|连续转换模式（扫描）|ADC_SQRx 和 ADC_JSQR 选中的所有通道都转换一次，并自动进入下一轮转换|

### 单通道 ADC 采集实验
单通道 ADC 采集实验配置步骤：
1. 配置 ADC 工作参数、ADC 校准：`HAL_ADC_Init()`、`HAL_ADCEx_Calibration_Start()`
2. ADC MSP 初始化：`HAL_ADC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置 ADC 相应通道相关参数：`HAL_ADC_ConfigChannel()`
4. 启动 A/D 转换：`HAL_ADC_Start()`
5. 等待规则通道转换完成：`HAL_ADC_PollForConversion()`
6. 获取规则通道 A/D 转换结果：`HAL_ADC_GetValue()`

### 单通道 ADC 采集（DMA 读取）实验
单通道 ADC 采集（DMA 读取）实验配置步骤：
1. 初始化 DMA：`HAL_DMA_Init()`
2. 将 DMA 和 ADC 句柄联系起来：`__HAL_LINKDMA()`
3. 配置 ADC 工作参数、ADC 校准：`HAL_ADC_Init()`、`HAL_ADCEx_Calibration_Start()`
4. ADC MSP 初始化：`HAL_ADC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
5. 配置 ADC 相应通道相关参数：`HAL_ADC_ConfigChannel()`
6. 使能 DMA 数据流传输完成中断：`HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
7. 编写 DMA 数据流中断服务函数：`DMAx_Channely_IRQHandler()`
8. 启动 DMA，开启传输完成中断：`HAL_DMA_Start_IT()`
9. 触发 ADC 转换，DMA 传输数据：`HAL_ADC_Start_DMA()`

### 多通道 ADC 采集（DMA 读取）实验
在[单通道 ADC 采集（DMA 读取）实验](#单通道%20ADC%20采集（DMA%20读取）实验)基础上修改与循环扫描、引脚设置、通道数相关代码。

### 单通道 ADC 过采样实验
如何用过采样和求均值的方式提高 ADC 的分辨率？
1. 根据要增加的分辨率位数计算过采样频率方程：$f_{os}=4^{w}f_{s}$ (其中，$f_{os}$ 是过采样频率，w 是希望增加的分辨率位数，$f_{s}$ 是初始采样频率要求)
2. 求均值：例如 12 位分辨率的 ADC 提高 4 位分辨率，采样频率就要提高 256 倍，即需要 256 次采集才能得到一次 16 位分辨率的数据，然后将这 256 次采集结果求和，求和的结果再右移 4 位，就得到提高分辨率后的结果。**注意：提高 N 位分辨率，需要右移 N 位**

### 内部温度传感器实验
温度计算公式：
$$
T(℃)=(\frac{V_{25}-V_{SENSE}}{Avg_Slope})+25
$$
$V_{25}$ =25℃时的 $V_{SENSE}$ 值（典型值为：1.43）
$Avg\_Slope$ =温度与 $V_{SENSE}$ 曲线的平均斜率（单位：mv/℃或 uv/℃）（典型值：4.3mv/℃）
$V_{SENSE}$ 是 ADC 采集到内部温度传感器的电压值

### 光敏传感器实验
光敏二极管核心是一个 PN 结，对光强非常敏感，单向导电性，工作时需加**反向电压**。无光照时，反向电流很小（一般小于 0.1 微安），称为暗电流；有光照时，光的强度越大，反向电流越大，形成光电流（非线性）。利用电流变化的特点，串联一个电阻，就可以得到电压的变化，通过 ADC 读取，从而知道光强变化。

## DAC
Digital-to-Analog Converter 

### DAC 工作原理
#### 参考电压/模拟部分电压
DAC 供电电源：$V_{SSA}、V_{DDA}(2.4V \le V_{DDA} \le 3.6V)$
DAC 输出电压范围：$V_{REF-} \le V_{out} \le V_{REF+}(即 0V \le V_{OUT} \le 3.3V)$

#### DAC 数据格式
DAC 数据格式：支持 8/12 位模式
1. 8 位模式：只能右对齐 `DHR8Rx` 、`DHR8RD`（双 DAC 通道转换用）
2. 12 位模式
	1. 右对齐 `DHR12Rx`、`DHR12RD`（双 DAC 通道转换用）
	2. 左对齐 `DHR12Lx`、`DHR12LD`（双 DAC 通道转换用）

#### 触发源
三种触发转换的方式：自动触发、软件触发、外部事件触发。

#### DMA 请求
只能由外部事件触发产生 DMA 请求。

#### DAC 输出电压
12 位模式下，DAC 输出电压计算方法：
$$
DAC输出电压=(\frac{DORx}{4096}*V_{REF+})
$$
8 位模式下，DAC 输出电压计算方法：
$$
DAC输出电压=(\frac{DORx}{256}*V_{REF+})
$$

### DAC 输出实验
#### DAC 输出实验配置步骤
1. 初始化 DAC：`HAL_DAC_Init()`
2. DAC MSP 初始化：`HAL_DAC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
3. 配置 DAC 相应通道相关参数：`HAL_DAC_ConfigChannel()`
4. 启动 D/A 转换：`HAL_DAC_Start()`
5. 设置输出数字量：`HAL_DAC_SetValue()`
6. 读取通道输出数字量（可选）：`HAL_DAC_GetValue()`

### DAC 输出三角波实验
使用 `void dac_triangular_wave(uint16_t maxval, uint16_t dt, uint16_t samples, uint16_t n)` 函数。其中，maxval 为数字量（幅值），dt 为采样间隔，samples 为采样点个数，n 为波形个数。
形参取值范围 $\frac{samples}{2}\le (maxval+1)$
相邻采样点数字量之间的递增或递减幅值：$(maxval +1)/(samples/2)$

### DAC 输出正弦波实验
1. 初始化 DMA：`HAL_DMA_Init()`
2. 将 DMA 和 DAC 句柄联系起来：`__HAL_LINKDMA()`
3. 初始化 DAC：`HAL_DAC_Init()`
4. DAC MSP 初始化：`HAL_DAC_MspInit()` 配置 NVIC、CLOCK、GPIO 等
5. 配置 DAC 相应通道相关参数：`HAL_DAC_ConfigChannel()`
6. 启动 DMA 传输：`HAL_DMA_Start()`
7. 配置定时器溢出频率并启动：`HAL_TIM_Base_Init()`、`HAL_TIM_Base_Start()`
8. 配置定时器触发 DAC 转换：`HAL_TIMEx_MasterConfigSynchronization()`
9. 停止/启动 DAC 转换、DMA 传输：`HAL_DAC_Stop_DMA()`、`HAL_DAC_Start_DMA()`

产生正弦波函数：
`void dac_creat_sin_buf(uint16_t maxval, uint16_t samples)`
其中，maxval 为数字量，samples 为每个周期采样点个数。
正弦波函数解析式： $y=Asin(wx+\phi)+b$，正弦波周期：$T=2\pi/w$，对应 $y=maxval*sin((2\pi/samples)*x)+maxval$
形参取值范围： $(samples/2)<maxval$
相邻采样点的 x 轴间隔： $2\pi / samples$

### PWM DAC 实验
在精度要求不高的场合，可以使用一种廉价的解决方案实现 DAC 输出：**PWM + RC 滤波器**

根据傅里叶理论，任意周期波形都可以分解为无限个频率为其整数倍的谐波之和，将 PWM 波分段函数展开成傅里叶级数可以得到 `直流分量+1次谐波分量（基波分量）+大于一次谐波分量之和` 的形式，通过低通滤波器过滤掉谐波分量，就可以只保留直流分量，得到 PWM DAC 输出。展开式可以简化为： $f(t)=\frac{n}{N}V_H$ （直流分量从 0 到 $V_H$ 之间随 n 线性变化，DAC 分辨率= $log_{2}n$）

8 位分辨率下对 RC 滤波器的设计要求：
1. 精度要求：一般要求 1 次滤波对输出电压的影响不要超过 1 个位的精度，也就是 3.3/256=0.01289V
2. 1 次谐波最大值：假设 VH 为 3.3V，VL 为 0V，那么一次谐波的最大值是 $2*3.3/\pi=2.1V$
3. RC 滤波电路提供至少-20lg (2.1/0.01289)=-44dB 的衰减
4. 截止频率要求：当定时器的计数频率为 72Mhz，PWM DAC 分辨率为 8 位时，PWM 频率为 72M/256=281.25KHz，若是 1 阶 RC 滤波，则要求截止频率为 1.77KHz，若是 2 阶 RC 滤波，则要求截止频率为 22.43KHz。

## IIC
### IIC 介绍
IIC：Inter Integrated Circuit，集成电路总线，是一种**同步、串行、半双工**通信总线。

三个信号：
1. 起始信号（S）：当 SCL 为高电平时，SDA 从高电平变为低电平
2. 停止信号（P）：当 SCL 为高电平时，SDA 从低电平变为高电平
3. 应答信号：上拉电阻影响下 SDA 默认为高，而从机拉低 SDA 就是确认收到数据即 ACK，否则 NACK

注意：
1. 数据先发送高位
2. 数据以字节（8bit）传输
3. 数据在 SCL高电平稳定

读数据帧和写数据帧：
![](Pasted%20image%2020230303233249.png)

### AT24C02
EEPROM 是一种掉电后数据不丢失的存储器，常用来存储一些配置信息，在系统重新上电时就可以加载。AT24C02 是一个 2K bit 的 EEPROM 存储器，使用 IIC 通信方式。

AT24C02 支持字节写模式和页写模式：
- 字节写模式就是一个地址一个数据进行写入
- 页写模式就是连续写入数据。只需要写一个地址，连续写入数据时地址会自增，但有页的限制，超出一页时，超出数据覆盖原先写入的数据。但读会自动翻页。

AT24C02 支持当前地址读模式，随机地址读模式和顺序读模式：
- 当前读模式是基于上一次读/写操作的最后位置继续读出数据
- 随机地址读模式是指定地址读出数据
- 顺序读模式是连续读出数据
















## IR-Remote
红外编码常用协议：
1. NEC Protocol 的 PWM（脉冲宽度调制）：以红外载波的占空比表示 0 和 1（如 NEC 协议）
	1. 发射红外载波的时间固定，通过改变不发射载波的时间来改变占空比
2. Philips RC-5 Protocol 的 PPM（脉冲位置调制）：以发射载波的位置表示 0 和 1
	1. 从发射载波到不发射载波为 0，从不发射载波到发射载波为 1
	2. 发射载波和不发射载波的时间相同，都是 0.68ms，每位的时间都是固定的

NEC 码位定义：
1. 红外发射器
	1. 发送协议数据 0 = 发射载波信号 560us + 不发射载波 560us 
	2. 发送协议数据 1 = 发射载波信号 560us + 不发射载波 1680us
2. 红外接收器：OUT 引脚电平输出情况（接收到红外载波时，OUT 输出低电平，否则输出高电平）
	1. 接收到协议数据 0 = 560us 低电平 + 560us 高电平
	2. 接收到协议数据 1 = 560us 低电平 + 1680us 高电平

NEC 遥控器指令格式：
1. 同步码（引导码）：低电平 9ms + 高电平 4.5ms（对于接收端）
2. 地址码
3. 地址反码
4. 控制码
5. 控制反码
注意：
1. 地址码、地址反码、控制码、控制反码均是 8 位数据格式
2. 按照低位在前，高位在后的顺序发送（LSB）
3. 采用反码是为了增加传输的可靠性（可用于校验）

红外遥控驱动步骤
1. 初始化 TIM4
	1. 设置 ARR 和 PSC 等参数
	2. `HAL_TIM_IC_Init` 初始化定时器
2. 使能 TIM4 和输入通道 GPIO 时钟
	1. GPIO 配置为复用功能
	2. `HAL_GPIO_Init` / `HAL_TIM_IC_MspInit`
3. 设置输入捕获模式 / 开启输入捕获
	1. `HAL_TIM_IC_ConfigChannel`
	2. 映射关系，输入滤波和输入分频
4. 使能定时器相关中断
	1. `__HAL_TIM_ENABLE_IT` 使能更新中断
	2. `HAL_TIM_IC_Start_IT` 使能捕获中断，使能定时器
	3. `HAL_NVIC_EnableIRQ` 使能定时器中断
	4. `HAL_NVIC_SetPriority` 设置中断优先级
5. 编写中断服务函数
	1. `TIM4_IRQHandler` 定时器 4 中断服务函数
	2. `HAL_TIM_IRQHandler` 定时器中断通用处理函数
	3. `HAL_TIM_PeriodElapsedCallback` 更新中断回调
	4. `HAL_TIM_IC_CaptureCallback` 捕获中断回调

## WiFi
正点原子串口 WIFI 模块 ATK-ESP8266

## AM2302 温湿度传感器模块



