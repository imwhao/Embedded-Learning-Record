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

STM32引脚类型：电源引脚、晶振引脚、复位引脚、下载引脚、BOOT引脚、GPIO引脚。

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

# HAL库的使用
裁剪的两种方式：
1. 在`stm32f1xxx_hal_conf.h`中注释掉对应外设
2. 不将驱动文件包含到`Drivers/STM32F1xx_HAL_Driver`中。

HAL库用户配置文件`stm32f1xxx_hal_conf.h`使用：
1. 裁剪HAL库外设驱动源码（条件编译使对应需要裁剪的驱动源码不编译）
2. 设置外部高速晶振频率
3. 设置外部低速晶振频率