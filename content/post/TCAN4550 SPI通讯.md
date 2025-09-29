---
title: "TCAN4550 SPI通讯"
date: 2023-12-18T15:13:29+08:00
description: ''
tags: ["CANFD", "CAN", "SPI"]
categories: ["STM32"]
slug: "TCAN4550-SPI-STM32"
draft: false
---

# TCAN4550 SPI通讯

TCAN4550，SPI 转 CANFD 的 CAN卡。使用 STM32F4 通过 SPI 进行收发测试，记录遇到的一些问题。

### 0.逻辑分析仪数据不固定

使用逻辑分析仪分析的时候发现有时的数据都不固定，原因是采样率不够，采样率需要是SPI频率的5-10倍。最后我降了SPI时钟（来自APB2分频）后，数据固定。

以及第一帧数据有`the inital(idle) state of the CLK line does not natch the setting` 提醒，配置CPOL 和 CPHA 以及MSB/LSB等。

![](../images/TCAN4550_SPI/image_19xwjCBvlB.png)

### 1.清除SPI err flag

TCAN4x5x\_Device\_ClearSPIERR(); // Clear any SPI ERR flags that might be set as a result of our pin mux changing during MCU startup

调用了

&#x20;AHB\_WRITE\_32(0x0c, 0xFFFFFFFF);       // Simply write all 1s to attempt to clear a SPIERR that was set

```c
void
AHB_WRITE_32(uint16_t address, uint32_t data)
{
    AHB_WRITE_BURST_START(address, 1);  // 61 00 0c 01
    AHB_WRITE_BURST_WRITE(data); // ff ff ff ff 
    AHB_WRITE_BURST_END();
}

```

从机未开机时：数据正常

![](../images/TCAN4550_SPI/image_RJvNBbJNAu.png)

从机开机，数据则不对。发现是因为引脚连接的问题。如果双方都有MOSI MISO 标识，则MOSI接MOSI，MISO接MISO即可。但是从机上只有SDI 和 SDO。我直接把主机MOSI接SDO，MISO接SDI了。其实是错的，按名称MOSI（主发从收）应该接从机的SDI（串行输入），MISO接SDO。接了后就好了

### 2.初始化问题

用demo把最近的发送加上去了，初始化+发送。但是一直没有反应，也不清楚是哪一块的配置问题。就在初始化中输出排查，发现 从5 这里就一直失败，得RESET才能再次设置

![](../images/TCAN4550_SPI/image_Cnm68QZ0Tm.png)

查了好久都没进展。去github上搜了一下，打开一个demo项目发现底下赫然出现的一行字

![](../images/TCAN4550_SPI/image_6pW9xdsWxN.png)

便去检查了下共地，（最开始加了，在调试过程中拔掉忘记了）。把GND接上，再发送则成功。

在DEMO中也有说明：

```Text
 *  Pinout
 *   - P1.4 SPI Clock / SCLK
 *   - P1.6 MOSI / SDI
 *   - P1.7 MISO / SDO
 *   - P2.5 SPI Chip Select / nCS
 *
 *   - P2.3 MCAN Interrupt 1 / nINT
 *   - Ground wire is important
 *
```

![](../images/TCAN4550_SPI/image_2KathaS-DE.png)

![](../images/TCAN4550_SPI/image_gUqN70bvRn.png)

### 3.接收数据

MCU设置引脚外部中断，连接CAN卡的GPO2引脚。

初始化修改 `devConfig.GPO2_CONFIG = TCAN4x5x_DEV_CONFIG_GPO2_MCAN_INT0;`

![](../images/TCAN4550_SPI/image_7hg6kuRcKx.png)

中断处理中计数

```c
volatile uint8_t TCAN_Int_Cnt = 0;          // A variable used to keep track of interrupts the MCAN Interrupt pin

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  switch(GPIO_Pin){
      case CAN_INT_Pin:
        TCAN_Int_Cnt++;
        break;
  }
}
```

使用demo代码接收数据即可。

### 4.总结

关键在连线上：

```Text
mcu         can卡

SCLK        SCLK
MISO        SDO
MOSI        SDI
片选         nCS
外部中断     GPO2
GND         GND

```

然后用demo代码就可以了。

如果用逻辑分析仪看SPI波形，需要注意配置。

**参考：**

-   [CLK线的初始（空闲）状态与设置不匹配：the inital(idle) state of the CLK line does not natch the setting](https://blog.csdn.net/m0_49005156/article/details/108476593 "CLK线的初始（空闲）状态与设置不匹配：the inital(idle) state of the CLK line does not natch the setting")
-   [TCAN4550: Problem in reading register from SPI](https://e2e.ti.com/support/interface-group/interface/f/interface-forum/914947/tcan4550-problem-in-reading-register-from-spi "TCAN4550: Problem in reading register from SPI")
-   [逻辑分析仪解析SPI数据](https://blog.csdn.net/lllmeimei/article/details/128408494 "逻辑分析仪解析SPI数据")
-   [SPI接口的MISO和MOSI如何连接](https://zhuanlan.zhihu.com/p/639642940 "SPI接口的MISO和MOSI如何连接")
