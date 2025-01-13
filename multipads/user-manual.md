# Multipads产品用户手册

## 文档说明

本文档为多极片（Multipads）产品的用户使用手册，包含硬件和软件。

文档：马工（matianfu@gingerologist.com, 个人常用邮箱）

日期：2024年4月



## 产品功能简述

Multipads产品用于把信号发生器产生的交直流信号，以用户可配置的方式，加载到多个电极片上；板载4组连接器，每组可连接9个电极片，总计可同时连接36个电极片。

- 每个极片均可独立配置接信号（signal），接公共端（com），或不连接（floating）；
- 支持手动和程控两种方式控制每个极片的连接；
- 用户可以配置最多9个自定义的profile，每个profile可以实现：
  - 设置全部36个极片的连接方式；
  - 设置两个阶段（phase a和phase b）交替工作，指定每个阶段里全部36个极片的连接方式和工作时间；
- 通过USB连接电脑，在电脑上定义profile；
- 定义好profile的硬件可脱离电脑工作，仅使用按键启动和停止某个profile；
- 支持上电后的自动检测（blink）；



## 使用说明

![multipads_board](/data/github/doc-them-all/multipads/images/multipads_board.png)

### 输入输出接口，供电，烧录，按键

- PCB上部的绿色接线端子（J10）接信号发生器产生的信号；
- PCB上部的四个黑色IDC连接器（J1-J4）接4组极片；在程序里，从左至右标识为A/B/C/D四组（即J4是A）；
- PCB左侧有黑色桶形电源连接器（2.5mm/5.5mm规格）和USB连接器，两者均可给PCB供电，也可以同时供电；板载电路会自动选择电压高的供电方且保证电源之间不会倒灌电流；
- PCB上电后电源连接器附近的发光二极管（D1）会亮；
- USB连接器可以连接电脑，可以不使用外接电源，电脑USB口的供电是足够的。推荐使用USB 3.0接口因为有900mA的供电电流保证，USB 2.0的电流保证500mA，理论上也是够的；
- PCB下部的J9连接器为JTAG接口，推荐使用ST-LINK V2/V3，JLink或兼容的烧录器均可使用；烧录器仅在烧录控制器固件的时候需要连接，其它使用场景不需要连接；
- 在JTAG接口右侧有个很小的开关（SW1）为reset开关，可以直接重启芯片，避免插拔电源；
- 在控制芯片右侧有10个按键开关，标识1-9和0；



### 拨动开关

板上中间一排有36个拨动开关，板上丝印标注从左至右分为ABCD 4组，每组从1到9编号。

每个开关都有三个位置（双刀三掷开关）。开关拨动到中央时，该路极片的开关状态受处理器程序控制。如果拨到上侧，则该极片接Signal，如果拨到下侧，接COM。这样用户可以简单实现手动设置开关，不需要修改固件或者用PC端更新配置。

无论手动控制还是处理器自动控制，拨动开关上下的LED都忠实显示PAD的连接状态；上侧LED亮表示PAD接Signal，下侧亮表示接COM，均不亮为不接。



### 自检

刚刚上电的板子，无论是否设置过按键1-9的功能，都可以用按键0触发自检模式，可以看到所有的LED都依次亮灭。



### 按键功能定义，PuTTY

用户可以用PC连接Multipads，通过PuTTY软件定义Multipads的按键功能。



PuTTY是一个Windows上的免费终端软件，可以在[putty.org](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)上下载安装。在撰写本文档时，最新版本是0.81。



使用USB线将Multipads板连接电脑，在Windows的设备管理器中找到Windows给Multipads分配的COM口名称，例如COM8。

<img src="/data/github/doc-them-all/multipads/images/com_port.png" alt="com_port" style="zoom:100%;" />



打开PuTTY软件，根据电脑上实际分配的COM口修改接口名称；把Speed修改为115200。

![putty_configured](/data/github/doc-them-all/multipads/images/putty_configured.png)



点击Open按键，应该看到PuTTY的命令窗口；先按一下回车触发显示`>`提示符，然后输入help，回车，可以看到命令帮助。如果点击Open时PuTTY没有反应，应去Windows设备管理器检查COM口，该COM口的编号是动态分配的，每次可能不同。

![putty_screenshot_01](/data/github/doc-them-all/multipads/images/putty_screenshot_01.png)



### 配置命令

- help命令显示帮助信息；
- blink命令可以触发Multipads自检（和上电后按键0功能一样）；
- list命令显示当前Profile 1-9的设置；
- define命令定义Profile；

#### list命令

list命令显示如下，如果板子没有被配置过，所有的Profile都是0，duration也是0。

![putty_screenshot_02](/data/github/doc-them-all/multipads/images/putty_screenshot_02.png)



#### Profile

在定义profile之前，用户需要简单理解一下微处理器是如何执行用户的定义的profile的。

每个profile包含两个阶段，phase a和phase b；每个phase都包含4组9个Pad的连接设置，用一个9字符的字符串表示一组Pad的设置，0表示不连接，1表示接COM，2表示接Signal；每个phase包含一个duration，表示该阶段持续的时间，单位为秒，duration可以为0，0表示该phase不会结束。



微处理器执行profile的方式是：

1. 先应用phase a的开关设置，然后等待phase a的duration时间；
1. 如果phase a duration为0，则停留在该状态，直到用户下一次按键；
2. 如果phase a duration不为0，开关状态保持该时间后，应用phase b的开关设置；
   1. 如果phase b duration为0，则停留在该状态，直到用户下一次按键；
   2. 如果phase b duration不为0，开关状态保持该时间后，再次应用phase a的开关设置，如此无限循环，直到用户下一次按键；



产品目前有两种需求：

(a) 设置为一个固定的开关模式，不切换；实现该需求可以仅定义phase的四组开关设置，定义phase a duration为0；这种情况下phase b可以是任何设置，微处理器仅执行前述的步骤1和2，与phase b的设置无关；

(b) 设置a/b循环模式；实现该需求可以定义phase a和phase b，均提供四组开关设置，且duration均不为0；这种情况向下微处理器会执行前述步骤的1，3，3.2；



如果用户需要指定(a)的执行时间也是可以的，即phase a的duration不为0；此时可以把phase b设置为全部是0，全部是0等于自动停止，因为所有开关都关闭了并进入等待用户操作状态，这种情况下微处理器的执行路径是1，3，3.1。该模式不在需求列表中，但用户如果希望使用这样的定时模式也是可以的。但目前的profile设计不支持给a/b循环模式设置一个总的执行时间（或循环次数），如果需要该功能可以联系开发者修改固件程序。



**注意**：目前**duration的上限值是3600**（即1小时），hardcode在代码里，如果用户需要更大的上限值可以联系开发者修改。



#### define命令

define命令的格式是：

```
define xx yyyyyyyyy yyyyyyyyy yyyyyyyyy yyyyyyyyy z
```

其中xx指定profile的编号（1-9）和阶段（phase a or b）；四个yyyyyyyyy字符串定义ABCD四组开关的设置；z是phase持续的时间，单位为秒，最大允许值3600，即1小时。

该格式是一个严格格式，不可缺少任何一个参数或者数据格式不符合要求，如果无法解析会显示错误原因。

下图显示定义Profile 8a的例子。

![putty_screenshot_03](/data/github/doc-them-all/multipads/images/putty_screenshot_03.png)



在输入define命令后，会有大约3秒钟的延迟，是程序把配置写入flash保存的原因；保存到flash内的配置在下次系统上电时会自动载入，这样PCB在配置好之后可以完全脱离电脑使用。



**注意**：在define命令结束前不要拔掉电源，有可能导致flash内数据丢失，这不会损坏芯片，但需要用户重新输入所有Profile定义；





### 使用

配置好profile之后，用户只需要同通过按键使用Multipads硬件。按键1-9触发响应的profile。

按键0具有两个功能；在上电之后，如果用户从未按过1-9按键，则按键0会触发blink命令执行自检，可以执行多次自检；如果用户按过1-9按键，则按键0的功能是停止当前执行的profile。

没有配置的profile（phase的配置全部为0的）等同于停止功能。



执行任何profile时都可以按键立刻切换到其它profile或者停止，包括自检，不需要用户等待某个操作完成。



## 已知问题

| No   | Domain   | Description                                                  | Cause                                                        | Solution                                           |
| ---- | -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| 1    | software | 命令行良好支持退格键, 但不保证良好支持其它控制键, 上下左右方向键, Home/End/PageDown/PageUp, Ctrl C/Ctrl V等组合键，输入这些按键有可能产生特别慢的反应甚至死机； | MCU上不会实现特别复杂的行编辑器功能                          | 不会修复，用户尽量避免在写命令的时候使用这些按键； |
| 2    | hardware | 未烧录过的板子A4/B5对应的Signal LED在上电后会亮，在烧录时这两个LED也会亮 | 这两个LED的控制信号使用了JTAG接口的引脚，无固件或烧录时引脚被拉高； | 不会修复，不影响使用                               |

