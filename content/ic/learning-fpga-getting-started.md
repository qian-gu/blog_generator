Title: 学习 FPGA 入门
Date: 2014-04-03 12:40
Category: IC
Tags: FPGA
Slug: learning-fpga-getting-started
Author: Qian Gu
Summary: 总结 FPGA 的学习经历，温故而知新。

## FPGA 是什么

FPGA 是 PLD 家族中的一员，要说清楚什么是 FPGA，就不得不说一说 PLD。（**以下内容来自wikipedia**）

### PLD & ASIC

早期的数字逻辑系统，是由中、小规模集成电路芯片搭建而成的。这种形式的电路在可靠性、工作速度、功耗和体积方面都难以满足大规模、高性能信息处理系统的要求 。后来，随着集成电路的发展，出现可专用集成电路 。

**专用集成电路**（Application Specific Integrated Circuits, [`ASIC`][ASIC]）是指依产品需求不同而非通用目的，而自定义的特殊规格集成电路 。ASIC 可以将整个系统集成到一个芯片上。由于芯片内集成度高、连线短，所以它可以满足之前 ”搭建系统“ 难以满足的性能指标 。

随着芯片尺寸的减小和设计工具的发展，这些年来 ASIC 芯片最大的集成度从 5,000 增长到了 超过100,000,000 门 。现代的 ASIC 通常都包含了微处理器、存储块（如 ROM 、RAM 、EEPROM 、Flash 等）。这种 ASIC 被称为 SoC （[system on chip][soc]），ASIC 设计师采用硬件描述语言（Hardware Description Language，HDL），比如 Verilog、VHDL，描述 ASIC 的功能 。

但是 ASIC 的研制周期长，现代信息处理的快速发展要求集成电路的设计、测试和生产周期尽可能的段，这就促进了可编程逻辑器件的发展 。严格地说，有些可编程器件出现的时间比 ASIC 出现的早，这里说的可编程器件主要是指 CPLD/FPGA 。

**可编程逻辑器件**（Programmable Logic Device,[`PLD`][PLD]）是一种用来搭建可重配置数字电路的电子器件。和逻辑门（logic gate）不同，PLD 出厂时逻辑功能是没有被定义的，在使用之前，必须先重配置（reconfigure）。

[ASIC]: http://en.wikipedia.org/wiki/Application-specific-integrated-circuit
[soc]: http://en.wikipedia.org/wiki/System-on-chip
[PLD]: http://en.wikipedia.org/wiki/Programmable-logic-devices

### 历史

#### ROM as PLD

在 PLD 器件被发明之前，就已经有人将 [ROM][ROM] 以 PLD 的概念来运用，用 ROM 芯片来充当一些输入性的组合逻辑（combinatorial logic）的函数发生器。ROM 有 m 个地址线，则有 `n = 2^m` 个输出结果，这和布尔逻辑是一一对应的，所以如果把 ROM 的地址线当作相互之间没有关系的输入，则相应的输出就可以实现不同的函数。

早期的 Mask ROM 存储数据的方式是使用内部的硬件电路，所以只能在出厂时就写入数据，而且以后不能更改。这就导致了一系列缺点：

1. 因为消费者必须联系制造商才能生产出自定义的芯片，所以只有买大量的 ROM 芯片时才经济划算 。

2. 因为同样的原因，从设计到最终生产出产品，中间耗费的时间很长 。

3. Mask ROM 在研发中基本上不能使用，因为设计师在改进设计时需要经常改动 ROM 里面的值 。

4. 如果一个设备中含有故障的 Mask ROM，那么修复这个设备的唯一方法就是召回设备并且更换其中的每一个 Mask ROM 。

**PROM**

[PROM][PROM]（Programmable Read Only Memory）是周文俊于 1956 年发明的。他在纽约 Garden City 的 American Bosch Arma Corporation 工作，当时，美国空军为了提升空军用计算机以及Atlas E/F波段导弹的灵活性和保安性而提出要求，这项技术就是为了满足这一要求而产生的 。

PROM 是通过熔丝/反熔丝（fuse/antifuse）实现对每个 bit 的设置 。通过使用高电压脉冲改变内部的物理结构，这种方法通常是不可逆的，所以它只允许用户更改一次配置 。PROM 解决了上面提到的问题 1 和问题 2 ，因为公司可以买一大批没有配置过的 PROM，设计人员可以根据自己的需要随意配置 。

**EPROM**

[EPROM][EPROM]（Erasable Programmable Read Only Memory）是 intel 公司的 Dov Frohman 于 1971 年发明的 。与 PROM 不同的是，EPROM 可利用高电压将资料编程写入，通过紫外线照射的方式不断的重置为未配置状态。因此，在封装外壳上会预留一个石英玻璃所制的透明窗以便进行紫外线曝光。写入程序后通常会用贴纸遮盖透明窗，以防日久不慎曝光过量影响资料。

因为EPROM 可以重复配置，所以它解决了上面的第 3 个问题 。

**EEPROM**

[EEPROM][EEPROM]（Electrically Erasable Programmable Read-Only Memory）于 1983 年被发明出来。相比EPROM，EEPROM不需要用紫外线照射，也不需取下，就可以用特定的电压，来抹除芯片上的信息，以便写入新的数据 。

如果设备可以从外部接收数据（比如 PC 通过串口线），就可以在线配置 EEPROM，这样它解决了问题 4 。

虽然解决了上面的 4 个问题，但是把 ROM 当作 PLD 器件使用，还是有很多弊端：

1. 与专用逻辑电路相比，ROM 的速度很慢

2. 当输入不同步时（异步状态），ROM 的输出有毛刺

3. 更加耗电

4. 与可编程逻辑相比，价格更贵，尤其是高速应用中

而且，大部分 ROM 没有输出寄存器，所以它不能直接应用在时序电路中，所以在状态机的设计中，通常还需要一个外部的 TTL 寄存器 。对电路设计的业余爱好者来说，有时也仍然用“2716”之类的普遍型EPROM芯片来充当PLD，这种用法有时也称为“穷人的PAL”。（PAL也是PLD的一种，以下将再进一步说明）

[ROM]: http://en.wikipedia.org/wiki/Read-only-memory
[PROM]: http://en.wikipedia.org/wiki/Programmable-read-only-memory
[EPROM]: http://en.wikipedia.org/wiki/EPROM
[EEPROM]: http://en.wikipedia.org/wiki/EEPROM

于是，就出现了 PLD 器件。

#### 早期可编程逻辑

+ 1969 年，Motorola 生产出 XC157，它是一个有 12 个逻辑门和 30 个独立输入/输出管脚的可编程逻辑正列。

+ 1970 年，德州仪器（TI）在 IBM 的 ROAM 基础上生产出 TMS2000，它有 17 个输入管脚，18 个输出管脚，8 个 JK 触发器来存储。TI 为这个设备发明了一个新名字 Programmable Logic Array（PLA）。

+ 1971 年，通用电器公司（GE）在新的 PROM 技术的基础上发明了一种可编程逻辑器件。这个实验性质的设备通过使用多层逻辑来提高 IBM 的 ROAM 性能 。GE 的这个设备是最早的 PLD 设备，比 Altera 的 EPLD 早了十几年 。

+ 1974 年，GE 和 Monolithic Memories 达成协议，开发一种可编程逻辑器件。这个设备被称为 ”Programmable Associative Logic Array“ 或者是 PALA 。最终于 1976 年完成 MMI 5760 ，它可以实现超过 100 门的时序电路。GE 的开发环境支持这一器件，它可以直接将布尔表达式转化为配置器件的代码，然而最终这个器件却没有上市。

#### PLA

1970 年，德州仪器（TI）在 IBM 的 ROAM 基础上生产出 TMS2000，它有 17 个输入管脚，18 个输出管脚，8 个 JK 触发器来存储。TI 为这个设备发明了一个新名字 Programmable Logic Array（[PLA][PLA]）。

PLA 具有一组可编程的 AND 阵列，AND 阵列之后连接一组可编程的 OR 阵列 ，这样就可以只在合乎设定条件时才允许产生逻辑信号输出 。

虽然名字中含有可编程 3 个字，但是并不是所有的 PLA 都可以现场编程，事实上许多都属遮罩性的可编程化，性质与ROM相同，必须在芯片制造厂内就执行与完成程序化设定，尤其是内嵌于电路较复杂的芯片（例如：微处理器）的PLA多属此种程序化方式。

[PLA]: http://en.wikipedia.org/wiki/Programmable-Logic-Array

#### PAL

[PAL][PAL]（Programmable Array Logic）是 Monolithic Memories 公司在 1978 年 3 月提出的，在数字电路中用来搭建逻辑功能的可编程器件的总称 。PAL 内部含有固定的或门阵列，可编程的与门阵列，从而实现所要求的逻辑函数。

PAL 内部有个 PROM 的核，外部附加的输出逻辑电路，这样就可以实现所需要的逻辑功能 。因为 PAL 是基于 PROM 的，所以要使用特殊的设备，PAL 才具有可编程性，而且是 ”一次编程“ 。

[PAL]: http://en.wikipedia.org/wiki/Programmable-Array-Logic

#### GAL

[GAL][GAL]（Generic array logic）是 PAL 的发展，是 Lattice Semiconductor 于 1985 年发明 。这个设备具有和 PAL 同样的功能，但是可以重配置多次，所以 GAL 在设计中很有用，一旦有错误，只需要擦除后重新配置即可 。

后来，International CMOS Technology (ICT) 公司发明了 类似的设备，称为 PEEL（programmable electrically erasable logic）。

[GAL]: http://en.wikipedia.org/wiki/Generic-array-logic

#### CPLD

[CPLD][CPLD]（Complex programmable logic device）适合用来实现各种运算和组合逻辑（combinational logic）。PAL、GAL仅适合用在约数百个逻辑门所构成的小型电路，若要实现更大的电路则适合用 CPLD，一颗CPLD内等于包含了数颗的PAL，各PAL（逻辑区块）间的互接连线也可以进行程序性的规划、烧录，CPLD运用这种多合一（All-In-One）的整合作法，使其一颗就能实现数千个逻辑门，甚至数十万个逻辑门才能构成的电路。

CPLD 与 PAL 的共同点：

+ 非易失性配置存储器。与 FPGA 不同，CPLD v不需要外部的 ROM，只要系统上电，就可以正常工作。

+ 对于许多旧的 CPLD 来说，布线约束要求大部分逻辑块要和输入输出相连接，以减少内部状态记录，对于新的 CPLD 系列来说，已经不需要这样了。

CPLD 与 FPGA 的共同点：

+ 可以利用大量的逻辑资源，CPLD 等价有有数百万的逻辑门资源可以用来实现比较复杂的设计，而 PAL 最多等价有几千个逻辑门，FPGA 有几万到几百万的逻辑门。

+ 提供一些更加灵活的资源，比如宏模块之间复杂的反馈连接和整数运算。

大的 CPLD 和小 FPGA 之间最显著的差别就是 CPLD 含有片内非易失性存储器 。因为非易失性存储器的特点，CPLD 在数字电路设计中被当作 ”boot loader“ 来使用，之后它再把系统的控制权转交给没有这种特性的设备，最好的例子就是使一块 CPLD 从非易失性存储器中装载配置 FPGA 所需要的数据 。

[CPLD]: http://en.wikipedia.org/wiki/Complex-programmable-logic-device

#### FPGA

[FPGA][FPGA]（Field Programmable Gate Array，FPGA）是在 PAL、GAL、CPLD 等可编程逻辑器件的基础上进一步发展的产物。它是作为专用集成电路领域中的一种半定制电路而出现的，既解决了全定制电路的不足，又克服了原有可编程逻辑器件门电路数有限的缺点。

当 PAL 忙于进展成 GAL、CPLD 时，另一种 “可编程化” 的流派也逐渐成形，此称之为现场可编程闸阵列（Field Programmable Gate Array，FPGA）。FPGA是以阵列（Gate Array）技术为基础所发展成的一种PLD 。所谓 ”Field Programmable“ 就是说芯片是出厂以后由客户或者设计师配置而工作的 。

1980 年代后期，Naval Surface Warfare Department 在 Steve Casselman 的提议下成立了实验项目，目的是为了研制一台由 600,000 个逻辑门组成的计算机。Casselman 最后成功了并且在 1992 年获得了专利 。

Xilinx 公司的共同创世人 [Ross Freeman][Freeman] 和 [Bernard Vonderschmitt][Vonderschmitt] 在 1985 年发明出第一款商业 FPGA —— XC2064 。XC2064 芯片有可以编程的逻辑门和可以编程的内部连接线，这开辟了一项新的技术和市场 。XC2064 有 64 个可配置逻辑块（configurable logic blocks，CLBs），和 3 输入查找表（lookup tables，LUTs）。

从 1985 开始到 90 年代中期，Xilinx 一直处于高速发展阶段，之后竞争对手出现了，截至 1993 年，Actel 占据了 18% 的市场 。

90 年代是 FPGA 爆炸式发展的年代，这一期间出现了大量高端技术和产品。在 90 年代初期，FPGA 主要应用于通信领域，在 90 年代后期，FPGA 已经广泛应用于消费品、汽车和工业应用 。

[FPGA]: http://en.wikipedia.org/wiki/Field-programmable-gate-array
[Freeman]: http://en.wikipedia.org/wiki/Ross-Freeman
[Vonderschmitt]: http://en.wikipedia.org/wiki/Bernard-Vonderschmitt

## 为什么选择 FPGA

### FPGA vs ASIC

**ASIC 的优点：**

ASIC 在批量生产时与通用集成电路相比具有体积更小、功耗更低、可靠性提高、性能提高、保密性增强、成本降低 。

**ASIC 的缺点：**

设计周期最长，设计成本贵，设计费用最高，适合于批量很大或者对产品成本不计较的场合。

至于 FPGA 的优点和缺点完全就是 ASIC 的取反 。FPGA 一般来说比专用集成电路（ASIC）的速度要慢，无法完成更复杂的设计，并且会消耗更多的电能。但是，FPGA 具有很多优点，比如可以快速成品，而且其内部逻辑可以被设计者反复修改，从而改正程序中的错误，此外，使用 FPGA 进行除错的成本较低 。在一些技术更新比较快的行业，FPGA几乎是电子系统中的必要部件，因为在大批量供货前，必须迅速抢占市场，这时FPGA方便灵活的优势就显得很重要。这也是 FPGA 能够发展起来的原因，市场是不会允许一个毫无优势的技术发展到今天这种地步的 。

两者不是对立的，由于各自的特点它们有各自适用的环境，不能一棒子打死，否定其中一个。事实上更多的情况是：设计开发是在普通的 FPGA 上完成的，然后将设计转移到一个类似于 ASIC 的芯片上 。

### FPGA vs CPLD

为了达到上述目的，还有一种方法是使用 CPLD 。
CPLD和FPGA都包括了一些相对大数量的可以编辑逻辑单元。CPLD逻辑门的密度在几千到几万个逻辑单元之间，而FPGA通常是在几万到几百万。

FPGA 与 CLPD 最大的区别就是：FPGA 是基于查找表（look up table，LUT），而 CPLD 是基于海门架构（sea-of-gates），也就是它们的系统结构 。CPLD 的结构具有一定的局限性 。这个结构由一个或者多个可编辑的结果之和的逻辑组列和一些相对少量的锁定的寄存器组成 。这样的结果是缺乏编辑灵活性，但是它的优点是，其延迟时间易于预计，逻辑单元对连接单元比率较高 。而FPGA具有的连接单元数量很大，这样虽然让它可以更加灵活的编辑，但是结构却复杂的多 。

CPLD 和 FPGA 另外一个区别是大多数的 FPGA 含有高层次的内置模块（比如加法器和乘法器）和内置的存储器 。一个由此带来的重要区别是，很多新的 FPGA 支持完全的或者部分的系统内重新配置 。允许他们的设计随着系统升级或者动态重新配置而改变 。一些FPGA可以让设备的一部分重新编辑，而其他部分继续正常运行 。

CPLD 与 FPGA 之间结构、原理上的差别导致两者应用上的差别 。考虑成本、性能要求等因素，应该根据实际情况选择 。

## Xilinx & Altera

FPGA 的制造商主要是 Xilinx 和 Altera 两家，他们合起来市场占有率达到了 80% 之多 。两家是 FPGA 技术的领导者也是长期竞争对手 。

[Xilinx][Xilinx] 于 1984 年创建于美国加利福尼亚州的硅谷，总部位于硅谷核心的圣何塞。它是一家主要提供 FPGA 的科技公司 。并且就是它的创始人是发明了 FPGA 。

[Altera][Altera] 是一家位于美国硅谷的可编程逻辑器件和 CPLD 的制造商 。该公司于1984年推出了其首款可编程逻辑设备。

两家一直是互为竞争对手，一般来说，大学里面都 Altera 的器件和 VHDL 上课，所以在学校里面用 Xilinx 的人比较少 。

因为原理是一样的，所以只要学会一种，另一种就很容易上手。学校实验室里面用的是 Xilinx 。

[Xilinx]: http://en.wikipedia.org/wiki/Xilinx
[Altera]: http://en.wikipedia.org/wiki/Altera

## Xilinx FPGA Architecture（架构）

FPGA 需要反复烧写，所以不能像 ASIC 一样通过固定的与非门来完成，只能采用一种易于反复配置的结构，查找表就可以很好的满足这一要求。目前主流 FPGA 都是采用了基于 SRAM 工艺的查找表结构（军品和宇航级 FPGA 采用 Flash或者熔丝/反熔丝工艺），通过烧写文件改变查找表内容的方法来实现对 FPGA 的重复配置。

查找表（look-up table，LUT）本质上就是一个 RAM 。当用户通过原理图或 HDL 语言描述了一个电路以后，FPGA 开发软件会计算逻辑电路的所有可能结果，并把真值表事先写入 RAM，这样，每输入一个信号进行逻辑运算就相当于输入一个地址进行查找，这样 LUT 就具有了和逻辑电路相同的功能 。实际上，LUT 具有更快的执行速度和更大的规模 。

上电时，FPGA将外部存储器中的数据读入片内RAM，完成配置后，进入工作状态；掉电后FPGA恢复为白片，内部逻辑消失。这样FPGA不仅能够反复使用，还无须专门的FPGA编程器，只需通用的EPROM、PROM编程器即可。

目前，Xilinx FPGA 仍然是基于查找表技术，但是其概念已经远远超出查找表技术的限制，并且整合了常用功能的硬核模块（如块 RAM，时钟管理和 DSP）。Xilinx FPGA 内部大致可以分为 6 部分：

### IOB

可编程输入/输出单元简称 I/O 单元，是芯片与外界电路的接口部分，完成不同电气特性下输入/输出信号的驱动和匹配 。

### CLB

CLB（Configurable Logic Block）是 FPGA 内的基本逻辑单元，Xilinx FPGA 的 CLB 由多个相同的 Slice 和附加逻辑组成 。

### DCM

业内大多数 FPGA 都提供数字时钟管理（Digital Clock Manager）。Xilinx FPGA 提供 DCM 和 PLL 。

### BRAM

大多数 FPGA 都具有内嵌的块 RAM，这大大扩展了 FPGA 的应用范围和灵活度 。块 RAM 可以被配置为单口 RAM、双端口 RAM、内容地址存储器（CAM）和 FIFO 等常用存储结构 。

### Routing Resource

布线资源连通 FPGA 内部的所有单元，而连线的长度和工艺决定着信号在连线上的驱动能力和传输速度 。Xilinx FPGA 的布线资源可以分为 4 类：全局布线资源、长线资源、短线资源、分布式资源 。

### Embedded Module

内嵌功能模块只要是指 DLL、PLL、DSP 和 CPU 等 **软核**，还有底层的 **硬核** 资源，比如内嵌的 Power PC、ARM9、DSP芯片等 。

***Xilinx 主流 FPGA***

Xilinx 主流的 FPGA 主要有A系列、K系列、V系列、Spartan系列，如今还有最新的 Zynq 系列，官网上有详细介绍 。

## 开发流程

FPGA 的开发流程如下图所示：

![design flow](/images/learning-fpga-getting-started/design-flow.jpg)

整个开发过程就是使用开发工具 ISE Design Suite，按照流程图进行 。图示是标准流程，但是实际上并不是严格按照每一个步骤进行 。

一般简化过的流程是

1. 设计可综合的代码

2. 综合 Systhesis

    前两步主要是确保写的代码是开发工具可以转化为实际电路。

3. 综合后仿真

    这一步保证模块的逻辑功能是正确的，即检验模块的结果是否和预期一致
    
4. 时序约束

    为设计添加时序约束和管脚约束
    
5. 实现 Implement

    按照约束条件将综合结果映射到实际器件中
    
6. 时序分析

    实际上，一次就能达到时序要求且布线成功的情况并不多，尤其是对于高速设计，所以需要根据上一步的时序结果对设计进行修改，以满足时序要求，类似于写软件的 Debug 。
    
7. 下载，在线调试

    将设计下载到芯片中调试。

## 参考

[PLD wikipedia][PLD]

[FPGA wikipedia][FPGA]
