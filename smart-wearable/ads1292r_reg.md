# ADS1292R寄存器



芯片有12个寄存器，Channel1是呼吸，Channel2是心率；

| addr | name    | comment                                                      |
| ---- | ------- | ------------------------------------------------------------ |
| 0x00 | ID      | read-only                                                    |
| 0x01 | CONFIG1 | sampling rate                                                |
| 0x02 | CONFIG2 | 0xEO vs 0xA0 in kalam32.<br />same: refbuf enabled, 2.42V ref, clock output disabled, test signal off<br />diff: Lead-off comp enabled in our design. |
| 0x03 | LEADOFF | 0x10, same: +95%/-5%, 6nA, DC lead off detect                |
| 0x04 | CH1SET  | 0x40                                                         |
| 0x05 | CH2SET  | 0x60                                                         |
| 0x06 |         |                                                              |

