代码里包含很多测例，会找时间逐一从`howland.c`文件中剥离出来并加以注释。



在撰写此文档时，重要的测列包括test20, test21和test9a。前面两个尚在howland.c文件中，未剥离，test9a已经是独立的test case。



其中一些设计决策如下：



in the sample code, the function `test91_spi_xfer()`

```c
static void test9a_spi_xfer()
{
    uint32_t err;

    // spi start
    static uint8_t data[16] = {
        0x60, 0x00,
        0x64, 0x00,
        0x68, 0x00,
        0x6c, 0x00,
        0x6f, 0xf0,
        0x6c, 0x00,
        0x68, 0x00,
        0x64, 0x00,
    };
    
    static nrf_drv_spi_xfer_desc_t xfer = {
        .p_tx_buffer = data,
        .tx_length = 2
    };

    const uint32_t flags =
        NRF_DRV_SPI_FLAG_HOLD_XFER |
        NRF_DRV_SPI_FLAG_TX_POSTINC |
        NRF_DRV_SPI_FLAG_NO_XFER_EVT_HANDLER |
        NRF_DRV_SPI_FLAG_REPEATED_XFER;

    err = nrf_drv_spi_xfer(&m_dac_spi, &xfer, flags);
    APP_ERROR_CHECK(err);
}
```



this function starts a series spi xfer in asynchronous mode. Noticing the flags:

- `NRF_DRV_SPI_FLAG_HOLD_XFER`  assures the xfer is triggered in next ppi event, instead of being triggered immediately. This is for the timing precision.
- `NRF_DRV_SPI_FLAG_REPEATED_XFER` means the xfer will be triggered several times by ppi event and `NRF_DRV_SPI_FLAG_TX_POSTINC` says in each trigger, new data pair is used.
- `NRF_DRV_SPI_FLAG_NO_XFER_EVT_HANDLER` suppresses the event handler, saving CPU time.



`test9a_spi_xfer` is called in `test9a_cycle_timer_callback`. Cycle timer, in this test, is configured as a periodical timer with a period of 4 seconds. In each cycle, th count timer counts how many times the burst timer.

 The burst timer is the 'atom' constructing the full timing diagram. It uses two channels. Channel 0 triggers spi xfer (on hold), channel 1 triggers ss pin and count timer.

- channel 1 使用fork同时trigger count和ss pin；
- 使用channel 1 trigger sspin在channel 0 trigger spi之后因为后面这个trigger有较大的延迟，trigger ss pin只需要一个总线时钟延迟。
- count timer会停止burst timer，但是在停止的时候，ss pin和spi xfer都已经trigger了，即当前这个周期的spi xfer会完成；下一个周期不会开始了；
- ss pin的rising edge是spi xfer end触发的，和burst timer或count timer均无关；



在test9a里，每一个周期在周期结束时由callback重新启动。

```C
static void test9a_cycle_timer_callback(nrf_timer_event_t event_type, void * p_context)
{
    NRF_LOG_INFO("cycle timer fired");

    test9a_spi_xfer();
    nrf_drv_timer_resume(&m_count_timer);
    nrf_drv_timer_resume(&m_burst_timer);
}
```



callback中有3个动作，重新调用`nrf_drv_spi_xfer`启动一串传输，恢复count timer和启动burst timer；



首先count timer的resume不是必须的，可以把count timer配制成CLEAR MASK而不是STOP MASK，这样就不用每次从新启动；



其次恢复burst timer是启动一连串的spi xfer的触发点，如果需要尽可能精确的工作，有一个timer用ppi触发burst timer即可。



test9a已经和接近一个可以工作的全功能模板。但它的一个关键缺失是，连续8次写入只能实现一个edge；我们有两种方式解决这个问题，一个是每一次更新edge后，调用`nrf_drv_spi_xfer`，它需要在一个segment内的最后一个xfer之后做；有可能机会主义的在count触发时做，因为从触发到中断响应有7-8uS的延迟；但让安全的办法是在spi end的callback里做；但是这里我们只需要最后一次的spi end callback，一个tricky的办法是，在burst的一个周期里触发2次count timer递增，一次是channel 1的incr，另一次在spi end里，这样比如连续写入8次就会count到16；原有的触发count结束的逻辑放在count timer的15触发，而callback调用nrf_drv_spi_xfer可以放在16。



另一个方式是，仍然在一个大的周期而不是segment里，只调用一次spi xfer，但我们需要把每个segment展开成一个巨大的数组。可以估计一下最大的情况：



每个segment有9个xfer，每个2字节，是18字节；

5个pulse + 4个间隔 + 1个 hold + 1个recycle + 1个padding = 12；乘以18共计216字节；



如果这样工作的话，burst timer和count timer都没有中断需要处理。



# 状态设计

## Stateful Data

- `st_index`: int, `[0..st_num-1]`，参数数组的索引；
- `st_num`: 参数数组的数量；
- `st_regs[]`：spi寄存器参数数组，使用uint8_t数组，每次写入2字节；实际有效的数组大小是2 * 9 * st_num；因为ST_MAX是12，所以数组分配空间是216字节；
- `st_timing[]`：每个level/segment的持续时间，时序说明见下节













## Timing

时序的示意图入下。示意图中s1..s9表示在多簇设置下有5次正脉冲，需求上单周期内的连续脉冲的次数可以设置为1至5次。如果设置为1为普通模式，设置为2-5为多簇模式。s10是脉冲结束后、启动电荷回收前的保持阶段，该阶段可以没有。s11是电荷回收阶段，是s1+s3+s5+s7+s9的倍数，相应的，回收阶段的电流是脉冲阶段的积分之一；s12是单周期里最后的padding，这个padding阶段也可以没有，如果定义的脉冲，可选的保持阶段，和回收阶段时间之和恰好是单周期强度。



受限于处理器和DAC的通讯速度，所有阶段（如果有的话）都应该大于等于50uS。



```
  s1      s3     s5      s7      s9
-----   -----   -----   -----   -----
|   |   |   |   |   |   |   |   |   |
|   |   |   |   |   |   |   |   |   |											s12
    -----   -----   -----   -----   -----                           -------------------------------
     s2      s4      s6      s8      s10|           s11             |
                                        ---.-------------------------
                                           ^
                                           | 
                                           | 50uS after falling edge
                                        
```



实现该时序的设计要点入下：

1. cycle timer channel 0工作在普通模式（非extended，没有STOP/CLEAR MASK能力），channel 0 compare事件通过ppi触发burst timer以启动每个阶段；
2. cycle timer channel 0中断重新设置channel 0 compare时间，采用累进方式，细节后述；
3. cycle timer channel 1最终把结束周期重新开始；channel 1 compare设置后不修改；
4. cycle的起点是图中标注为箭头的位置；
   1. 选择在回收周期（图中s11）内的位置作为折返点是因为回收周期是保证出现且较长的周期，保持阶段和补时阶段都可能没有；
   2. 有50uS的延迟是因为在所有周期里，当channel 0 compare事件发生时都会有连续9次spi写入，调用nrf_drv_spi_xfer需要保证在全部spi写入完成之后；50uS是比较稳妥的时间；该值可以低至4 * 9 - 8 = 28uS，8是中断延迟。
5. 在每次nrf_drv_spi_xfer时一次提供全部的regs数组，之后每个channel 0的ppi触发会先让当前配置生效然后写入下一个阶段的配置；注意在折返点时，下一个阶段（s12，或者s1）的寄存器设置已经写入，即s12（如果有补时阶段）或者s1（如果没有补时阶段）的寄存器设置，是nrf_drv_spi_xfer参数数组的最后一组；
6. 在折返点，event_count被归零；在cycle timer chan0中断里累加，



cycle timer的time line如下图所示（以12阶段为例）：

```
0---(s11-50)---(s11+s12-50)---(s11+s12+s1-50)---...---(sigma[s1..s12]-50)---{sigma[s1..s12]}
```



cycle timer callback

```C
// for channel 0
nrf_drv_timer_compare(&m_cycle_timer, NRF_TIMER_CC_CHANNEL0, cycle_timer_channel0_timeout[index], true);
index++;
index = index % num_of_stages;

// for channel 1

```



| index | interrupt handler                 | comment |
| ----- | --------------------------------- | ------- |
| 0     | nrf_drv_timer_compare(); index++; |         |









时间上进入任何一个Sn（n=0..st_num-1）阶段，都是由cycle timer触发，cycle timer fire引起ppi启动burst timer，然后burst timer和count timer一连串的动作，顺序写入9次寄存器，其中第一次是同步生效指令，让上阶段写入的本阶段的配置同步生效，其余8个是下一阶段的8通道DAC配置。



在整个大周期里nrf_drv_spi_xfer只被调用一次；



S0是初始态，因为无限循环没有最终态。

进入s0有两种方式，初始化和cycle timer触发：

- cycle timer触发导致之前通过`nrf_drv_spi_xfer`排队的一组spi reg被写入dac，其中第一个是生效指令，其余8个是s1的寄存器配置；这个是自动的，与中断代码无关。
- 已经调用了`nrf_drv_spi_xfer`让生效指令和第一个positive pulse的进入排队状态，等待cycle







## 测例

DAC088的power down指令对本应用没有意义，该指令不是enable/disable某个channel的意思，power down的channel，任何写指令都会生效，即不能理解为power down/disable了一个channel后写指令没有影响；相反，写指令会override power down的结果。

### `test0a_all_mid`

该测试把所有通道写成`0x80`并停留在该状态；

测量所有DAC通道都被置为设计电平，约1.62V左右，VMID也为该电平；OPAMP开路输出则有一些为最高电压（约24.3V)，另一些为0V。

如果用电阻（680ohm, x1 or x2 in series）接地，则相应OPAMP输出达到0，如果接3.3V则相应OPAMP输出也在3.3V左右，符合howland电路设计目标。



### test0a_a_max

该测试把所有通道写成`0x80`，然后把通道A单独写成`0xFF`。

测量DAC A输出3.3V，其它通道均为1.65V左右，VMID也为该电平；OPAMP开路输出，A通道为24.3V。

如果680欧电阻把A接到地，





