# 软件设计文档

## 文档

本文档简述多极片项目的软件设计。用户不必阅读此文档，软件工程师如果想在现有设计上做修改应阅读本文档。

文档作者/硬件设计：马工（matianfu@gingerologist.com, 个人常用邮箱）



## 开发工具

软件使用意法半导体官方开发工具，STM32CubeIDE开发；赚些本文档时版本为1.15.0，该软件频繁自动升级，但兼容性良好。

> 需要注意的是，该IDE提供了一个双向代码生成工具，可以图型化配置处理器资源，引脚使用，Clock Tree，Middleware（包括RTOS）等等。该配置文件不是编译项目必须的，但开发者一直使用该工具配置项目。配置文件的名称是multipads.ioc，该文件的版本可能随着IDE升级而升级，如果升级到更高版本，低版本的IDE软件可能会无法打开。



## 功能范围

1. 可以从USB/串口通讯，接收和执行用户发送的指令；

2. 通过用户发送的指令定义1-9个开关切换的profile；

3. 可以响应10个按键
   1. 按键1-9的功能是启动用户自定义的某个profile；
   2. 按键0在上电后，如果用户未曾按1-9的任意一个按键，是启动自检（blink）功能，可多次按；如果用户曾经按过1-9的任何一个按键启动自定义profile，则该按键执行停止当前profile功能；
   
4. 包含一个自检（blink）模式，依次打开关闭所有LED；可通过按键和命令触发；

   

## RTOS任务结构

因为响应按键，处理用户命令，和控制开关是同时的，尤其控制开关有时间要求，故使用了FreeRTOS。在ioc配置里配置。

实际使用2个任务，ux任务处理用户输入包括按键响应，profile任务执行profile，profile任务使用一个queue通讯，消息是一个`profile_t`结构体，后述。



## 命令行实现

ux任务里提供了命令行实现，包括line editor（用户可以用backspace修改错误输入），使用了一个第三方库，embedded_cli，具体版本和链接在代码中有注释。



## Profile

`profile_t`是profile的定义，也是profile任务接收的消息格式。

```c
typedef struct {
	uint32_t pgcfg_a[4];
	uint32_t pgcfg_b[4];
	uint32_t duration_a_sec;
	uint32_t duration_b_sec;
} profile_t;
```

成员变量名称中的a和b指a阶段（phase a）和b阶段（phase b）。



每个profile包含两个phase，在执行时交替轮换；每个phase有一个pad group configuration数组，数组的index 0-4对应硬件的四个pad连接器，每个连接器有9个pad，对应18个开关，uint32_t的config数据里低18bit是开关的开启和关闭状态；

- bit 0是pad 1接com的开关，bit 1是pad 1接signal的开关；设置为1打开，0关闭。
- bit 2是pad 2接com的开关，bit 3是pad 2接signal的开关；设置为1打开，0关闭，依次类推，共使用18个bit。



duration_x_sec的单位是秒，指该阶段执行的时间，可以为0，如果为0则停滞在这个阶段一直到收到下一个profile消息。



举例：

1. 如果要仅用一个phase，可以只定义a的profile，并把duration_a_sec设置为0；此时phase b的设置已经无关；
2. 如果使用两个phase，则a和b的duration都要给一个有限且小于3600（1小时）的值，这样才能切换；
3. 如果phase的duration是有限值，phase b的是0，这个profile不在功能要求至内，但也可以执行，即先执行phase a的设置，最后停在phase b的设置上，直到下一个profile消息到来；但这个模式可能也有一个价值就是可用实现一个有timeout的任务。



在用户角度理解，目标板的行为有开始和停止的概念，而实现上没有这个概念，profile任务一定在执行一个profile，停止是通过执行一个全设置为0的profile实现的，全设置为0时所有开关关闭，因为duration为0，任务进入无限等待状态，等待下一个profile消息到来，效果上这就是用户理解的停止了。



当前的Profile设计简单而且健壮，利用FreeRTOS queue提供的xQueueReceive实现时间等待和任务中断，这样不管用户如何使用按键和命令启动任务，任务之间都不会发生冲突，也不强制用户必须完成一个任务后再启动一个任务。



### Blink

Blink的时序要求无法用Profile定义实现；所以在启动Blink时发送给profile任务一个特殊的数据，把profile的所有uint32_t变量都填充了`0xdeadbeef`，收到该消息后，profile任务走特殊的执行路径实现该功能。



## 重要文件

- `embedded_cli.h`和`embedded_cli.c`来自第三方，未修改；
- `command.c`是ux任务；
- `profile.c`是profile任务；
- `main.c`和`main.h`是主任务，在使用图形化工具修改ioc文件配置后，生成的代码会修改这两个文件；代码生成器在源码中定义了一些用户代码块，但尽管如此，仍应尽可能避免在这两个文件里添加和修改自定义代码，最好是把代码都写在其它文件里。
  - uxTask的执行函数配置成weak（没有extern选项），在main.c里留下一个空函数，实际生效的是command.c里的版本；
  - profileTask的执行函数被配制成extern，这样全部代码都在profile.c里；



## 命令格式

参见用户手册



## 已知问题

参见用户手册



## 源代码

交付的源码包是完整的工程字目录，包括配置和编译后的文件都没有清除，解压后应该可以直接导入STM32CubeIDE中。



