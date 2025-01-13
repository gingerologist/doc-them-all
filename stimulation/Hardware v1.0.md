# Hardware Design Document (v1.0)

Author: Ma (matianfu@gingerologist.com)

| version    | data       | comment                     |
| ---------- | ---------- | --------------------------- |
| v1.0 alpha | 2023-10-27 | initial draft               |
| v1.0 beat  | 2023-11-19 | add (schematic) design note |





## Requirement

### Power

| Power Net | Source                         | Comment                                            |
| --------- | ------------------------------ | -------------------------------------------------- |
| GND       | input                          | Ground                                             |
| 5V_USB    | input                          | USB input                                          |
| 5V_DAC    | input                          | For DAC input                                      |
| 3V3       | LDO                            | for MCU, DAC digital logic                         |
| +15       | boost                          | for opamp and analog switch, positive power supply |
| -15       | (boost) + charge pump inverter | for opamp and analog switch, negative power supply |





### Key Parts and Alternatives

|      | Category   | Manfuacturer | Parts                                                        | Package                  | LCSC code               | LC Stock  |
| ---- | ---------- | ------------ | ------------------------------------------------------------ | ------------------------ | ----------------------- | --------- |
|      | Connector  |              |                                                              | USB-C                    |                         |           |
|      | RX Power   | Renesas/IDT  | [P9222-RAZGI8](https://item.szlcsc.com/1602790.html)         | WLCSP-40 (0.4mm BGA)     | C1511994                | low       |
| *    | Boost      | TI           | [LM27313XMFX/NOPB](https://item.szlcsc.com/318997.html) [LM27313XMF/NOPB](https://item.szlcsc.com/33319.html) | SOT-23-5                 | C341750, C32355         | yes       |
| *    | LDO        | TI           | [TPS70933DRVR](https://item.szlcsc.com/176546.html)          | DFN 2x2                  | C165164                 | yes       |
| *    | MCU        | Noridc       | [NRF52832-CIAA-R](https://item.szlcsc.com/523264.html)       | WLCSP50 (3.0x3.2, 0.4mm) | C509405                 |           |
|      | MCU        | Nordic       | [NRF52832-QFAA-R](https://item.szlcsc.com/78669.html)        | QFN48 (6x6)              | C77540                  |           |
| *    | Octal DAC  | TI           | [DAC43608RTER](https://item.szlcsc.com/2771349.html)         | QFN16 (3x3)              | C2679432                | low       |
|      | Qctal DAC  | TI           | DAC43508                                                     | QFN16 (3x3)              | ti only                 | no        |
|      | Octal DAC  | TI           | [DAC088S085CIMTX/NOPB](https://item.szlcsc.com/476166.html)  | TSSOP16 (5.0x4.4)        | C469914                 | low       |
|      | Octal DAC  | TI           | [DAC088S085CISQ/NOPB](https://item.szlcsc.com/2743575.html)  | QFN16 (4x4)              | C2651666 (ti)           | no        |
|      | Quad DAC   | TI           | [DAC084S085CISD/NOPB](https://item.szlcsc.com/202998.html)   | WSON10 (3x3)             | C201674                 | high      |
|      | Octal DAC  | Analog       | [AD5318ARUZ-REEL7](https://item.szlcsc.com/531492.html)      | TSSOP16 (5.0x4.4)        | C515875                 | high      |
|      | Octal DAC  | Analog       | [AD5328BRUZ-REEL7](https://item.szlcsc.com/29914.html)       | TSSOP16 (5.0x4.4)        | C29162                  | high      |
| *    | Quad OPAMP | ST           | [TL084IPT](https://item.szlcsc.com/3362421.html), [TL084CPT](https://item.szlcsc.com/2744188.html) | TSSOP14 (5.0x4.4)        | C2969957, C2652279      | high, low |
|      | Quad OPAMP | Analog       | [ADTL084ARZ-REEL7](https://item.szlcsc.com/609158.html)      | TSSOP14 (5.0x4.4)        |                         |           |
|      | Quad OPAMP | Analog       | [ADA4062-4ARUZ-RL](https://item.szlcsc.com/683552.html)      | TSSOP14 (5.0x4.4)        | C653998                 |           |
|      | Quad OPAMP | Analog       | [ADA4062-4ACPZ-R7](https://item.szlcsc.com/529351.html)      | LFCSP16 (4x4)            | C514310                 |           |
|      | Octal SPST | Analog       | [ADG1414BCPZ-REEL7](https://item.szlcsc.com/207776.html)     | LFCSP24 (4x4)            | C206656                 | high      |
| *    | Quad SPST  | Analog       | [ADG1412YCPZ-REEL7](https://item.szlcsc.com/139928.html)     | LFCSP16 (4x4)            | C128643                 |           |
|      | Quad SPST  | Vishay       | [DG1412EEN-T1-GE4](https://item.szlcsc.com/222754.html)      | QFN16 (4x4)              | C222459 (preorder)      |           |
|      | Quad SPST  | TI           | [TMUX6212PWR](https://item.szlcsc.com/3155368.html)          | TSSOP14 (5.0x4.4)        | C2902526                |           |
|      | Quad SPST  | TI           | [TMUX6212RUMR](https://item.szlcsc.com/4225068.html)         | WQFN16 (4x4)             | C3658208 (preorder, ti) |           |

说明

- Wirless Charger
  - P9222是瑞萨的方案，考虑该方案的原因是它具有和Transmitter交换数据的能力；
- Boost
  - 两个LM27313是相同型号包装不同，XMF后缀是1000颗盘装，XMFX后缀是3000颗，立创分为两个Part。
- LDO & MCU
  - TPS70933 LDO规格为30V输入，3.3V输出，150mA LDO；
  - 采用高压输入允许使用P9222供电时调高输入电压；
  - 使用3.3V输出因为nRF系列DK版的板载debug out只能调试3.3V（实际上是3V）的目标板，如果使用1.8V的目标板则需要使用JLink；
  - MCU采用WLCP50封装是考虑如果使用P9222，会支持0.4mm BGA和0.15/0.25盘中孔；
- DAC
  - DAC的理想是
    - DAC088S085CISQ/NOPB，4x4mm封装，8通道，4.5uS，立创标价太贵，无库存；
    - DAC43508/43608，3x3mm封装，8通道，10uS，立创备货量很低；
  - v1设计采用DAC088的TSSOP封装型号或两颗DAC084，主要是考虑DAC08系列settling time低；如果最终布板面积要求高于settling time要求，可替换为
  - Analog的AD5308/5318/5328均为TSSOP封装，如果接受该封装也可以考虑；
- OPAMP
  - 需要+/-15V的版本，TL084系列；从ST的TL084 CPT开始；
  - TI的TL084H是+/-20V的版本，有DYY封装（4.2x3.3），立创无货只能从TI订；
- SPST 
  - Analog ADG1412YCPZ；因为有所谓Burst Mode支持，多簇，没办法使用1414型号，否则可有更好的



## PCB

### BGA Land Pattern

For P9222, NSMD with 0.2mm pad size is used as recommended in datasheet. Soldermask 0.28mm, this leaves 0.12mm (4.7mil) for solder mask width, which is slightly larger than the technology limit (4mil) for jlc.

0.3mm/0.2mm is choosed for VIP (via in pad) size. This could be changed to 0.25/0.15mm if required.

> https://www.jlc.com/portal/vtechnology.html

**Notice** Pads with VIP are essentially SMD pad with 0.3mm pad and 0.28mm mask.



For nRF52832-CIAA, which is also a 0.4mm BGA package, the same NSMD pad and mask size are used, aka, 0.20mm/0.28mm. Nordic provides a reference design for two layer pcb, in which no microVia or VIP are used. We follow this design.



### Zones

- Clearance 0.15mm
- Minimal Width 0.2mm
- Thermal relief gap 0.2mm
- Thermal spoke width 0.25mm

TPS61046

TPS61170 wide input

TLV76733 3.3V wide input ldo 3.3V

TLV76750 5.0V wide input ldo 5.0V
