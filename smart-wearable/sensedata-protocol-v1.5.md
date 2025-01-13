# SenseData Protocol (v1.5)

## TODO

- [x] version和instance id的设计，是否要加入新的字段区分配置。仔细考虑需求。



## 版本

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



## Goal

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



## CDC通讯

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

### Command (REG) Protocol

from PC to MCU

| Preamble (8)                            | Type (2)  | Length (2) | Payload | CRC (2)    |
| --------------------------------------- | --------- | ---------- | ------- | ---------- |
| 0x55 0x55 0x55 0x55 0x55 0x55 0x55 0xD5 | 0x01 0x01 | 0x         |         | CK_A, CK_B |

overhead 14 bytes

for max86141 (1 byte instanceId  19 regs) payload length is fixed 20, total length is 20 + 14 = 34

----





