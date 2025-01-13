# Development Note

Author: matianfu@gingerologist.com

本文档不讨论第一版硬件；仅基于最新的硬件原理图和全柔板的硬件的开发过程撰写。



## Log

### Progress (as of this writing)

原开发板硬件固件调试大多完成，包括：

1. （固件）蓝牙通讯基础框架
2. （固件）程序的基础结构
3. （固件）NRF_LOG（串口，尝试过JLINK RTT，后取消；JLINK RTT效率很低，仅适合一个Task的情况）
4. （固件）USB_CDC通讯
5. （固件）多种传感器走USB_CDC时的通讯协议
6. （手机）APP完成了导航界面，程序基础UI框架，蓝牙连接部分；
7. （电脑）APP完成了除IMU之外的所有传感器的数据传输，实时绘图，数据存文件；



新产品板需要增加和优化功能：

1. （固件）支持GPIO开关机；
2. （固件）支持屏显；
3. （固件）走串口的NRF_LOG没有了，要么自己实现一个backend走USB，或者恢复JLink RTT，后者需要大面积合并当前task代码，需要尝试后才有设计决策；
4. （固件）等算法工程师提供血压算法代码合并到代码里；
5. （固件）全局内存使用优化；
6. （固件）支持蓝牙需要的Notificatoin Data，定义各种传感器数据；
7. （手机）支持通过蓝牙Notification获得各种传感器数据，在界面上绘制，其中心跳是图标，其它（可能）是单纯数字或者Gauge；
8. （电脑）尽可能可以支持之前的原始数据导出，但可能需要通过蓝牙配置；低优先级需求；



比较急迫的任务是：

1. 上电烧录（已经OK）；
2. 支持按键开机（正在做）；
3. 确定一个Debug的方式（马上做）；



### 2023-10-30

#### Set up and Debug

如果芯片完全没有烧录过固件，首次上电需要：

1. 供电，用USB通过电脑供电；
2. 连接Debug接口的3个PIN（swdio, swdclk, gnd）到nRF52 DK（这个板子是nRF52832的，不是nRF52840，但一样使用，Debug Out接口不完全一致，见Nordic官方说明）；
3. 需要Hold住Power键保证MCU供电；
4. 通过nRF Connect的Flash工具下载程序，记得第一次使用应在命令行里执行一次`nrfjprog -e`命令擦除内置的配置寄存器，否则诸如NFC，RESET PIN的设置会受影响，在工程里配置的编译选项不会生效；

```
C:\Users\matia>nrfjprog -e
Erasing user available code and UICR flash areas.
Applying system reset.
```



### 2023-11-2

#### 片上设备资源分配

nRF52840有三个通道可配置的同步串行设备，每个通道可配置为SPI或者I2C。如果设备更多也可以共享通道，使用SDK里的manager库，manager库可以用一个片上控制器轮询外设，包括每次通讯时需要重新配置管脚的情况。本项目中加速度计和LCD屏幕使用同一个twi实例。

QSPI是独立片上资源，与SPI/TWI无关。

片上设备资源使用：

- `TWI0` - QMA6100P + LCD
- `SPI1` - ADS1292R
- `SPI2` - MAX86141
- `QSPI` - Flash (not used)
- `SAADC` - Battery Level (with timer)
- `GPIOTE` - Power Button, LED
- `USB` - USB-CDC





| 序号 | 模块               | MCU管脚             | MCU资源                    |
| ---- | ------------------ | ------------------- | -------------------------- |
| 1    | ECG:ADS1292R       | ADS-DI-P0.01        | SPI instance id 1          |
|      |                    | ADS-CLK-P0.26       |                            |
|      |                    | ADS-DO-P0.27        |                            |
|      |                    | ADS-CS-P1.13        |                            |
|      |                    | ADS-DRY-P1.10       |                            |
| 2    | 加速度计：QMA6100P | QMA-SDA-P0.06       | TWI instance id 0 (shared) |
|      |                    | QMA-SCL-P0.07       |                            |
|      |                    | QMA-INT1-P0.08      |                            |
| 3    | 温度：M601Z        | M-INT-P0.31         |                            |
|      |                    | M-DQ-P0.00          |                            |
|      |                    | P0.04               |                            |
| 4    | LCD屏幕            | RES-P0.11           | TWI instance id 0 (shared) |
|      |                    | D0-SCL-P0.12        |                            |
|      |                    | D1D2-SDA-P1.09      |                            |
| 5    | 血氧血压/MAX86141  | MAX-INT-P0.24       | SPI instance id 2          |
|      |                    | MAX-SCLK-P0.14      |                            |
|      |                    | MAX-SDO-P0.13       |                            |
|      |                    | MAX-SDI-P0.15       |                            |
|      |                    | MAX-CS-P0.17        |                            |
| 6    | FLASH              | QSPI-CS-P0.18/RESET | QSPI                       |
|      |                    | QSPI-DQ1-P0.22      |                            |
|      |                    | QSPI-DQ2-P0.23      |                            |
|      |                    | QSPI-DQ3-P1.00      |                            |
|      |                    | QSPI-DQ0-P0.21      |                            |
| 7    | 锂电池检测         | VADC-P0.28/AIN4     | SAADC 1 Channel            |
| 8    | 开机使能键         | POWER-ON-P0.29      | GPIO(TE)                   |
| 9    | 关机检测管脚       | POWEROFF-DET-P0.02  | GPIO(TE), Debounce         |
| 10   | USB调试口          | D-                  | USB-CDC                    |
|      |                    | D+                  |                            |
| 11   | LED系统指示灯      | LED-P1.04           | GPIO(TE)                   |
| 12   | 程序调试           | SWDIO               | Debug                      |
|      |                    | SWDCLK              |                            |
