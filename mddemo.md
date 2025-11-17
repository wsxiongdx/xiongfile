
## 珏能示例代码

## 介绍
测试该珏能测控系统是为了在离⼦阱系统上实现量⼦纠错。当前公司使⽤ ARTIQ (时序控制) 加   Spectrum AWG (量⼦操作) ⽅案, 不⽀持纠错, 只⽀持运⾏静态量⼦线路。

关于珏能的介绍请参考下面文档：

[未仪的仪器介绍](https://futureinstrument.com/#/README)

[珏能示例代码文档](http://triq.cc:8000/#/software)
*用户名：unitary 密码：Un1tary123*

[珏能相关客户代码管理文档](https://xiaoshecode.github.io/rtmq/SeqConfig/)



## 代码架构个人理解

**该架构为Python描述到底层硬件执行的完整链路**。python指令通过类似Wasm的rtmq2进行编译和执行。

下面为个人理解的执行链路：

     用户层:
    seq('rwg0', multi_tone, 0)  →  seq函数
        ⇓
    编译层:
    assembler → core_domain → 指令生成函数
        ⇓
    mov()/cli()/chi() → 32位机器码
        ⇓
    asm[:] → 指令序列缓冲区
        ⇓
    传输层:
    run_cfg → pack_frame() → USB数据帧
        ⇓
    ft601_intf → 物理USB传输
        ⇓
    硬件层:
    FPGA指令缓存 → 指令解码 → 执行单元
        ⇓
    SPI控制器 → DDS配置 → 射频输出
        ⇓
    定时器控制 → 边带生成 → 波形调制



<div style="page-break-after: always;"></div>

# 示例1：点亮LED

**下面代码为led测试示例代码**.通过指定`seq('rwg0', led, 1)`中的rwg0参数确定控制几号卡槽位的板卡的LED，这里1代表二进制00000001表示D7灯亮。如果为127（二进制01111111）则控制D1~D7灯同时亮。

```python
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
"""  
珏能 基础示例 1: LED 控制基础  
演示 LED 的开启、关闭操作  
"""  
"""1. 硬件连接初始化"""  
# 连接USB ID为"IONCV2PROT"的白色大机箱（出厂默认ID，可通过软件修改）  
intf_usb = ft601_intf("IONCV2PROT")  
# 定义需控制的RWG卡槽位：1号槽和4号槽（物理槽位编号）  
rwgs = [1, 4]  
# 配置硬件运行目标：包含1号/4号槽RWG卡 + 主控制模块（[0]代表主模块）  
# 作用：指定软件控制的硬件范围，建立多模块协同的基础配置  
run_all = run_cfg(  
    intf_usb,                # 传入USB通信接口（已建立的硬件连接）  
    rwgs + [0],              # 控制对象：1号RWG卡、4号RWG卡、主模块  
    core=rwg.C_RWG           # 核心类型：明确以RWG卡控制逻辑为主（兼容主模块）  
)  
asm.cfg = run_all           # 将配置加载到汇编器上下文（asm）  

"""2. 定义LED控制函数"""  
@multi(asm)  # 修饰符：表示该函数可在多模块（RWG卡/主模块）上运行  
def led(x):  
    """控制硬件LED灯的亮灭状态  
    参数x：整数，通过二进制位控制单个灯（1个bit对应1个灯，1=亮，0=灭）  
    例如：x=127 → 二进制01111111 → 对应D1~D7灯亮  
    测试发现：二进制00010000（x=16）对应第4个灯亮也就是D3灯（bit4=1）  
    """    R.led = x  # R.led为硬件寄存器，直接写入二进制控制信号  
    
"""3. 序列编排：定义硬件执行步骤"""  
# 创建序列汇编器：关联硬件配置与模块类型（1号RWG卡→rwg0，4号RWG卡→rwg1，主模块→main）  
seq = assembler(run_all,[(f'rwg{i}',rwg.C_RWG) for i in range(len(rwgs))]+[('main',main.C_MAIN)]) # RWG卡模块（类型为RWG核心）+主控制模块  
# 清除历史序列（可选，避免旧配置干扰，根据需求启用）  
# seq.clear()  
# 定义序列步骤：在默认模块（通常为主模块）执行led函数，控制灯亮灭  
# 此处x=127（二进制01111111），控制D1~D7灯同时亮,如果是128则只有D0灯亮  
seq(None, led, 127)  
# 以下为测试用序列步骤（注释保留，用于单独测试某个RWG卡的LED）  
# seq('rwg0', led, 1)  # 在1号RWG卡（rwg0）执行，x=1→二进制00000001→D7灯亮  
# seq('rwg1', led, 3)  # 在4号RWG卡（rwg1）执行，x=3→二进制00000011→D6、D7灯亮  
####### time.sleep(3)  #注意这个方法不可用，不支持  
# seq(None, led, 0)  

"""4. 运行序列：执行所有定义的硬件操作"""  
seq.run()
```


<div style="page-break-after: always;"></div>

# 示例2：单频率RWG输出

*该示例为单卡输出可改编为多频率RWG的单卡输出*
**单频率RWG**对应我们目前使用的白色机箱的卡槽为编号1（具体可查询官方文档），单频率RWG的RF通道用0x00,0x20,0x40,0x60表示4个通道（具体看程序的注释）。

```python
from oasm.rtmq2.intf import ft601_intf#,uart_intf  
from pulser2 import *  
"""  
珏能 基础示例 2: 单频率RWG4通道输出  
演示 单频率RWG 4通道的同时输出波形或单独输出  
"""  
  
"""1. 硬件连接初始化"""  
# 连接USB ID为IONCV2PROT（出厂默认，可软件修改）的白色大机箱，如测试单板盒改成uart_intf和相应串口号  
intf_usb = ft601_intf("IONCV2PROT")#uart_intf("COM3",10000000)  
intf_usb.nod_adr = 0# 硬件节点地址  
intf_usb.loc_chn = 3 # 本地通信通道（固定为3，用于RWG卡数据传输）  
# 默认运行目标设置为到1号槽RWG卡的连接（因为我们单频率RWG在1槽）  
asm.cfg = rwg_run = run_cfg(intf_usb,[1],core=rwg.C_RWG)     #控制对象：1号RWG卡  
# 软重启RWG卡  
# rwg_run(rwg.boot())  

"""2. 定义RWG卡函数""" 
@core_run # 修饰符表示函数在RWG卡上运行  
def single_card(*carriers):  
    """  
    整张卡128tone可以在不同RF通道间任意分配，默认的分配是每个RF通道32个tone，也就是0x00~0x1F对应对应 RF0 通道,0x20~0x3F,0x40~0x5F,0x60~0x7F  
    单tone卡跟多tone卡的唯一区别是多余的tone被禁用，但程序结构是一样的  
    -- 我们这个为单频率RWG只用0x00,0x20,0x40,0x60就可以了，如果是多频率的可以同时编写并输出128个频率  
    -- (freq,amp,phase(可省略)), freq: -50~50，amp：0~1，phase: 0~1  
    --该卡为AWG+DDS的复合体运行rwg_play 默认会停留在最后的状态，所以最后需要再来一个 rwg_play{0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)}    幅值为0来关闭输出。这样就能看到上一段的10e6us的输出效果了，如果不加下面rwg_play会使得上面的波形状态一直输出不会关闭"""  
    # 步骤1：初始化4个RF通道的中心频率，单位MHz  
    rwg_init(carriers)  
    # 步骤2：硬件初始化等待， 初始20us定时器（单位为cycle=4ns），用于写入第一段的配置  
    count_down(5000)  
    # 等待上一个定时器时间到，在下一个时间块的开始时刻将本次rwg_play写入的配置生效，同时设置时长10us的定时器  
    #步骤3：生成并输出射频信号  
    # rwg_play(10e6,{0x00:(0,0.6),0x20:(5,0.1),0x40:(10,0.1),0x60:(-10,0.6)})  #10为持续时间，后面0x40:(10,0.1)代表频率偏移10Mhz，幅值为0.1  
    rwg_play(6e6, {0x00: (0, 0.8), 0x20: (5, 0), 0x40: (0, 0), 0x60: (0, 0)})    #单通道输出  
    # 步骤4：关闭射频信号  
    rwg_play(10, {0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)})  
  
single_card(50,110,150,200)  
# single_card(0, 120, 0, 0)
```



<div style="page-break-after: always;"></div>

# 示例3：多频率RWG输出

*该示例为多卡并行输出*
**多频率RWG**对应我们目前使用的白色机箱的卡槽为编号4，多频率RWG可以同时编写并输出128个频率（整张卡128tone可以在不同RF通道间任意分配，默认的分配是每个RF通道32个tone）。

```python
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
"""  
珏能 基础示例 3: 多频率RWG 输出  
演示 多频率RWG 与单频率RWG 的并行输出波形  
     或多频率RWG 单独输出  
"""  
"""1. 硬件连接初始化"""  
# 连接USB ID为IONCV2PROT（出厂默认，可软件修改）的白色大机箱  
intf_usb = ft601_intf("IONCV2PROT")  
intf_usb.nod_adr = 0  
intf_usb.loc_chn = 1  
rwgs = [1,4]    # 使用1、4槽的RWG卡 ，多频率RWG与单频率RWG并行输出波形  
# rwgs = [4]    #只有4卡槽为多频RWG，可使多频率RWG单独输出，  
asm.cfg = run_all = run_cfg(intf_usb, rwgs+[0])    # 传入USB通信接口（已建立的硬件连接）# 控制对象：4号RWG卡、主模块  
# 创建序列汇编器：关联硬件配置与模块类型（4号RWG卡→rwg1，主模块→main）  
seq = assembler(run_all,[(f'rwg{i}',rwg.C_RWG) for i in range(len(rwgs))]+[('main',main.C_MAIN)])  
"""2、参数定义（信号生成的基础配置）"""  
car = [95]*20                   #载波频率：20个载波，统一设为95 MHz  
amp = 1                         # 信号振幅  
"""3、函数定义（生成多频边带信号）"""  
def multi_tone(card):  
    from random import random  
    # 1. 多频信号基础参数  
    n = 1                           # 每个边带组的频率成分数量（仅1个频率，单频边带）  
    dur = 3e6                       # 信号持续时间，单位为us  
    df = 0.5                        # 频率间隔（单位MHz，边带间的频率差）  
  
    # 2. 硬件同步与初始化  
    wait_master()                   # 初始化当前RWG卡的载波频率（针对card=0和1，对应rwg0和rwg1）  
    rwg_init([car[2*card],0,car[2*card+1],0],[0,0,0,0],[64,0,64,0])          # RF0~RF3通道的中心频率、相位偏移、边带数量  
    #注意：1、中心频率如果两张卡（rwg0 用car[0]、car[1]，rwg1 用car[2]、car[3]，均为 95 MHz）；如果一张卡就直接car[0]、car[1]  
    #      2、多频信号数量总数不能超过128个  
    count_down(5000)  
    # 3. 边带配置（生成128个边带中的2个有效边带  
    sbgs = {i:((i-n/2)*df,amp,random()) for i in range(n)}               # 前64个边带（0~63）：生成1个边带，频率偏移=(i - n/2)*df = (0 - 0.5)*0.5 = -0.25 MHz，随机相位random()生成 0~1 的相位，避免两张卡的边带信号干涉  
    sbgs |= {i+64:((i-n/2)*df,amp,random()) for i in range(n)}            # 后64个边带（64~127）：生成1个边带，频率偏移同样为-0.25 MHz         用 |=合并第二个字典  
    # 4. 输出多频信号  
    rwg_play(dur,sbgs,rf=[0,2],sel=rwg.to_sel(2*dur))                # 启用当前RWG卡的2个RF通道（RF0、RF1）  
    wait()  
    rwg_play(10, {0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)})    #关闭输出  
"""4、控制两张卡输出"""  
# 主模块触发，让2张RWG卡或者1张卡进入就绪状态  
seq('main',trig_slave)  
# 2. 给每张RWG卡分配任务：循环调用multi_tone函数  
for i in range(len(rwgs)):  
    seq(f'rwg{i}',multi_tone,i)# 给rwg{i}分配任务：执行multi_tone，传入card=i  
# seq.run()  
seq.run(disa=True)
```


<div style="page-break-after: always;"></div>

# 示例4：RPC测试


```python
from oasm.rtmq2 import *  
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
"""  
珏能 基础示例 4: PRC led 测试  
演示 LED 灯循环亮与RPC测试  
  
实验用于验证 珏能 中 RPC 函数与 R.led 函数之间的参数传递机制  
 实验结论：  
  
- @intf_run 和@core_run 修饰函数可以在 硬件 和 PC 端之间传递数据  
"""  
"""1. 硬件连接初始化"""  
# intf = sim_intf('core'); intf.__enter__()  
intf = ft601_intf("IONCV2PROT")  
intf.nod_adr = 0  
intf.loc_chn = 1  
run = run_cfg(intf,[1],core=C_STD); run(asm())  
asm.cfg = run  

"""2、led测试，直接用R.led控制led，不用像Example_1中通过控制RWG来控制led"""  
#1.测试R.led能否直接用  
# core_write(R.led, 123)   #01111011  
# core_read(R.led)  
#2、测试用R.led循环  
# with asm:  
#     with For(R[0],128):   #R[0]从0循环到  
#         R.led = R[0]    ## 每次循环：将计数器R[0]的值赋给LED寄存器  
#         wait(10000)      ## 硬件延迟：等待10000个时钟周期（1cycle=4ns → 10000×4ns=40μs  
#     run(asm())   ## 执行这个硬件序列（让循环和LED变化生效）  
  
"""3、PRC test"""  
import math  
# intf.oper.clear()  
  
@intf_run  
def sqrt_sum(arr):  
    arr = [int(math.sqrt(i)) for i in arr]  
    copy(0, arr)  
    R.led = int(sum(arr))  
@core_run  
def test(n):  
    i = R[0]  
    for_(i, n)  
    R.led = i * i  
    intf_run_async(print)(i, R.led)   #intf_run_async异步打印——调用接口层的print，打印当前i和LED值（异步=不阻塞硬件循环，硬件继续跑）  
    DCH[i] = R.led  #存储数据——把i的平方值写入硬件缓存区DCH（DCH是硬件上的数组  
    # wait(2500)  
    wait(10000000)  # ## 硬件延迟：等待10000000个时钟周期（1cycle=4ns）测试发现如果超过40ms则会阻塞。  
    end()  
    # print(disassembler()(asm[:],0))  
    intf_run(print)(R.led, 123)    #intf_run同步打印——调用接口层print，（同步=等打印完再继续硬件操作），123为固定值  
    intf_run(print)(0, rng=20, narg=5)   #打印硬件数据——从硬件地址0开始，读取5个数据（rng=20=地址范围，narg=5=读取个数），测试发现rng = 4*narg，不然地址就乱了  
    sqrt_sum(0, rng=20, narg=5)  
    intf_run_async(print)(R.led, 456)  
    intf_run_async(print)(0, rng=20, narg=5)  
    intf_send(R.led, 789)  
    intf_send(0, rng=20)  
test(5)
```



<div style="page-break-after: always;"></div>

# 示例5：IO测试
该示例仅为了测试各板卡的` dio=[0, 2, 3]`IO口输出是否正常，以及TTL测试。
`当 dio=[0]表示该卡的IO0通道输出高电平，其他通道为低电平;当dio=[1，2，3]则4个通道只有为IO0通道为低电平。`

```python

from oasm.rtmq2.intf import ft601_intf#,uart_intf  
from pulser2 import *  
  
intf_usb = ft601_intf("IONCV2PROT")  
intf_usb.nod_adr = 0  
intf_usb.loc_chn = 3  
  
asm.cfg = rwg_run = run_cfg(intf_usb,[1],core=rwg.C_RWG)  
  
@core_run  
def single_card(*carriers):  
  
    rwg_init(carriers)  
  
    count_down(5000)  
    rwg_play(5e6,{}, dio=[0, 2, 3])  
    rwg_play(5e6, {}, dio=[1, 2, 3])  
    rwg_play(10e6, {0x00: (0, 0.8), 0x20: (5, 0), 0x40: (0, 0), 0x60: (0, 0)},dio=[0,1,3])  
    rwg_play(10, {0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)})  
  
single_card(50,110,150,200)
```


<div style="page-break-after: always;"></div>

# 示例6：测试波形同步

```python
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
  
intf_usb = ft601_intf("IONCV2PROT")  
rwgs = [1,4]  
asm.cfg = run_all = run_cfg(intf_usb, rwgs+[0])  
seq = assembler(run_all,[(f'rwg{i}',rwg.C_RWG) for i in range(len(rwgs))]+[('main',main.C_MAIN)])  
  
# 预初始化函数  
def pre_init(card, frequencies):  
    rwg_init(frequencies)  
    wait_master()  
    count_down(5000)  
  
 
seq('rwg0', pre_init, 0, [100,100,350,350])  
seq('rwg1', pre_init, 1, [150,150,150,150])  
seq('main', trig_slave)  
  
seq('rwg0', rwg_play, 10e6, {0x00:(0,1),0x20:(0,1)}, dio=[0,1,3])  
# seq('rwg0', rwg_play, 10, {0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)})  
seq('rwg1', rwg_play, 10e6, {0x00:(-50,1),0x01:(0,1),0x20:(-1,1),0x21:(0,1)}, dio=[0,1,3])  
# seq('rwg1', rwg_play, 10, {0x00: (0, 0), 0x20: (0, 0), 0x40: (0, 0), 0x60: (0, 0)})  
seq.run()
```


<div style="page-break-after: always;"></div>


# 示例7：测试时序代码

## MS门时序图
![[271e7bb903b7d1ffda8ce80fc3686f14.png]]
MS门代码简写
![[1983262c21fd8333b9d33836aa3b469a.png]]
## 时序示例代码
该代码严格按照上面的时序而编写，但是输出时间为了示波器可观测则统一延长输出时间到秒量级，实际测试需要修改回去。

```python
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
  
intf_usb = ft601_intf("IONCV2PROT")  
rwgs = [1,4]  
asm.cfg = run_all = run_cfg(intf_usb, rwgs+[0])  
seq = assembler(run_all,[(f'rwg{i}',rwg.C_RWG) for i in range(len(rwgs))]+[('main',main.C_MAIN)])  
  
# 预初始化函数  
def pre_init(card, frequencies):  
    rwg_init(frequencies)  
    wait_master()  
    count_down(5000)  
# 初始化  
seq('rwg0', pre_init, 0, [150,150,150,150])  
seq('rwg1', pre_init, 1, [100,100,350,350])  
seq('main', trig_slave)  
  
#DDS1-DDS4 测试,时间为可观测量，秒量级（单位为ns）
seq('rwg0',rwg_play,5e6,{0x00:(0,1),0x20:(0,1),0x40:(0,1)},dio=[0])  
seq('rwg0',rwg_play,10e6,{0x60:(0,1)},dio=[0])  
seq('rwg0',rwg_play,5e6,{0x20:(0,1)},dio=[0])  
seq('rwg0',rwg_play,22e5,{},dio=[0])  
seq('rwg0',rwg_play,10e5,{0x20:(0,1)},dio=[0,1])  
seq('rwg0',rwg_play,18e5,{0x00:(0,1),0x20:(0,1),0x40:(0,1)},dio=[0])  
seq('rwg0', rwg_play, 10, {})  

# #awg1-4 ttl1-4测试  
seq('rwg1',rwg_play,15e6,{},dio=[1,2,3])  
seq('rwg1',rwg_play,5e6,{},dio=[0,2])  
seq('rwg1',rwg_play,2e5,{0x00:(-1,1),0x01:(0,1),0x20:(-1,1),0x21:(0,1)},dio=[0,1,3])  
seq('rwg1',rwg_play,5e5,{0x00:(-1,1),0x01:(0,1),0x20:(-1,1),0x21:(0,1),0x40:(0,0.5),0x60:(-1,1),0x61:(1,1)},dio=[0,1,3])  
seq('rwg1',rwg_play,10e5,{0x00:(-1,1),0x01:(0,1),0x20:(-1,1),0x21:(0,1),0x40:(0,1),0x60:(-1,1),0x61:(1,1)},dio=[0,1,3])  
seq('rwg1',rwg_play,5e5,{0x00:(-1,1),0x01:(0,1),0x20:(-1,1),0x21:(0,1),0x40:(0,0.5),0x60:(-1,1),0x61:(1,1)},dio=[0,1,3])  
seq('rwg1',rwg_play,10e5,{},dio=[0,1,3])  
seq('rwg1',rwg_play,18e5,{},dio=[1,2,3])  
seq('rwg1', rwg_play, 10, {})  
#dds1-4 ttl5-6,dds5没写  
# seq.run(disa=True)  
seq.run()
```
单独解释一下`seq('rwg0',rwg_play,18e5,{0x00:(0,1),0x20:(0,1),0x40:(0,1)},dio=[0])`这个代码意思rwg0表示板卡索引，18e5为脉冲时间，0x00:(0,1)（单频率RWG输出代码注释有解释）,dio=[0]为该卡的io0通道输出高电平。

## 测试时序循环

```python
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
  
intf_usb = ft601_intf("IONCV2PROT")  
rwgs = [1, 4]  
cycles = 2  # 循环次数  
# # 循环播放函数  
# def loop_play(cycles_count, duration, sbgs, **kwargs):  
#     for i in range(cycles_count):  
#         rwg_play(duration, sbgs, **kwargs)  
#         wait()  # 等待当前播放完成  
for i in range(cycles):  
    print(f"第 {i + 1} 次循环")  
    asm.cfg = run_all = run_cfg(intf_usb, rwgs + [0])  
    seq = assembler(run_all, [(f'rwg{i}', rwg.C_RWG) for i in range(len(rwgs))] + [('main', main.C_MAIN)])  
    # 预初始化函数  
    def pre_init(card, frequencies):  
        rwg_init(frequencies)  
        wait_master()  
        count_down(5000)  
    # 初始化  
    seq('rwg0', pre_init, 0, [150, 150, 150, 150])  
    seq('rwg1', pre_init, 1, [100, 100, 350, 350])  
    seq('main', trig_slave)  
    # 单次播放  
    seq('rwg0', rwg_play, 5e6, {0x00: (0, 1)}, dio=[0])  
    seq('rwg0', rwg_play, 10, {})  
    seq('rwg1', rwg_play, 10e5,{0x00: (-1, 1), 0x01: (0, 1), 0x20: (-1, 1), 0x21: (0, 1), 0x40: (0, 1), 0x60: (-1, 1), 0x61: (1, 1)},dio=[0, 1, 3])  
    seq('rwg1', rwg_play, 10, {})  
    seq.run()  
    # 延迟  
    if i < cycles - 1:  
        import time  
  
        time.sleep(10)
```

<div style="page-break-after: always;"></div>

# 示例8：TTL测试（黑色机箱）
    连接COM口（设备管理器中-A后缀）的黑色外设机箱  
    
该机箱中有3张卡（AD5791、AD5372、gpio），该代码可单独测试机箱中的各板卡的功能：

| **子卡型号** | **功能 / 通道数** | **参数**             |
| -------- | ------------ | ------------------ |
| AD5791   | 电压输出 / 2通道   | ±10V， 20bit， 1M刷新率 |
| AD5372   | 电压输出 / 32通道  | ±10V               |

```python

from oasm.rtmq2.intf import uart_intf  
from oasm.rtmq2.intf import ft601_intf  
from pulser2 import *  
from oasm.dev.da4b1_g8 import *; flex = da4b1_g8  
  
# 连接COM口（设备管理器中-A后缀）的黑色外设机箱  
intf_usb = uart_intf("COM4",10000000)  
intf_usb.nod_adr = 0  
intf_usb.loc_chn = 1  
  
asm.cfg = flex_run = run_cfg(intf_usb, [0], core=flex.core)  
  
# 软重启外设机箱  
# flex_run(flex.boot())  
  
def test_gpio():  
    R.dio.dir = 0  
    with While():  
        R.ttl = -1  
        wait(1*flex.us)  
        R.ttl = 0  
        wait(1*flex.us)  
  
def test_5372():  
    with While():  
        for i in range(100):  
            flex.spi00.dac_5372(-8,(i-50)/5)  
            wait(20*flex.us)  
  
def test_5791():  
    with While():  
        for i in range(100):  
            for chn in range(8):  
                getattr(flex,f'spi{chn+1:02x}').dac_5791(5.0*(chn-2+(i/110)))  
                # getattr(flex,f'spi{chn+1:02x}').dac_5791((i-50)/5)  
            wait(1*flex.us)  
  
# flex_run(test_gpio)()  
# flex_run(test_5372)()  
flex_run(test_5791)()
```
