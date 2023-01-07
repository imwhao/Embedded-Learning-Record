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

## JLINK/STLINK下载
具体配置见[程序下载方法2：JLINK程序下载](https://www.bilibili.com/video/BV1Lx411Z7Qa/?p=9)。

## 程序调试
1. JTAG调试需要占用5个引脚。
2. SWD调试需要占用2个引脚。

JTAG/SWD调试原理：Cortex-M内核含有内核硬件调试模块，该模块可在取指（指令断点）或访问数据（数据断点）时停止。内核停止时，可以查询内核的内部状态和系统的外部状态。完成查询后，可恢复程序运行。

为了在调试期间可以使用更多GPIOs，通过设置复用重映射和调试I/O配置寄存器(AFIO_MAPR)的SWJ_CFG[2:0]位，可以改变重映像配置。具体配置参考《STM32F10xxx参考手册》 。




