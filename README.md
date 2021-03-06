# 通用8051单片机PWM模块

普通的8051单片机是没有硬件PWM模块的，要基于定时器等硬件、结合软件实现PWM输出。

本项目使用一个定时器（和对应中断向量），实现8路PWM输出。优点是输出的PWM的周期、脉宽精确，不会出现抖动（比如舵机抖动），缺点是时间有最小粒度的限制，且8路输出引脚必须属于同一个端口（P0、P1等）。

假设系统使用12MHz晶振，把时间粒度设为200个系统周期，则时间粒度是200us，即0.2ms。那么PWM周期和脉宽都必须是0.2ms的整数倍。这是受限于PWM模块的实现原理的。

在初始化时，把定时器设置为8位自动重装模式，并且TH填入256-200=56，这样每200个系统周期都会触发一个定时器中断。在定时中断的一开头，对端口（下文假设是P1）做如下操作：

```
P1 = (P1 | g_or_mask) & g_and_mask;
```

这样保证了对P1各个引脚的修改的时间间隔都是严格的0.2ms。

其中，g_or_mask是一个初始化后不再改变的字节。如果P1的8个引脚都用来输出PWM，那么g_or_mask就是0xff。一般地，P1中哪些引脚用来输出PWM（最多支持8路，可以只配置若干路），g_or_mask中对应的位就是1。g_or_mask就是用来把各个PWM输出引脚拉高的。

而g_and_mask则是一个不停改变的字节。在每次中断中，都会根据各个引脚的脉宽来调整g_and_mask，为下一次中断做好准备。那么重点就在于g_and_mask的更新算法。

以4路PWM为例，周期设为g_period=100（即100\*0.2ms=20ms）。在全局有一个数组，叫做g_channel_widths[4]，记录的是各个channel的脉宽。还有一个编译时确定的只读数组g_channel_pins[4]，记录了各个channel的引脚。

假设4个channel的引脚各为2, 3, 6, 7（即分别是P1_2，P1_3, P1_6和P1_7），则
```
g_channel_pins[4] = {2, 3, 6, 7}
```
而4个channel的脉宽分别是40, 25, 33, 16（即分别是40\*0.2ms=8ms, 25\*0.2ms=5ms, 33\*0.2ms=6.6ms和16*0.2ms=0.32ms）。

这两个数组构成一张表如下：

通道|时间|引脚
--|--|--
0|40|2
1|25|3
2|33|6
3|16|7

由这张“配置”表，可以生成一张“执行”表：

时间|动作
--|--
16|拉低引脚7
25|拉低引脚3和7
33|拉低引脚3、6和7
40|拉低引脚2、3、6和7

执行表按照时间排序。“动作”需要转换成g_and_mask，所以这张“执行”表在机器中如下：

时间|掩码
--|--
16|01111111
25|01110111
33|00110111
40|00110011

而g_or_mask一直固定为11001100，因为只有2, 3, 6, 7四个引脚。

每次中断时，g_next_time变量都会加一，如果g_next_time增长到了“执行”表中的某一行的时间时，则把g_and_mask设为该行的掩码。

基本的原理就是这样。不过具体的实现细节还需要考虑并发性。有可能脉宽进行修改时，中断发生了，那么"执行表“会发生脏读，可能导致中断的逻辑全部乱了。所以代码中会有两张”执行表“交替使用。

注意，使用STC系列的51单片机时，由于指令周期短了很多，所以可以做到更细的粒度（比如20，即20个系统周期，在12MHz晶振下精度为20us，在48MHz晶振下精度为5us）。

代码必须使用SDCC编译！