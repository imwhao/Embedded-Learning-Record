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
RS-232电平与CMOS/TTL电平对比：
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
3. 应用：在一些需要高稳定性的产品中，并且对时间精度要求较低的场合。
**注意：独立看门狗是异常处理的最后手段，不可依赖，应在设计时尽量避免异常。**

IWDG溢出时间计算
IWDG溢出时间计算公式：$$T_{out}=\frac{psc*rlr}{f_{IWDG}}$$
其中，$T_{out}$是看门狗溢出时间，$f_{IWDG}$是看门狗的时钟源频率，$psc$是看门狗预分频系数，$rlr$是看门狗重装载值。

IWDG配置步骤：
1. 取消PR/RLR寄存器写保护，设置IWDG预分频系数和重装载值，启动IWDG `HAL_IWDG_Init()`
2. 及时喂狗，即写入0xAAAA到IWDG_KR `HAL_IWDG_Refresh()`
