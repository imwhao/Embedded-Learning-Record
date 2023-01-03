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
[下载接口](https://www.bilibili.com/video/BV1bv4y1R7dp?t=1425.2&p=4)：
1. JTAG：占用5个引脚，可以下载、仿真、调试
2. SWD：占用2个引脚，可以下载、仿真、调试（推荐）
3. 串口：占用2个引脚，只能下载程序，不能调试

STM32引脚类型：电源引脚、晶振引脚、复位引脚、下载引脚、BOOT引脚、GPIO引脚。



