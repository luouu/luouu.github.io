MIPI (Mobile Industry Processor Interface) 是2003年由ARM, Nokia, ST ,TI等公司成立的一个联盟，目的是把手机内部的接口如摄像头、显示屏接口、射频/基带接口等标准化，从而减少手机设计的复杂程度和增加设计灵活性。 

MIPI联盟下面有不同的WorkGroup，分别定义了一系列的手机内部接口标准，比如摄像头接口CSI、显示接口DSI、射频接口DigRF、麦克风 /喇叭接口SLIMbus等。统一接口标准的好处是手机厂商根据需要可以从市面上灵活选择不同的芯片和模组，更改设计和功能时更加快捷方便。下图是按照 MIPI的规划下一代智能手机的内部架构。

![img](.img/mipi/MIPI-Protocol-865x1024.png)

## MIPI规范

已经完成和正在计划中的规范如下(https://zh.wikipedia.org/wiki/MIPI#cite_note-Working_Groups:_Overview-1)。

| **工作组**                    | **规范名称**                                                 |
| ----------------------------- | ------------------------------------------------------------ |
| Camera工作组                  | MIPI Camera Serial Interface 2 (MIPI CSI-2) v2.0 (March 2017)MIPI Camera Serial Interface 3 (MIPI CSI-3) v1.1 (March 2014)MIPI Camera Command Set (MIPI CCS) v1.0 (October 2017) |
| Device Descriptor Block工作组 | 暂无                                                         |
| DigRF工作组                   | DigRF Baseband/RF Digital Interface Specification v4 (Feb. 2014) |
| Display工作组                 | DBI-2DPI-2DSI-2 v1.0 (January 2016)DCS                       |
| 高速同步接口工作组            | HSI 1.0                                                      |
| 接口管理框架工作组            | 暂无                                                         |
| 低速多点连接工作组            | SLIMbus v2.0 (August 2015)SoundWire v1.1 (August 2016)       |
| NAND软件工作组                | 暂无                                                         |
| 物理层工作组                  | C-PHY v1.2 (March 2017)D-PHY v2.1 (March 2017)M-PHY v4.1 (March 2017) |
| 软件工作组                    | 暂无                                                         |
| 系统电源管理工作组            | SPMI v2.0 (August 2012)                                      |
| 检测与调试工作组              | 暂无                                                         |
| 统一协议工作组                | UniPro 1 point-to-point v1.61 (October 2015)PIE              |

#### **2. DSI & CSI**



MIPI是一个比较新的标准，其规范也在不断修改和改进，目前比较成熟的接口应用有DSI(显示接口)和CSI（摄像头接口）。CSI/DSI分别是指其承载的是针对Camera或Display应用，都有复杂的协议结构。以DSI为例，其协议层结构如下：

![img](.img/mipi/MIPI-DSI.png)

CSI/DSI的物理层（Phy Layer）由专门的WorkGroup负责制定，其目前的标准是D-PHY。D-PHY采用1对源同步的差分时钟和1～4对差分数据线来进行数据传输。数据传输采用DDR方式，即在时钟的上下边沿都有数据传输。

D-PHY的物理层支持HS(High Speed)和LP(Low Power)两种工作模式。HS模式下采用低压差分信号，功耗较大，但是可以传输很高的数据速率（数据速率为80M～1Gbps）； LP模式下采用单端信号，数据速率很低（<10Mbps），但是相应的功耗也很低。两种模式的结合保证了MIPI总线在需要传输大量数据（如图像） 时可以高速传输，而在不需要大数据量传输时又能够减少功耗。

![img](.img/mipi/MIPI-DPHY.png)





https://zhuanlan.zhihu.com/p/100476927


