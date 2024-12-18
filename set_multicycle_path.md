Q：多周期路径中的检查保持时间时刻,为什么默认是在建立时间检查的前一个cycle?请大家谈谈自己的理解。 如:Set_multicycle_path -setup 7 -to [whatever] 那么hold time 应该在7-1这个cycle检查,为什么?

A：

多周期路径中检查保持时间，如果你对建立时间设置多周期，那么保持时间检查就默认在前一建立时间，比如：
楼主所设定：
set_multicycle_path -setup 7 -to [whatever]
若一个周期为10ns，则此时数据到达触发器的时间范围应该是：[60+Th，70-Ts]
而至于为什么，一方面就是primetime等的默认，另外就是在检查单周期时，通常也是要求如此的，即要求数据在[Th，10-Ts]时间范围内到达。（Th表示保持时间，Ts表示建立时间）

但倘若你同时设定建立时间和保持时间都为多周期路径约束，那么延时范围就可以变大，如：
set_multicycle_path -setup 7 -to [clk]
set_multicycle_path -hold 6 -to [get_pins C_reg /D]
那么此时的保持时间检查就在Th时，而不是在60+Th时，而此时的arrival time的范围就变大了
是：[Th，70-Ts]
所以一般采用后者设定，这样多周期路径部分电路的优化范围就变大了。

重申一点：采用
set_multicycle_path -setup 7 -to [clk]
set_multicycle_path -hold 6 -to [get_pins C_reg/D]

这样多周期路径部分电路的优化范围确实变大了，但是会有亚稳态的潜在危险，所以要小心！

而只用：
set_multicycle_path -setup 7 -to [clk]
你又必须保证path_delay足够大，不会产生hold-time violation，DC几乎不可能做到这一点。

加register-enabling logic解决亚稳态

首先：
无论是单周期还是多周期，hold检查都默认为setup的前一个时钟。
你写了set_multicycle_path -setup 7 -to [clk]之后，dc何pt默认set_multicycle_path -hold 0 -to [get_pins C_reg/D]，如果你不写set_multicycle_path -hold 6 -to [get_pins C_reg/D]
其他的Harva 说得很详细了。
其次：
如richardhuang1 所说，加使能，在该采的时钟周期采数据。

Harva 和handsome说得很精彩,我帖一个synopsys的解释,大家共同学习.
set_multicycle_path and hold checks
-----------------------------------

Question:


I have a path that is set as multicycle path for the setup check. For some
reason, PrimeTime seems to be treating it as a multicycle path for hold time
checking as well. I'm using:

set_multicycle_path -setup 7 -to [whatever]

Why are the hold time checks multicycle?

Answer:


By default, if you specify 'set_multicycle_path -setup X', PrimeTime and
Design Compiler assume the datapath could change during any clock before
clock edge number X. To deal with this situation, PrimeTime and Design
Compiler implicitly add 'set_multicycle_path -hold 0 -to [whatever]'. This
positions the hold check one clock cycle before the setup check, effectively
constraining the path delay to be between X-1 and X clock cycles, or in
equation form:

X-1 cycles + T_hold < path delay (min)
                     path delay (max) < X cycles - T_setup

So by default the tools assume you want the path buffered up so that the
minimum change is > X-1 cycles.

This may not be the desired behavior. You can move the hold check back
towards the start of the multicycle period by specifying:

set_multicycle_path -hold X-1 -to [whatever]

In the above example, add

set_multicycle_path -hold 6 -to [whatever]

to the constraints and the hold check should occur on the desired edge. Note
that moving this check back requires the designer to handle possible
metastability. If the endpoint is a multi-bit signal, then you may need
to generate register-enabling logic to avoid clocking data before all of
it is valid.





我们先来说说什么是multicycle path，通常情况下，在同一个时钟驱动下的寄存器之间信号传输都是单周期的，如下图所示：



也就是说，当0时刻时钟上升沿到来后，FF1的Q端发生变化，再经过中间的组合逻辑电路，我们期望这个值是在10时刻的时钟沿被FF2采到。

但是在有的电路设计中，要么是有意为之，要么是单周期无法close timing，我们会遇到下面的电路。



其中FF4的Q变化要经过2个周期才能被FF5的D采样到，这个时候我们就要告诉STA工具，这个path不是单周期的，需要2个周期，在prime time中，这个command是set_multicycle_path。

我们先看set_multicycle_path有哪些主要参数（具体全部参数请参阅STA工具的command line reference）

 
set_multicycle_path path_multiplier [-setup|-hold] 
                    [-start|-end] 
                    -from <StartPoint>
                    -through <ThroughPoint>
                    -to <EndPoint>
 其中这个path_multicycle就是我们接下来重点要看的。

首先我们先说default值，也就是说什么都不设，这个path_multiplier对于setup来说是1，hold是0，也就是我们上面单周期的约束情况，我们在设计multicyle的时候，重点就是调整这个path_multiplier。

-setup和-hold是让你来分别指定setup和hold的多周期值，注意，如果你只设了setup的值而没有设hold的值，那么hold的值也会相应的进行改变，这个我们在下面的例子里会讲到。

-start和-end只是在不同的频率的clock分析中会用到，即launch path的clock和capture path的clock是不同频率时才用，对于launch path和capture path的寄存器都是同一个clock的电路，-start和-end不需要指定，因为没有意义。这里你可能会问，不同频率的时钟那不就是异步电路了嘛？是不是要跨时钟域多CDC，还能做STA吗？答案时不一定，不同频率依然可以是同步电路，即时钟沿是对齐的，另一方面，同一个频率的两个clock也可能是异步电路，关键是看时钟沿对齐没有。

-from/though/-to就是大家熟悉的指定timing path的起始，经过和终止路径，一般在约束特定的时候有用。

同频约束：
 好，我们先看简单的情况，即同一个clock的multicycle path设置：



 这种情况下其实就是默认的，你可以什么都不设，对应的其实就是：

set_multicycle_path 1 -setup  -from CLK1 -to CLK2
set_multicycle_path 0 -hold   -from CLK1 -to CLK2
那我们看下面的情况，也就是setup需要5个周期



set_multicycle_path -setup 5 -from CLK1 -to CLK2
注意，如果你不设置-hold，那么会自动调整为setup的沿的前一个沿，也就是说，其实如果只设了上面这一条，hold的timing不是CLK1的0时刻对应的时钟沿，而是CLK1的4时刻对应的时钟沿。

那么相应的，如果你要碰到下面这种情况呢？你就必须得写hold的set_multicycle_path了



set_multicycle_path 5 -setup  -from CLK1 -to CLK2
set_multicycle_path 4 -hold   -from CLK1 -to CLK2
注意hold的cycle是相对于setup path的沿往前算的，也就是说CLK2的setup capture edge为1，前一个沿为0，也就是默认的值，而如果你要继续往前面的沿来设就要调整multiplier的值，上面这个图作者把cycle值标在CLK2的下面了，便于大家理解。

好，上面这就是同一个clock或者同频率同步clock的设法。

慢时钟到快时钟约束：
我们下面来看不同频率的clock的设法，先来看从慢时钟到快时钟，假设慢时钟是快时钟的3分频。

默认情况下，如果不设multicycle path，setup/hold是这么分析的：



这也好理解，setup就是下一个沿，如果要用multicycle path写出来，就是：

set_multicycle_path 1 -setup  -end   -from CLK1 -to CLK2
set_multicycle_path 0 -hold   -start -from CLK1 -to CLK2
这里要注意-end和-start的用法，-end是指capture flop的clock，-start是指launch flop的clock，默认的，-setup是对-end，也就是上面CLK2的1个周期后，而-hold是对-start，也就是CLK1的0时刻的上升沿，在调整周期数的时候，一定要分清是针对哪一个CLK来调整。

看下面的setup的多周期例子



可以看到，setup要变成CLK2的2个周期，所以约束要写成：

set_multicycle_path 2 -setup  -end   -from CLK1 -to CLK2
如果你不写hold的约束，那么hold会自动根据setup调整。

那你如果想做下面图8的check



你是不是以为把hold的周期数设成下面这样就好了呢？

set_multicycle_path 1 -hold -from CLK1 -to CLK2
错了，这里要注意，前面说过，hold默认的是start clock，如果按照上面这句来写，那么实际上是相对CLK1移了一个周期。

那么实际上check就变成了：



那么要怎么正确设置图8的约束呢，这个时候我们要调整的是hold，要依据CLK2的周期来调整，要加-end选项。

set_multicycle_path 2 -setup  -end   -from CLK1 -to CLK2
set_multicycle_path 1 -hold   -end   -from CLK1 -to CLK2
快时钟到慢时钟约束： 
好，我们再来看从快时钟到慢时钟的写法，还是以3分频为例。

先看默认的check是什么？



那如果只调整setup的周期数，注意这里的多周期是对CLK1而言的

set_multicycle_path 2 -setup  -start  -from CLK1 -to CLK2
 设置上述约束，会check什么呢？

hold会相应调整，而且是以CLK1的变化调整：



要想把hold的沿对齐，那就要用下面的约束：



set_multicycle_path 2 -setup  -start   -from CLK1 -to CLK2
set_multicycle_path 1 -hold   -start   -from CLK1 -to CLK2
总结一下：
对于同频率的多周期电路约束，不需要用-start和-end，先设置setup的周期数，假设setup周期数是X，那么如果想要check最worst的hold情况，就要把hold的周期数设为X-1，即：



set_multicycle_path X   -setup  -from CLK1 -to CLK2
set_multicycle_path X-1 -hold   -from CLK1 -to CLK2
对于慢时钟到快时钟，如果多周期是针对快时钟的，对于setup可以加-end也可以不加，假设setup周期数是X，而对于hold如果想check沿对齐的情况，必须加-end选项，周期数为X-1。



set_multicycle_path X   -setup  -end   -from CLK1 -to CLK2
set_multicycle_path X-1 -hold   -end   -from CLK1 -to CLK2
对于从快时钟到慢时钟，如果多周期是针对快时钟的，对于hold可以加-start也可以不加，但是对于setup周期数一定要加-start



set_multicycle_path X   -setup  -start   -from CLK1 -to CLK2
set_multicycle_path X-1 -hold   -start   -from CLK1 -to CLK2
当然，以上都是设置CLK to CLK 之间的MCP，也可以针对某些特定路径下MCP，就采用pin到pin的形式，不要看到pin to pin的MCP就懵掉了。

