# 学习准备
实验基于正点原子`stm32f103精英开发板V1`。

学习资源：
1. [正点原子论坛](http://www.openedv.com/forum.php)
2. [stm32f103精英开发板V2资料下载中心](http://www.openedv.com/docs/boards/stm32/zdyz_stm32f103_jingyingV2.html)
3. [stm32f103精英开发板V1资料下载中心](http://www.openedv.com/docs/boards/stm32/zdyz_stm32f103_jingying.html)
4. [手把手教你学STM32 HAL库开发全集（新版视频）](https://www.bilibili.com/video/BV1bv4y1R7dp)
5. [手把手教你学STM32入门教学视频（老版视频）](https://www.bilibili.com/video/BV1Lx411Z7Qa/) 

开发板使用说明：
1. 连接外设时可使用资料中`精英板 IO引脚分配表.xlsx`文件查询引脚分配情况。
2. 例程测试时确保开发板B0，B1都接在GND

# 基础篇
## 程序下载
[下载接口](https://www.bilibili.com/video/BV1bv4y1R7dp?t=1425.2&p=4)：
1. JTAG：占用5个引脚，可以下载、仿真、调试
2. SWD：占用2个引脚，可以下载、仿真、调试（推荐）
3. 串口：占用2个引脚，只能下载程序，不能调试

### 串口下载
使用工具：FlyMcu
1. 搜索并选择CH340虚拟串口并设置波特率。
	1. F4系列波特率最大可以使用76800
	2. F1系列的波特率最大可以使用460800
2. 选择hex文件
3. 勾选『编程前重装文件』
4. 勾选『校验』和『编程后执行』
5. **不要**勾选『编程到FLASH时写选项字节』
6. 选择『DTR的低电平复位，RTS高电平进BootLoader』

### JLINK/STLINK下载
具体配置见[程序下载方法2：JLINK程序下载](https://www.bilibili.com/video/BV1Lx411Z7Qa/?p=9)。

## 程序调试
1. JTAG调试需要占用5个引脚。
2. SWD调试需要占用2个引脚。

JTAG/SWD调试原理：Cortex-M内核含有内核硬件调试模块，该模块可在取指（指令断点）或访问数据（数据断点）时停止。内核停止时，可以查询内核的内部状态和系统的外部状态。完成查询后，可恢复程序运行。

为了在调试期间可以使用更多GPIOs，通过设置复用重映射和调试I/O配置寄存器(AFIO_MAPR)的SWJ_CFG[2:0]位，可以改变重映像配置。具体配置参考《STM32F10xxx参考手册》 。

## 存储器映射
给存储器分配地址的过程叫存储器映射。

STM32将4G(2^32)空间平均分为8个块。

## 寄存器映射
寄存器是单片机内部一种特殊的内存，可以实现对单片机各个功能的控制。给寄存器的地址命名的过程就叫寄存器映射。

### 地址计算
寄存器地址计算过程（以GPIOA_ODR寄存器为例）：
1. 获取外设挂在哪个总线上面？查系统结构图。
2. 获取总线基地址，APB2总线基地址：0X4001 0000。
3. 获取外设地址偏移量，GPIOA相对APB2总线偏移量是：0X800。
4. 获取寄存器地址偏移量，ODR相对GPIOA外设基地址的偏移量是：0X0C。
5. 寄存器地址=BUS_BASE_ADDR+PERIPH_OFFSET+REG_OFFSET
   GPIOA_ODR = 0X4001 0000+0X800+0X0C=0X4001080C

### 映射方法
1. 宏定义直接映射，将寄存器名字映射到寄存器地址。
2. 结构体映射，将同一个外设的寄存器作为成员放在一个结构体中，成员数据类型位数与寄存器位数相同，又因为同一个结构体内存连续分配，所以只需要映射外设基地址，外设中的所有寄存器就能完成映射。
结构体映射例子：`#define GPIOA ((GPIO_TypeDef*)GPIOA_BASE)`

## HAL库工程建立
裁剪的两种方式：
1. 在`stm32f1xxx_hal_conf.h`中注释掉对应外设
2. 不将驱动文件包含到`Drivers/STM32F1xx_HAL_Driver`中。

HAL库用户配置文件`stm32f1xxx_hal_conf.h`使用：
1. 裁剪HAL库外设驱动源码（条件编译使对应需要裁剪的驱动源码不编译）
2. 设置外部高速晶振频率
3. 设置外部低速晶振频率

## stm32启动模式
### STM32启动模式（F1)
在系统复位后，SYSCLK的第4个上升沿，BOOT引脚的值会被锁存。BOOT0和BOOT1为启动模式选择引脚。不同启动模式（主闪存存储器、系统存储器、内置SRAM）将0x00000000（堆栈指针MSP初始值，即栈顶地址）和0x00000004（程序计数器PC初始值，即复位中断服务函数地址）所映射的地址不同。

### STM32CubeMX
ST开发的一款图形配置工具，可通过配置自动生成初始化代码。

## stm32时钟树
### 系统时钟树配置步骤
1. 配置HSE_VALUE：告诉HAL库外部晶振频率`stm32xxxx_hal_conf.h`
2. 调用`SystemInit()`函数（可选）：在启动文件中调用，在`system_stm32xxxx.c`定义
3. 选择时钟源，配置PLL：通过`HAL_RCC_OscConfig()`函数设置
4. 选择系统时钟源，配置总线分频器：通过`HAL_RCC_ClockConfig()`函数设置
5. 配置扩展外设时钟（可选）：通过`HAL_RCCEx_PeriphCLKConfig()`函数设置
正点原子中调用3+4+5=`sys_stm32_clock_init()`

### 外设时钟使能和失能
**我们要使用某个外设，必须先使能该外设时钟！**（为降低功耗，外设时钟默认失能）

```c
__HAL_RCC_GPIOA_CLK_ENABLE(); /*使能GPIOA时钟*/
__HAL_RCC_GPIOA_CLK_DISABLE(); /*禁止GPIOA时钟*/ 
```

## SYSTEM文件夹介绍
正点原子提供的常用函数。其中子文件夹包括：
### sys 
包括中断类函数、低功耗类函数、设置栈顶指针函数、**系统时钟初始化函数**、Cache配置函数（F7/H7）
### delay 
初始化系统滴答定时器、微妙/毫秒延时函数：
	1. 初始化滴答定时器时声明了一个全局变量`g_fac_us=sysclk/分频数`，代表1$\mu s$计时器的自减值。（参数`sysclk`单位为MHz）
由系统滴答计时器实现微秒延时函数。（不超频情况下最大能够实现约1.8s的微妙级延时）
用微秒延时函数实现毫秒延时函数。
### usart
printf函数输出流程：
1. printf()用户调用。
2. printf函数由编译器提供的stdio.h解析。
3. fputc()最终实现输出（用户需要根据最终输出的硬件重新定义该函数，此过程称为printf重定向）。

printf函数支持：
1. 避免使用半主机模式（仿真器在电脑上输入输出）。两种方法：
	1. 微库法：在魔术棒target中勾选Use MicroLIB，实现简单，速度慢，兼容性差。
	2. 代码法：需要实现一个预处理、两个定义、三个函数，实现复杂，速度快，兼容性好。（`usart.c`文件中实现，直接复制即可）
2. 实现fputc函数。

# 入门篇
## GPIO
### GPIO需要熟悉的知识
GPIO(General Purpose Input Output)作用：
1. 输入（采集外部器件信息）
2. 输出（控制外部器件工作）

GPIO电气特性：
1. 电压工作范围 2V至3.6V
2. GPIO识别电压范围：CMOS端口（不带FT）：-0.3至1.164V 0,1.833至3.6V 1。
3. GPIO输出电流：单个IO最大25mA

STM32引脚类型：电源引脚、晶振引脚、复位引脚、下载引脚、BOOT引脚、GPIO引脚。

GPIO的8种工作模式：
1. 输入浮空
2. 输入上拉
3. 输入下拉
4. 模拟输入
5. 开漏输出
	1. 不能输出高电平，只能输出0或高阻态，除非有内部或外部的上拉
6. 推挽输出
	1. 可以输出0和1
	2. 驱动能力强
7. 开漏式复用功能
8. 推挽式复用功能

GPIO寄存器介绍：
1. 端口配置低寄存器(GPIOx_CRL) (x=A..E)
2. 端口配置高寄存器(GPIOx_CRH) (x=A..E)
3. 端口输入数据寄存器(GPIOx_IDR) (x=A..E)
4. 端口输出数据寄存器(GPIOx_ODR) (x=A..E)
5. 端口位设置/清除寄存器(GPIOx_BSRR) (x=A..E)
6. 端口位清除寄存器(GPIOx_BRR) (x=A..E)
7. 端口配置锁定寄存器(GPIOx_LCKR) (x=A..E)

ODR寄存器控制输出和BSRR寄存器控制输出的区别：ODR可读可写，寄存器赋值的步骤是读、改、写，BSRR只可写，控制输出直接写。使用ODR在读和修改访问之间产生中断时可能发生风险，BSRR无风险。

### 通用外设驱动模型
四步法：
1. 初始化：时钟设置、参数设置、IO设置、中断设置（开中断、设NVIC）（可选）
2. 读函数（可选）
3. 写函数（可选）
4. 中断服务函数（可选）

#### GPIO配置步骤：
1. 初始化
	1. 使能时钟`__HAL_RCC_GPIOx_CLK_ENABLE()`
	2. 设置工作模式`HAL_GPIO_Init()`
2. 设置输出状态（可选）`HAL_GPIO_WritePin()`、`HAL_GPIO_TogglePin()`
3. 读取输入状态（可选）`HAL_GPIO_ReadPin()`

## 中断
中断的作用：
1. 实时控制：在确定时间内对相应事件作出响应。
2. 故障处理：检测到故障需要第一时间处理。
3. 数据传输：不知道数据什么时候来，如串口数据接收。
高效处理紧急程序，不会一直占用CPU资源。

### NVIC
Nested vectored interrupt controller，嵌套向量中断控制器，属于内核。
NVIC支持：256个中断（16内核+240外部），支持256个优先级，允许裁剪。（ST公司裁剪为16个）

stm32中断优先级基本概念：
1. 抢占优先级（pre）：高抢占优先级可以打断正在执行的低抢占优先级中断。
2. 响应优先级（sub）：当抢占优先级相同时，响应优先级高的先执行，但是不能互相打断。
3. 自然优先级：抢占和响应都相同的情况下，自然优先级（中断向量表的优先级）越高的，先执行。

stm32中断优先级分组：通过配置`AIRCR[10:8]`对`IPRx bit[7:4]`的四个位进行抢占优先级数目和响应优先级分配。

STM32 NVIC的使用：
1. 设置中断分组`AIRCR[10:8], HAL_NVIC_SetPriorityGrouping`
2. 设置中断优先级`IPRx bit[7:4], HAL_NVIC_SetPriority`
3. 使能中断`ISERx, HAL_NVIC_EnableIRQ`

### EXTI
External(Extended) interrupt/event Controller，外部（扩展）中断事件控制器。包含20个产生事件/中断请求的边沿检测器，即总共：20条EXTI线（F1）。

EXTI寄存器：
1. 中断屏蔽寄存器(EXTI_IMR)
2. 事件屏蔽寄存器(EXTI_EMR)
3. 上升沿触发选择寄存器(EXTI_RTSR)
4. 下降沿触发选择寄存器(EXTI_FTSR)
5. 软件中断事件寄存器(EXTI_SWIER)
6. 挂起寄存器(EXTI_PR) 

AFIO：Alternate Function IO，即复用功能IO，主要用于重映射和外部中断映射配置（F1）：
1. 调试IO配置：AFIO_MAPR[26:24]，配置JTAG/SWD的开关状态
2. 重映射配置：AFIO_MAPR，部分外设IO重映射配置
3. 外部中断配置：AFIO_EXTICR1~4，配置EXTI中断线0~15对应到哪个具体IO口。
特别注意：配置AFIO寄存器之前要使能AFIO时钟，方法如下：`__HAL_RCC_AFIO_CLK_ENABLE();对应RCC_APB2ENR寄存器 位0`

中断的产生：
1. EXTI中断
	1. GPIO设置输入模式。
	2. AFIO（F1）或SYSCFG（其他）设置EXTI和IO映射关系。
	3. EXTI。
	4. NVIC：设置中断分组、优先级、使能。
	5. CPU：按优先级顺序，依次处理中断。
2. 外设中断
	1. USART/TIM/SPI...
	2. NVIC：设置中断分组、优先级、使能。
	3. CPU：按优先级顺序，依次处理中断。

STM32 EXTI的配置步骤（GPIO外部中断）：
1. 使能GPIO时钟
2. 设置GPIO输入模式
3. 使能AFIO/SYSCFG时钟
4. 设置EXTI和IO对应关系
5. 设置EXTI屏蔽，上/下沿
6. 设置NVIC
7. 设计中断服务函数
注：HAL库步骤2~5使用`HAL_GPIO_Init`一步到位。

STM32 EXTI的HAL库配置步骤（GPIO外部中断）：
1. 使能GPIO时钟`__HAL_RCC_GPIOx_CLK_ENABLE`
2. GPIO/AFIO(SYSCFG)/EXTI `HAL_GPIO_Init`
3. 设置中断分组`HAL_NVIC_SetPriorityGrouping`，**此函数只需要设置一次**
4. 设置中断优先级`HAL_NVIC_SetPriority`
5. 使能中断`HAL_NVIC_EnableIRQ`
6. 设计中断服务函数`EXTIx_IRQHandler`，中断服务函数，清中断标志
   STM32仅有EXTI0~4、EXTI9_5、EXTI15_10，7个外部中断服务函数

HAL库中断回调处理机制介绍：
1. 中断服务函数
2. HAL库中断处理公用函数
3. HAL库数据处理回调函数（函数名以Callback结尾）

## 串口通信
串口：即串行通信接口，指按位发送和接收的接口。如：RS-232/422/485等。

常见的串行通信接口：

| 通信接口                     | 接口引脚                                                     | 数据同步方式 | 数据传输方向 |
| ---------------------------- | ------------------------------------------------------------ | ------------ | ------------ |
| UART<br />（通用异步收发器） | TXD:公共端<br />RXD：接收端<br />GND：公共地                 | 异步通信     | 全双工       |
| 1-wire                       | DQ:发送/接收端                                               | 异步通信     | 半双工       |
| IIC                          | SCL:同步时钟<br />SDA:数据输入/输出端                        | 同步通信     | 半双工       |
| SPI                          | SCK:同步时钟<br />MISO:主机输入，从机输出<br />MOSI:主机输出，从机输入<br />CS:片选信号 | 同步通信     | 全双工       |

### RS232
RS-232 电平与 CMOS/TTL 电平对比：

| 类型       | 逻辑1     | 逻辑0     |
| ---------- | --------- | --------- |
| RS232      | -15V至-3V | +3V至+15V |
| CMOS(3.3V) | 3.3V      | 0V        |
| TTL(5V)    | 5V        | 0V        |
结论：CMOS/TTL电平不能与RS232电平直接交换信息。


### USART
Universal synchronous asynchronous receiver transmitter，通用同步异步收发器。
Universal asynchronous receiver transmitter，通用异步收发器，USART裁剪调同步功能。

F1波特率计算公式：$$baud=\frac{f_{ck}}{16*USARTDIV}$$

USART/UART异步通信配置步骤：
1. 配置串口工作参数 `HAL_UART_Init()`
2. 串口底层初始化 `HAL_UART_MspInit()`配置GPIO、NVIC、CLOCK等
3. 开启串口异步接收中断 `HAL_UART_Receive_IT()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 编写中断服务函数 `USARTx_IRQHandler()`、`UARTx_IRQHandler()`
6. 串口数据发送 寄存器 `USART_DR` HAL库 `HAL_UART_Transmit()`

## IWDG
IWDG简介：
- IWDG全称：Independent watchdog，即独立看门狗。
- IWDG本质：能产生**系统复位信号**的计数器。
- IWDG特性：独立的计数器，时钟由独立的RC振荡器提供（可在待机和停止模式运行），看门狗被激活后，当递减计数器计数到0x000时产生复位。
- 喂狗：计数器计到0之前，重装载计数器的值，防止复位。

IWDG作用：
1. 异常：外界电磁干扰或自身系统（硬件或软件）异常，造成程序跑飞，如：陷入某个不正常的死循环，打断正常的程序运行。
2. IWDG作用：主要用于检测外界电磁干扰，或硬件异常导致的程序跑飞问题。
3. 应用：在一些需要高稳定性的产品中，并且对时间精度要求较低（RC振荡器精度较低）的场合。
**注意：独立看门狗是异常处理的最后手段，不可依赖，应在设计时尽量避免异常。**

IWDG溢出时间计算
IWDG溢出时间计算公式：$$T_{out}=\frac{psc*rlr}{f_{IWDG}}$$
其中，$T_{out}$是看门狗溢出时间，$f_{IWDG}$是看门狗的时钟源频率，$psc$是看门狗预分频系数，$rlr$是看门狗重装载值。

IWDG配置步骤：
1. 取消PR/RLR寄存器写保护，设置IWDG预分频系数和重装载值，启动IWDG `HAL_IWDG_Init()`
2. 及时喂狗，即写入0xAAAA到IWDG_KR `HAL_IWDG_Refresh()`

## WWDG
Window watchdog，即窗口看门狗。
WWDG简介：
- WWDG本质：能产生**系统复位信号**和**提前唤醒中断**的计数器。
- WWDG特性：
	- 递减的计数器。
	- 当递减计数器值从0x40减到0x3F时复位。
	- 计数器的值大于W[6:0]值时喂狗会复位。
	- 提前唤醒中断（EWI）：当递减计数器等于0x40时可产生。
- 喂狗：在窗口期内重装载计数器的值，防止复位。
	- $W[6:0]\ge窗口期>0x3F]$

WWDG作用：监测单片机运行时效是否精准，主要检测**软件异常**。
WWDG应用：需要精准监测程序运行时间的场合。

WWDG超时时间计算：$$T_{out}\frac{4096*2^{WDGTB}*(T[5:0]+1)}{F_{wwdg}}$$
其中，$T_{out}$是WWDG超时时间（没喂狗）；$F_{wwdg}$是WWDG的时钟源频率；4096是WWDG固定的预分频系数；$2^{WDGTB}$是WWDG_CFR寄存器设置的预分频系数值；T[5:0]是WWDG计数器低6位。

WWDG配置步骤：
1. WWDG工作参数初始化：`HAL_WWDG_Init()`
2. WWDG Msp初始化：`HAL_WWDG_MspInit()`配置NVIC、CLOCK等
3. 设置优先级，使能中断：`HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
4. 编写中断服务函数：`WWDG_IRQHandler()`->`HAL_WWDG_IRQHandler`
5. 重定义提前唤醒回调函数：`HAL_WWDG_EarlyWakeupCallback()`
6. 在窗口期喂狗：`HAL_WWDG_Refresh()`

IWDG 和 WWDG 对比：

| 对比点         | 独立看门狗                   | 窗口看门狗                       |
| -------------- | ---------------------------- | -------------------------------- |
| 时钟源         | LSI（40KHz或32KHz）          | PCLK1或PCLK3                     |
| 复位条件       | 递减计数到0                  | 计数值大于W[6:0]值喂狗或减到0x3F |
| 中断           | 没有中断                     | 计数值减到0x40可产生中断         |
| 递减计数器位数 | 12位（最大计数范围：4096~0） | 7位（最大计数范围：127~63）      |
| 应用场合       | 防止程序跑飞，死循环，死机   | 检测程序时效，防止软件异常                                 |

## TIMER
### 定时器概述
软件定时原理：使用纯软件（CPU死等）方式实现定时（延时）功能。

软件延时缺点：
1. 延时不精准
2. CPU死等

定时器定时：使用精准的时基，通过硬件的方式，实现定时功能。

stm32定时器分类：
1. 常规定时器
	1. 基本定时器
	2. 通用定时器
	3. 高级定时器
2. 专用定时器
	1. 独立看门狗
	2. 窗口看门狗
	3. 实时时钟
	4. 低功耗定时器
3. 内核定时器 SysTick定时器

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
	1. 16位递增计数器（计数值：0~65535）
	2. 16位预分频器（分频系数：1~65536），分频系数为寄存器值+1
	3. 可用于触发DAC

定时器中断实验相关寄存器（F1）：
1. TIM6和TIM7控制寄存器1（TIMx_CR1）
	1. 位7 ARPE（Auto-reload preload enable)
		1. 0：TIMx_ARR寄存器没有缓冲（对寄存器写入后直接生效）。
		2. 1：TIMx_ARR寄存器具有缓冲（更新事件发生后才写入影子寄存器生效）。
		3. 作用：在微妙级计时前后计时时间发生变化时，等待更新事件发生后对寄存器的写入会耗费一定时间，当具有缓冲时可以在更新时间发生前写入寄存器，增加计时精度。
	2. 位0 CEN（Counter Enable）
2. TIM6和TIM7 DMA/中断使能寄存器（TIMx_DIER）
3. TIM6和TIM7状态寄存器（TIMx_SR） 用于判断是否发生了更新中断，由硬件置1，软件清零
4. TIM6和TIM7计数器（TIMx_CNT）计数器值可读可写
5. TIM6和TIM7预分频器（TIMx_PSC）分频系数为PSC值+1

定时器溢出时间的计算方法：$$T_{out}=\frac{(ARR+1)*(PSC+1)}{F_t}$$
其中，$T_{out}$是定时器溢出时间，$F_t$是定时器的时钟源频率，$ARR$是自动重装载寄存器的值，$PSC$是预分频器寄存器的值。（ARR+1是因为当设置ARR为0时也要计一个数）

定时器中断实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_Base_Init()`
2. 定时器基础MSP初始化 `HAL_TIM_Base_MspInit()`配置NVIC、CLOCK等
3. 使能更新中断并启动计数器 `HAL_TIM_Base_Start_IT()`
4. 设置优先级，使能中断 `HAL_NVIC_SetPriority()`、`HAL_NVIC_EnableIRQ()`
5. 编写中断服务函数 `TIMx_IRQHandler()`等->`HAL_TIM_IRQHandler()`
6. 编写定时器更新中断回调函数 `HAL_TIM_PeriodElapsedCallback()`

### 通用定时器
#### 通用定时器简介（F1）
1. 通用定时器TIM2/TIM3/TIM4/TIM5
2. 主要特性（加粗为相对基本定时器的新特性）
	1. 16 位递增、**递减、中心对齐**计数器（计数值：0~65535）
		1. 递增、递减计数器又称边沿对齐计数器
	2. 16位预分频器（分频系数：1~65536）
	3. 可用于触发DAC、**ADC**
	4. 在更新时间、**触发事件、输入捕获、输出比较**时，会产生中断/DMA请求
	5. **4个独立通道，可用于：输入捕获、输出比较、输出PWM、单脉冲模式**
	6. **使用外部信号控制定时器且可实现多个定时器互连的同步电路**（定时器的级联）
	7. **支持编码器和霍尔传感器等**

计数器时钟源：
1. 内部时钟（CK_INT），来自外部总线APB提供的时钟（是否x2需要看预分频器）
2. 外部时钟模式1：外部输入引脚（TIx），来自定时器通道1或者通道2引脚的信号
3. 外部时钟模式2：外部触发输入（ETR），来自可以复用为TIMx_ETR的IO引脚
4. 内部触发输入（ITRx），用于与芯片内部其它通用/高级定时器级联

#### 通用定时器PWM输出实验
通用定时器输出PWM原理：
假设采用递增技术模式，ARR为自动重装载寄存器的值，CCRx为捕获/比较寄存器x的值，当CNT<CCRx，IO输出0，当CNT>=CCRx，IO输出1。此时，PWM波周期或频率由ARR决定，PWM波占空比由CCRx决定。

通用定时器PWM输出实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_PWM_Init()`
2. 定时器PWM输出MSP初始化 `HAL_TIM_PWM_MspInit()`配置NVIC、CLOCK、GPIO等
3. 配置PWM模式/比较值等 `HAL_TIM_PWM_ConfigChannel()`
4. 使能输出并启动计数器 `HAL_TIM_PWM_Start()`
5. 修改比较值控制占空比（可选） `__HAL_TIM_SET_COMPARE()`
6. 使能通道预装载（可选）`__HAL_TIM_ENABLE_OCxPRELOAD()`

PWM周期/频率的计算方法同定时器溢出时间的计算方法：$$T_{out}=\frac{(ARR+1)*(PSC+1)}{F_t}$$
其中，$T_{out}$是定时器溢出时间，$F_t$是定时器的时钟源频率，$ARR$是自动重装载寄存器的值，$PSC$是预分频器寄存器的值。（ARR+1是因为当设置ARR为0时也要计一个数）

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
1. 确定 $t_{DTS}$ 的值：$f_{DTS}=\frac{F_t}{2^{CKD[1:0]}}$，其中 CKD[1:0]为时钟分频因子，这 2 位定义在定时器时钟（CK_INT）频率、死区时间和由死区发生器与数字滤波器（ETR, Tlx）所用的采样时钟（$t_{DTS}$）之间的分频比例。
2. 判断 DTG[7:5]，选择计算公式。
3. 代入选择的公式计算。

高级定时器互补输出带死区控制实验配置步骤：
1. 配置定时器基础工作参数 `HAL_TIM_PWM_Init()`
2. 定时器 PWM 输出 MSP 初始化 `HAL_TIM_PWM_MspInit()`
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
| ------- | --------- | --------- | ------------------------------------- |
| CS      | 片选      | 低电平    | 选中器件，低电平有效，先选中，后操作  |
| WR      | 写        | ↑         | 写信号，上升沿有效，用于数据/命令写入 |
| RD      | 读        | ↑         | 读信号，上升沿有效，用于数据/命令读取 |
| RS      | 数据/命令 | 0=命/1=数 | 表示当前是读写数据还是命令，也叫 DC 信号      |
| D[15:0] | 数据线    | 无        | 双向数据线，可以写入/读取驱动 IC 数据                                      |

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

修改 `usmart_port.c/.h` 即可完成移植，修改 `usmart_config.c` 即可添加自己想要调用的函数。






