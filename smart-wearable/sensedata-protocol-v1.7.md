# SenseData Protocol (v1.7)

## Document Version

维护者：马 matianfu@gingerologist.com

| 日期       | 版本 | 变更                                                         |
| ---------- | ---- | ------------------------------------------------------------ |
| 2023-07-15 | 1.0  | draft，不稳定版本，定义了Packet，TLV，MAX86141的Global Message，部分寄存器配置和FIFO数据格式； |
| 2023-07-29 | 1.1  | 增加ads1292r定义；                                           |
| 2023-08-03 | 1.2  | 修改ads1292r的数据包格式，增加算法库产生的数据；修改global tlv的名称为brief tlv，以避免和nodejs的global变量名冲突；修改每包样本数缺省值，25->50；在brief tlv里提供样本数，心律和呼吸字段； |
| 2023-09-11 | 1.3  | 增加温度传感器定义；                                         |
| 2023-09-25 | 1.4  | 增加Rougu的Spo2数据定义；                                    |
| 2023-11-10 | 1.5  | 增加合并abp和spo2的max86141定义；                            |
| 2023-12-01 | 1.6  | 移除合并的spo2和abp定义，修改spo2和abp定义；                 |
| 2023-12-20 | 1.7  | 增加蓝牙接口协议                                             |



## Objectives

本文档定义nRF52840平台上的各种传感器和PC之间的传输协议；最初的目标包括：

1. Analog/Maxim MAX86141（血氧+脉搏）
1. TI ADS1292R（呼吸+心律）
3. QMA6110p（IMU）
4. m601z（体温）



一些设计原则

- 协议原则上做中立设计，不依赖MCU类型；
- 假设目标系统包含多种传感器类型和同一类型多个传感器实例；
- 协议设计的主要目的是获取数据和调优算法，在性能和通讯带宽允许的情况下提供尽可能多的业务相关的传感器配置数据；协议不假设和配置目标设备的行为模式，例如是否使用中断，是否使用低功耗模式等等；应用可以自己拓展
- 使用灵活的格式，例如TLV（Type-Length-Value）；
- 使用大包，使用每包设置，使用二进制数据，使用简化的CRC校验；
- 必须支持单向的data logging，即mcu不响应任何命令，只依据缺省设置输出数据；
- 可选支持双向通讯；支持如下操作：
  - 查询系统上有哪些传感器
  - 查询每个传感器支持几种配置
  - 可选择某个传感器的某个配置



Endianness：所有多字节整数都是little endian的；



## CDC (MCU to PC)

### CDC Packet

CDC Packet是USB通讯时，MCU固件向PC发送传感器数据的包格式。

| Preamble (8 bytes)                      | Type (2bytes) | Length (2 bytes)        | Payload (variable) | CRC (2 bytes)                  |
| --------------------------------------- | ------------- | ----------------------- | ------------------ | ------------------------------ |
| 0x55 0x55 0x55 0x55 0x55 0x55 0x55 0xD5 | 0x01 0x01     | uint16_t, little endian | array of TLV data  | CK_A (uint8_t), CK_B (uint8_t) |

1. `Type`固定为`0x0101`，表示传感器数据，目前无其它数据类型定义；
2. `Length`是Payload长度，不含CRC；`Length`为`0`的空包合法，例如用于Heart Beat；
3. CRC计算不包含`Preamble`，包含`Type`，`Length`，`Payload`；计算公式如下（来自u-blox UBX协议）：

```C
uint8_t buffer[N]; // N bytes of data
uint8_t CK_A = 0; 
uint8_t CK_B = 0;
for (int i = 0; i < N; i++)
{
	CK_A = CK_A + buffer[i];
	CK_B = CB_B + CK_A;
}
```



### Payload and Type-Length -Value

`Payload`由一个或多个`TLV`组成。



| Type (1 bytes) | Length (2 bytes) | Value (variable length) |
| -------------- | ---------------- | ----------------------- |
| uint8_t        | uint16_t         | uint8_t[]               |

第一个`TLV`的类型必须是`0xff`，称`Brief` TLV；不同的传感器的`Brief`不同，可理解为多态数据；Brief TLV的头部格式如下：

| type (1) | length (2) | sensor id (2) | instance id (1) | version (1) |
| -------- | ---------- | ------------- | --------------- | ----------- |
| 0xff     | -          | -             | -               | -           |

目前定义的所有传感器数据，`version`字段均为0， `instance id`字段有使用；但`instance id`和`version`的定义未最终敲定；未来也可能抛弃`version`把`instance id`和`version`解释成一个字段。对于项目开发而不是目标开放系统，这样设计已经足够。

> 协议最初设计是试图仅可能开放的，但实际的传感器的可配置的工作能力千差万别，广泛兼容的驱动结构对于单一项目来是过度设计；简化的设计，即使兼容旧设备，也可以用`instance id`的方式区分。



### MAX86141 (sensor id: 1)

#### Brief

| type | length     | sensor id (2) | instance Id (1) | Version (1) |
| ---- | ---------- | ------------- | --------------- | ----------- |
| 0xff | 0x04, 0x00 | 0x01, 0x00    | spo2(0), abp(1) | 0x00        |

#### Other TLV

| Type | Length                                  | Value                                             | implemented |
| ---- | --------------------------------------- | ------------------------------------------------- | ----------- |
| 0x08 | variable                                | fifo data                                         | y           |
| 0x10 | 7, [6 (0x10 - 0x15) + 1]                | PPG configuration + Picket fence                  | y           |
| 0x20 | 12, [3 + 9]                             | LED Seq Control + PA                              | y           |
| 0x2C | 6 (single) or 12 (dual)                 | PPG1 DAC                                          | n           |
| 0x41 | 2                                       | die temperature                                   | n           |
| 0xe7 | 16                                      | abp coefficient (4 float, 2 for sbp, 2 for dbp)   | y           |
| 0xe8 | 16                                      | feature data (1 feature per second, configurable) | y           |
| 0xe9 | 50 (SPO samples) * 24 (rougu data size) | third party algorithm data                        | y           |

#### 实际使用的包格式

- spo and abp cfg packet: brief + ppgcfg + leccfg
- spo samples: brief + spo samples + rougu data, 50 samples (100 items, ir1, red1) per packet
- abp samples: brief + abp samples, 256 samples (512 items, ir1, ir2) per packet
- abp coeff: brief + coeff



### ADS1292R (sensor id: 2)

#### Brief

| Type | Length     | Sensor Id (2) | Instance Id (1) | Version (1) | Num of Samples (1) | Heart Rate (1) | Respiratory Rate (1) |
| ---- | ---------- | ------------- | --------------- | ----------- | ------------------ | -------------- | -------------------- |
| 0xFF | 0x07, 0x00 | 0x02, 0x00    | 0x00            | 0x00        | 50 (default)       | 0xff           | 0xff                 |

#### Others

| Type | content                                                      | Length (N=50) | Comment                                 |
| ---- | ------------------------------------------------------------ | ------------- | --------------------------------------- |
| 0x00 | 12 register values                                           | 12            | All regs                                |
| 0x10 | N raw samples (status, chan1, chan2) = N * 9                 | 450           | RDATAC                                  |
| 0x80 | N ecg (uint16_t) = N * 2                                     | 100           | rog filtered ecg                        |
| 0x81 | N/10 Out_Signal1 (uint32_t, 4), BRHPFilter_Y_OUT (uint32_t, 4) | 40            | rog processed breath (2-stage filtered) |

#### PC文件存储可选字段（obsolete）

每个发送给PC的包包含50个原始样本，每个原始样本包含status（Lead Off状态），通道1（呼吸），通道2（心电），原始数据未数字滤波，有明显的工频干扰。Lead Off状态根据配置，仅包括通道1和通道2，不包括RLD。

- 全部12个寄存器状态，每包一组12个；
- 原始status，含Lead Off状态，每包50个；
- 通道1数据，每包50个；
- 通道2数据，每包50个；
- ZR算法低通滤波后的心电数据；每包50个，uint16_t格式（精度）；
- ZR算法先低通滤波后的数据lp，再高通滤波后数据hp，uint32_t格式（精度）；每包5个，该算法是选点完成的，每10个点里选一个，因为原始采样率250sps，选点等于是25sps，该操作等价于已经去掉了12.5Hz以上的频率；
- ZR算法计算得到的heart rate；
- 需计算得到的respiratory rate，ZR算法目前未提供；原则上该值应该是FFT后的基频；
  - 另：ZR提供的寄存器配置未使用芯片的调制接调功能，不清楚该调值接调功能对性能的影响，ti手册上有优化配置调值解调和相位优化的描述；
- 其它算法中使用但未输出的数据
  - qrs detect返回的delay，具体含义未知，是大约100左右的整数值；
  - qrs detect累加的qrs count，这个应该是检测到的qrs计数，根据原始代码，每包数据处理后清零（但原始算法一包是80个数据点，我们目前是50个）；
  - 其它算法使用的内部数据，winPeak，filterData，sbPeak，含义未知；
  - 以上数据如果需要输出可以输出；qrs count可以比较持续使用移动窗口的情况（即每包数据无穷大），和现在的情况，每50个点处理后清零，初步比较没看到明显区别。

### 1-Wire Temperature (sensor id: 4)

#### Brief (8 bytes, minimal)

| Type (1) | Length (2) | Sensor Id (2) | Instance Id (1) | Version (1) | Num (1) |
| -------- | ---------- | ------------- | --------------- | ----------- | ------- |
| 0xFF     | 0x0, 0x00  | 0x04, 0x00    | 0x00            | 0x00        | 0-8     |

#### Others

| Type | Content          | Length | Comment |
| ---- | ---------------- | ------ | ------- |
| 0x00 | [ID, TEMP] pairs | 80     | -       |

ID长度64bit，8 bytes；TEMP 2 Bytes，如果temp读取失败，0xff 0xff。每个传感器10bytes，任意排序，实际最多8个，即Length最大80。



### Generic ADC (sensor id: 0xFFF0, Obsolete)

#### Brief

- 采样率（Sampling Rate，`uint16_t`）
- 采样率单位（Hz，kHz，MHz，`uint8_t`）
- 样本位宽（Bit Depth，`uint8_t`）
- 参考电压（VREF，mV单位，`uint16_t`）
- 样本数量（uint16_t）

| Type | Length     | Sensor Id (2) | Version (1) | Instance Id (1) | SR (2) | SR Unit (1) | BD (1) | VRef (2) | Sample Num (2) |
| ---- | ---------- | ------------- | ----------- | --------------- | ------ | ----------- | ------ | -------- | -------------- |
| 0xFF | 0x0C, 0x00 | 0xf0, 0xff    | 0x00        | 0x00            |        |             |        |          |                |

Example: SAADC on nRF51

| Type | Length     | Sensor Id (2) | Version (1) | Instance Id (1) | SR (2) | SR Unit (1) | BD (1) | VRef (2)   | Sample Num (2) |
| ---- | ---------- | ------------- | ----------- | --------------- | ------ | ----------- | ------ | ---------- | -------------- |
| 0xFF | 0x0C, 0x00 | 0xf0, 0xff    | 0x00        | 0x00            | 250    | 0           | 14     | 600 (0.6V) | 120            |


#### Others

| Type | Content                                                      | Length (N=50) | Comment  |
| ---- | ------------------------------------------------------------ | ------------- | -------- |
| 0x00 | 2 (uint16_t) x 120                                           | 240           |          |
| 0xf0 | not defined yet, may be configuration for nRF52840, such as<br />gain, acquisition time, oversample, etc. |               | All regs |

```
N = 120, 0x00

Brief len  = 3 + 12 	=  15
local 00   = 3 + 240 	= 243
                        = 258 (payload)
               + 14    	= 272 (packet)
```

## CDC (PC to MCU)

| Preamble (8)                            | Type (2)  | Length (2) | Payload | CRC (2)    |
| --------------------------------------- | --------- | ---------- | ------- | ---------- |
| 0x55 0x55 0x55 0x55 0x55 0x55 0x55 0xD5 | 0x01 0x01 | 0x         | -       | CK_A, CK_B |

- see `APP_USBD_CDC_ACM_USER_EVT_RX_DONE` handler in `usbcdc.c`.

- see `handle_command` function, where
  - type == 2, no argument for `get-abp-coeff`, triggering max task coeff packet
  - type == 3 && length == 16 for `set-abp-coeff`.

This getter/setter design simulates a register and simplifies implementation.

## BLE (MCU to Mobile App)

### BLE Packet

BLE使用NUS底层，使用Type-Seq-Data格式，因为蓝牙按包发送，接收端按包接受。

| type (1)                                | sequence (1) | data (variable) |
| --------------------------------------- | ------------ | --------------- |
| temp(1), ecg(2), spo(3), abp(4), adc(5) | 未实际使用   | -               |

### Body Temperature (m601z)

每个传感器数据结构大小10字节，data大小是传感器数量x10，包大小再加2。

```C
typedef struct __attribute__((packed)) ow_m601z_id_temp
{
    uint8_t id[8];
    uint8_t temp[2];
} ow_m601z_id_temp_t;
```

### ECG (ads1292r)

包结构的等价C语言定义如下：

```C
typedef struct ble_ecgdata
{
    unsigned char type;	// 2
    unsigned char sequence; // not used
    unsigned char heart_rate;
    unsigned char rsvd1;
    unsigned char rsvd2;
    unsigned char rsvd3;
    int			  sample[10]; // 5 sample average
} ble_ecgdata_t;
```

其中sample是5点平均值；heart_rate用计算得到的最后一个值填充（即50个样本的最后一个值），和PC一致。

### SPO (max, 0)

```c
// deprecated
typedef struct ble_spodata // packed 
{
	unsigned char type; 			// 3
    unsigned char sequence; 		// not used
    uint16_t      saO2_avg;			// 50 value avg
    uint32_t      hr_avg;			// 50 value avg
    uint32_t      ir1[10];			// 5 sample avg
    uint32_t      rd1[10];  		// 5 sample avg
} ble_spodata_t; 					// 90 bytes in total, 1 per second

// from code
typedef struct __attribute__((packed)) max_ble_spo_pac
{
    uint16_t len;
    uint8_t type;           // 3
    uint8_t seq;            // not used
    uint32_t saO2;          // 50 values avg
    uint32_t heartRate;     // 50 values avg
    uint32_t ir1[10];       // 5 values avg
    uint32_t rd1[10];       // 5 values avg
} max_ble_spo_pac_t;        // 90 bytes in total (packet size), 1 packet per second
```

### ABP (max, 1)

```C
/**
 * for abp, sampling rate is 2048 and each packet contains 256 samples, which means 
 * 8 packets per second. 1 of them have feature / bp data.
 * for ble, we have 2 samples per packet.
 */
typedef struct __attribute__((packed)) max_ble_abp_pac
{
    uint16_t    len;
    uint8_t     type;           // 4
    uint8_t     seq;            // not used
    int         sbp;            // -1 for invalid
    int         dbp;            // -1 for invalid
    uint32_t    ir1[2];         // 128-avg
    uint32_t    ir2[2];         // 128-avg
} max_ble_abp_pac_t;
```



### ADC

```C
typedef struct __attribute__((packed)) max_ble_adc_pac
{
    uint16_t    len;
    uint8_t     type;           // 5
    uint8_t     seq;            // not used
	float		voltage;		// ieee754 little endian on Cortex M4
} ble_adc_pac_t;
```



### IMU

type + seq + 7 bytes (start from reg 0x00), see qma6110p datasheet.



## Critial Bug Report and Fix

### ECG Error

#### 现象

ECG心律值偏高

#### 分析方法

1. 加入硬件timer，每秒钟fire一次；在DRDY的中断函数里加入atomic counter递增计数；在timer里打印计数并清零；目的是检查中断次数是否和设置的250Hz一致；
2. 在DRDY中断函数（`ads1292r_drdy_handler`）中打印xQueueReceive错误的情况；

#### 代码工作方式说明

ECG使用DRDY中断触发SPI数据读取，在DRDY中断里需要从IDLE队列里分配内存，在SPI读取完成后将内存压入PENDING队列；ECG任务则从PENDING队列里取数据处理，处理结束后把内存归还给IDLE队列。

#### 测试结果

1. 中断计数多数情况下是250次，偶见249次，即频率比较接近250Hz预设值，测量误差在可接受范围内。
2. 在DRDY中断函数中看到了xQueueReceive错误打印，表明<span style="color:red">中断处理函数中存在内存申请错误，发生错误时内存块数量为8</span>。

#### BUG原因

在任务较多的情况下，ECG任务需要等待较多的时间才能被调度执行处理PENDING队列里的传感器数据；如果等待时间较长，所有内存块都被使用并推入PENDING队列后，如ECG任务仍未能及时处理，DRDY中断就无法分配到内存了，会导致数据点丢失。

#### FIX

FIX的方式是增加内存块数量到16，在代码中定义为`NUM_OF_RECORDS`宏。

#### 验证方式

目前代码中已经不使用NRF_LOG输出，改用SEGGER_RTT，需安装Jlink RTT Viewer看到打印，该软件是免费使用的，直接使用调试器接口，即SWD的烧录接口工作，不需要额外的串口，USB，或者其它信号（包括SWO）。

打开JLink RTT Viewer可以看到，如果NUM_OF_RECORDS定义为8，有下面的代码的打印输出，如果设置为16则看不到打印输出，如果未来继续增加功能导致再次看到该输出则需要继续增加内存块数量。

```
793		SEGGER_RTT_printf(0, "---- ecg xqecv err ----\r\n");
```



## 呼吸数据说明（20240318）

源码里rdatac_record_t数据结构包含了完整的9字节原始数据，该原始数据也直接发送给了PC客户端；



第三方算法是选点（每10个点）后两次滤波的，JavaScript代码里写的lp和hp对应低通滤波和高通滤波，低通滤波的结果是中间结果，可不用。

