Title: RISC-V Spec 阅读笔记 #1 —— Unprivileged ISA
Date: 2020-11-17 21:55
Category: RISC-V
Tags: RISC-V, Spec
Slug: risc-v-spec-notes-1-unprivileged
Author: Qian Gu
Series: RISC-V Notes
Summary: Volume I: Unprivileged ISA 读书笔记

!!! note

    Unprivileged ISA 文档的版本号:20191213

## Introduction

RISC-V 的目标可以说非常宏大、也非常务实，可以用这几个关键词来概括：完全 open、可实现、通用、模块化、可扩展。RISC-V 在定义时尽可能地规避了具体的实现细节（虽然 ISA 中有些设计是出于实现考虑的），所以这个 ISA 应该当成各种不同实现方案的统一软件可见接口，而非某种特定微架构实现的专属。整个手册分为两部分：

+ Volume I: Unpriviledge ISA
+ Volume II: Priviledge ISA

在设计这些 ISA 时都遵循了“尽量移除对特定微架构依赖”的原则，这样可以在简化 ISA 同时保证 implementation 最大程度的灵活性。

!!! tip
    ISA 作为软件和硬件之间的接口，其地位非常重要。曾经有很多各种各样的 ISA，其中大部分都随着历史消亡了，只剩下个别占领了市场主流，并不断演进。但是目前大部分 ISA 被商业产权保护，普通人无法使用，而且因为要向后兼容而有历史包袱，在这样的背景下，RISC-V 最早起源于 UC Berkeley 的教学需求，逐渐发展壮大，如今在业界如火如荼。

    定义一个新的 ISA 并不是简单定义指令集就足够的，还需要大量的投入，比如文档、编译器工具链、测试套件、教学材料等等，即使这些都全做出来了别人也不一定会用，做出来简单，要想推动整个生态是件非常难的事情。看看 RISC-V 基金的董事会和赞助商，就会发现全是著名科技公司和大佬，也只有他们才能集整个产业界的力量推动新的 ISA 发展。

ISA 之所以叫架构，是因为它作为软硬件世界之间的桥梁，不仅仅包含了指令集，还需要包含很多其他内容，比如：

+ 数据类型
+ 存储模型
+ 软件可见的处理器状态 GPR + CSR + PC
+ 指令集
+ 系统模型：状态 + 特权级别  + 中断/异常
+ 外部接口： I/O + 管理

### Hardware Platform Terminology

| 术语                   |      含义                                                                 |
| --------------------- | ------------------------------------------------------------------------- |
|  `hardware platform`  | RISC-V core + non-RISC-V core + accelerator + memory + I/O + interconnect |
|  `core`               | 包含独立的 IFU 的模块，一个 core 可能包含多个 `hart` 和 `coprocessor`     |
|  `coprocessor`        | 附着在 RISC-V core 上，大部分情况下由 RISC-V 指令流控制，但是具有架构状态和有限的自主控制权来运行扩展指令   |
|  `accelerator`        | 不可编程的固定函数单元 or 针对特定任务的可以自动运行的 core，如 I/O 加速器                         |

### Software Execution Environment and Harts

RISC-V 程序的行为依赖于它的执行环境，`EEI (Execution Environment Interface)` 定义了下面这些内容：

+ 程序的初始状态
+ Hart 的数量和类型
+ hart 支持的特权模式
+ memory/IO 的访问及特性
+ 每个 hart 支持的合法指令，即 ISA
+ 中断和异常的处理方式

EEI 的典型例子有 Linux `ABI (Application Binary Interface)` 和 RISC-V `SBI (Supervisor Binary Interface)`。

EEI 的实现方式有多种：

| 实现方式   | 含义 |
| ---------- | ----- |
| Bare metal | 由硬件直接实现的 hart，指令可以访问所有的物理地址空间，硬件平台定义上电复位后的执行环境 |
| OS         | 通过复用 hart 和虚拟地址控制访问 memory ，OS 可以实现多个用户层级的 execution environment（即多个 thread） |
| hypervisor | 对 guest OS 访问提供多个 supervisor 级别的 execution environment（即多个虚拟机） |
| emulator   | 在另一个硬件平台（如 x86）上模拟 RISC-V 的 hart，比如 Spike、QEMU、rv8 等 |

+ `Hardware Thread`：随着处理器技术的发展，现在的处理器已经突破了多核的概念，在单个核心中包含多个硬件线程的技术叫做硬件超线程 `Hyper-threading`，每个硬件线程有自己独立的 RF 等上下文资源，但是不同线程共享同一份运算资源，所以面积效率很高。
+ `Hart`：为了区别多核，RISC-V 定义的术语，取 Hardware Thread 之意，包含自动取指和执行所涉及的硬件资源的统称。

!!!note
   + 一般软件的接口都比实际硬件接口更加抽象，因为 EEI 越抽象移植性就越好。通常 EEI 都是一层一层堆叠在一起的，上层的 EEI 依赖于下层的 EEI。
   + hart 由 execution environment 提供，所以可能会出现 guest hart 数量比 host hart 数量多的情况，即时分复用。这种情况下，由 execution environment 确保每个 hart 行为正常，尤其是要支持 guest hart 抢占 host hart。

### ISA Overview

RISCV ISA 由必选的 Base Integer ISA 和其他可选 ISA 组成，完整的子集列表直接看 spec 即可。其中必选的 base interger ISA 和以前的 RISC 处理器的 ISA 非常相似，只是去掉了分支延迟槽并支持了可选的变长编码。I 子集是一个精心设计过的针对工具链来说合理的最小化集合，基于它可以实现一个基本的软硬件“骨架”，并进一步定制化。I 子集一共有 4 种形式，它们的区别在于 register 的位宽、register 的数量、寻址空间大小： 

| ISA    | XLEN (register 位宽) | registe 数量 | 寻址空间范围 (Byte) |
| ------ | -------------------- | ------------ | ------------------- |
| RV32I  | 32                   | 32           | $2^{32}$            |
| RV64I  | 64                   | 32           | $2^{64}$            |
| RV32E  | 32                   | 16           | $2^{32}$            |
| RV128I | 128                  | 32           | $2^{128}$           |

!!! tip

    4 个 base ISA 被当作完全不同的 ISA 来对待，所以有个常见问题：

    Q：为什么不设计一个统一的 ISA，即让 RV32I 是 RV64I 的严格子集？一些早期 ISA(SPARC, MIPS) 就采用了这样的设计规则，使得可以在 64bit 的硬件上运行 32bit 的程序。

    A：ISA 分开设计的主要优点是可以针对某个子集独立优化，不需要为支持其他子集而消耗资源，主要缺点则是当在一个 ISA 上模拟另外一个 ISA 时的硬件更复杂。实际上寻址模式和捕获非法指令的不同往往意味着即使某两个 ISA 是子集关系仍然需要两套电路以及某种模式切换，而且 RISC-V 的 base ISA 之间的相似性可以降低多版本的开销。虽然理论上可以把 32bit 的 lib 和 64bit 的代码链接在一起，但因为程序调用和系统调用接口的不同实际中并不可行。

    RISC-V 的特权架构中 misa 寄存器有个字段专门用来控制在同一份电路上可模拟的 unprivileged ISA，而且可以看到最新的 SPARC 和 MIPS 也放弃了对在 64bit 系统上直接运行 32bit 程序的支持。

    另外一个相关的问题是：为什么相同的 32bit 加法在 RV32I 和 RV64I 中的 opcode 不同？应该遵守字面意思，W 后缀表示 word（32bit），D 后缀表示 double word（64bibt），按如下方式编码，这样和 LW 的编码方式也能保持统一。

    | ISA   | 32-bit add | 64-bit add | 32-bit load | 64-bit load |
    | ----- | ---------- | ---------- | ----------- | ----------- |
    | RV32I | ADDW       | N/A        |  LW         | N/A         |
    | RV64I | ADDW       | ADDD       |  LW         | LD          |

    而目前的设计为：

    | ISA   | 32-bit add | 64-bit add | 32-bit load | 64-bit load |
    | ----- | ---------- | ---------- | ----------- | ----------- |
    | RV32I | ADD        | N/A        |  LW         | N/A         |
    | RV64I | ADDW       | ADD        |  LW         | LD          |

    实际上一开始确实是按照提议的方式编码的，但是在后来的演进过程中修改为目前的方式。之所以把 32bit 加法的编码设置成不兼容，目的是为了消除 RV32I 中的不对称性（RV32I 有 ADDW 和 AND 但是没有 ANDW）。从事后复盘看目前的方式是不合理的，主要原因有两个：

    + 同时设计两个 ISA，而不是是一层一层地叠加
    + 设计 ISA 时的一个理念：RV64I 必须完全包含 RV32I（RV32I 已经使用了 ADD，所以 RV64I 只能修改指令的含义，并新增 ADDW）

    因为 RV32I 和 RV64I 已经 frozen 了，所以无法修改了，目前看这样做几乎没有什么实际问题。以后可以将 W 后缀作为某个 RV32I 变种的扩展集，这样就能在 RV32I 和 RV64I 之间保持一致的编码了。

### Memory

+ 寻址范围是 $2^{XLEN}$ Byte
+ 1 word = 32bit(4Byte)；1 halfword = 16bit(2Byte)；1 doubleword = 64bit(8Byte)，1 quadword = 128bbit(16Byte)，和 XLEN 无关
+ memory 地址是循环的，最大的地址溢出后自动回到 0 地址，硬件计算地址时以 $2^{XLEN}$ 为模会自动忽略溢出即可
+ 由 execution environment 将硬件资源映射到 hart 的地址空间中，通常地址空间被分成了不同段，如 memory、I/O device 等，每个段有自己的属性
+ 多个 hart 的 memory 可以完全一样，也可以完全独立或部分重叠
+ memory 访问可以分为 implicit 和 explicit 两类
    + implicit access：取指操作，没有对应的指令，硬件自动读取。这种类型的访问可以提前进行（比如 I$ 指令预取），当需要和 store 数据同步时必须用 fence 或 cache control 指令显式地同步
    + explicit access：load/store 指令的操作
+ RISC-V 默认使用 `RVWMO(RISC-V Weak Memory Ordering)` 作为内存一致性模型，实现也可以选择约束更强的 TSO(Total Store Ordering) 模型

### Base ISA Encoding

RISC-V 指令可以是变长的，但是所有 base ISA 都按照 16bit 对齐，即其指令长度都是 16bit 的倍数。

使用术语 `IALIGN`（单位为 bits）表示指令对齐约束，IALIGN 的取值只能是 16 或 32：base ISA 的 IALIGN 是 32，C 子集和其他扩展 ISA 可以是 16。

使用术语 `ILEN`（单位为 bits）表示实现支持的最大指令长度，它永远是 `IALIGN` 的整数倍。

Base ISA 既可以是大端模式，也可以是小端模式，但是指令存储一定是以 16bit 为单位的数据包的小端模式。

### Exceptions, Traps, Interrupts

+ `Exception` 表示指令执行中处理器本身出现异常情况而停止执行当前程序
+ `Interrupt` 表示外部异步事件导致处理器停止执行当前程序，转而去完成其他事情，完成后再继续之前的程序
+ `trap` 表示由 exception 或 interrput 导致的 CPU 控制权向 trap handler 的转移过程

4 种不同的 trap：

|     | Contained | Requested | Invisible | Fatal |
| --- | --------- | --------- | --------- | ----- |
| Execution terminates ? | N | N | N | Y |
| Software is oblivious? | N | N | Y | Y |
| Handled by environment? | N | Y | Y | Y |

## RV32I Base ISA

一共有 40 条指令，查询 reference card 即可。

!!! tip

    一些资料中会描述说 I 指令集包含 47 条指令，这里有点歧义。以前的历史版本中 I 确实有 47 条指令，但是在最新的版本中，有 7 条指令被移了出来，只剩下只包 40 条指令，通常所说的 47 条指令是把另外两个必要的子集也包含在内：

    + `Zifencei` 子集：包含 1 条指令 `FENCE.I`
    + `Zicsr` 子集：包含 6 条 csr 相关的指令

    在一些实现中，可能会把 ECALL/EBREAK 当成一条永远 trap 的硬件指令来处理，且把 FENCE 指令当成 NOP 来处理，这时 I 子集的指令条数就缩水到了 38 条。

    除了 A 子集需要特殊的硬件来支持原子性操作之外，RV32I 基本上可以模拟任何其他子集。

### Programmers' Model

RV32I 的 Unpriviledged state 分为两类：

+ 通用寄存器 XRF：一共有 32 个，XLEN=32，其中 x0 为常数 0
+ `CSR`：内部寄存器，专有的 12bit 地址编码空间

还有个特殊的 register：`pc`。

I 子集只涉及通用寄存器，CSR 寄存器在后面的 `Zicsr` 部分介绍。

!!!tip
   RV32I 中没有定义 stack pointer 和 return address 寄存器，而是直接使用某个 XRF。标准的软件调用规范中指定了 x1 作为 return address，x2 保存 sp。遵守这个规范可以生成最小体积的代码。

!!! tip
    一个 ISA 中寄存器的个数对代码体积、性能、功耗有巨大的影响。到底应该设计多少个 register 也是有讲究的，有种意见是对于 I 子集只用 16bit 的指令编码配合 16 个 register 就已经足够了，但是如果指令中包含 3 个寄存器地址，则光地址就需要 12bit，只剩了 4bit 来编码 opcode，这基本上是不可能的。而如果指令只包含 2 个地址，那么实现相同功能就需要更多的指令，降低效率。为了简化硬件设计也应该避免 24bit 这种中间长度的指令格式，所以最终选择了 32bit 的指令来编码配合 32 个寄存器。寄存器数量多一些对性能提升也有帮助，比如 loop unrolling, software pipelining, cache tiling 这些技术都对寄存器数量有很大的需求。

### Format

主要有 4 种格式：

+ `R` (Register)
+ `I` (Immediate) 
+ `S` (Store)
+ `U` (Upper)

当 branch 或 jump 指令跳转的地址没有 4Byte 对齐时，会在 branch/jump 指令处产生 exception。当支持 16bit 指令时(IALIGN = 16)，对齐约束放宽到 16 的倍数。

RISC-V 的指令格式是精心设计过的，目的就是为了简化硬件的译码电路：

+ RV32I 的指令长度都是 32bit，按照 4Byte 对齐存储
+ 所有指令中的 rs1, rs2, rd 都在固定位置
+ 所有指令中的立即数都是按照有符号的方式扩展（除了 CSR 指令中的 5bit 立即数）
+ 所有立即数都放在指令可用空间的最左边 bit 位置
+ 所有立即数的符号位都在指令的固定位置 bit[31]

TODO：和蜂鸟总结的 dec 优势对比，查缺补漏

!!! tip

    译码模块中 register 标识符的逻辑通常都是关键路径，所以 RISC-V 在设计指令格式的时候，不管是什么格式类型的指令，都把标识符放在固定位置，付出的代价则是指令中立即数的位置会随着指令类型变化。

    实际上，大部分立即数的位宽要么很小，要么就要占满 XLEN bit，RISC-V 选择了一种非对称的方式切分立即数（用两条指令来搬运一个立即数：第一条指令搬运低 12bit，第二条指令搬运剩余的 20bit），这样做的好处是可以增加常规指令可用的 opcode 编码空间。

    所有立即数都是符号为扩展的，因为我们没有发现 MIPS 中按 0 扩展能带来什么好处，这样做同时也能最大限度地保持 ISA 的简洁。

因为立即数的原因，所以有了另外两个变种格式：

+ `B` (Branch) 类型：它和 `S` 一样都用 12bit 来编码立即数，唯一区别是 B 中的立即数是 S 中立即数的 2 倍，也就是说 S 中的 12bit 表示 imm[11:0]，而 B 中的 12bit 表示 imm[12:1]，而且这 12bit 在指令中的位置是精心设计过的，并不是简单移位
+ `J` (Jump) 类型：它和 `U` 的关系也是类似的，唯一区别是 U 和 J 需要 shift 的 bit 位数不一样，同理，立即数的位置是精心设计过的，并不是简单填充

!!! tip

    立即数的符号位扩展是其关键路径之一，RISC-V 把所有立即数的符号位都放在第 31bit，好处是可以让符号位扩展和译码并行。

    虽然有些复杂的实现会给 branch 和 jump 指令计算分配专有加法器，所以并不会受益于不同指令中的立即数位置固定的设计，但是我们想降低最简实现的硬件成本。通过变换立即数的 bit 位置，而不是使用动态的硬件 mux，指令信号的 fanout 和 mux 数量大概减少为原来的一半。而这种混乱的立即数编码带来的开销可以忽略不计。

下面分类描述 I 子集的 40 条指令，大概可以分成 4 类：

|   类型   |  数量  |
| ------- | ----- |
| 整数计算指令 | 21 |
| 控制转移指令 | 8 |
| Load/Store 指令 | 8 |
| Memory 顺序指令 | 1 |
| 系统调用和断点 | 2 |

### Integer Computational Instructions

整数计算指令只有两类：

+ `I-type`：register 和 immediate 相计算
+ `R-type`：register 和 register 相计算

两类都会有 rd 寄存器来保存结果，rs 和 rd 都为 XRF，而且都不会产生算术异常。

!!! tip

    我们并没有设计特殊的指令集来检测 overflow，因为可以用 branch 指令很廉价的实现这个功能。

    + 无符号数加法的 overflow 检查只需要在 add 指令后面加一条 branch 指令即可：

            #!text
            add t0, t1, t2
            bltu t0, t1, overflw

    + 有符号数加法，如果已知一个操作数的符号（I 类型加法），那么只需要在 add 后面加一条 branch 指令即可：

            #!text
            addi t0, t1, +imm
            blt t0, t1, overflow

    + 对于一般的有符号数 R 类型的加法，需要 3 条指令来检测求和结果是否比任何一个加数都小（除非一个操作数是负数）

            #!text
            add t0, t1, t2
            slti t3, t2, 0
            slt t4, t0, t1
            bne t3, t4, overflow

注意：

+ `SLTIU` 指令需要先对立即数进行符号位扩展，然后再当成无符号数来比较
+ `NOP` 是伪指令，以 `ADDI x0, x0, 0` 的方式实现
+ `LUI` 和 `AUIPC` 都是 U 类型的格式，用来组装一个 32it 数据，区别是 `LUI` 只做组装，而 `AUIPC` 完成组装后还会有一个额外的加法

!!! tip

    `AUIPC` 指令支持以“双指令序列”的方式访问当前 PC 的任意 offset 位置，既可以用来做控制流的转移，也可以用来做数据访问，可以访问相对于当前 PC 值的任意 32-bit 地址。

    + 控制流转移： AUIPC + JALR 组合，AUIPC 先把目标地址的高 20bit 存储到 rd 中，然后这个 rd 作为 JAL 的 rs，叠加上 imm 就可以算出完整的 32bit 目标地址
    + 数据访问： AUIPC + LOAD/STORD，AUIPC 先把目标地址的高 20bit 存储到 rd 中，然后这个 rd 作为 ld/st 的 rs，叠加上 imm 就可以算出完整的 32bit 目标地址
    
    TODO：看蜂鸟，补充实际指令序列的例子
    
    原理：AUIPC 已经包含了立即数的高 20bit，所以只需要其他指令配合产生立即数的低 12it 即可。

    虽然获取当前 PC 值可以通过把 AUIPC 的立即数设置为 0 来实现，也可以通过 `JAL +4` 的方式实现，但是后者的问题在于可能会导致流水线停顿或者是污染 BTB。

!!! tip

    一般 NOP 指令都用来处理地址边界问题以使得指令对齐，或者为指令修改预留空间。虽然 NOP 的实现方式有很多种，我们只定义了一种作为示范，给微架构层面的优化留有空间，同时也可以使得汇编代码的可读性更好。NOP 指令也可以用 HINT 指令来实现。

    选用 ADDI 来实现 NOP 的原因是，这样占用的资源最少（除非是在 decode 阶段把它优化掉了），只需要读一个 register，而且在超标量处理器中加法器是最常见的操作，AGU 可以像计算地址一样直接执行这条指令，其他的 register-register 指令（比如 ADD 或 logical 指令、shift 指令）都需要额外的硬件才能完成。

### Control Transfer Instructions

控制流相关的指令一共有两类，而且都没有 delay slot（延迟槽），如果跳转地址没有对齐，则会产生一个不对齐异常。

+ 无条件跳转 jump
+ 有条件分支 branch

因为 `JAL` 指令属于 J-type，所以它包含了 imm[20:1] bit，所以可以跳转的范围是 [-1MB, +1MB] 内。JAL 会把自己后面的指令地址 (pc+4) 存到 rd 中，方便跳回后继续执行。一般标准的软件调用惯例是 rd = x1 作为返回地址，x5 作为 alternate link register。

!!! tip
    这个 alternate link register 可以在保留常规的返回地址寄存器 rd 不变前提下支持调用一些 millicode，之所以选择 x5 是因为在标准调用中它是一个临时寄存器，而且和 x1 的编码只有 1bit 不同。

!!! tip
    无条件 jump 指令（JAL 和 JALR）都使用 PC 的相对地址来支持地址不相关的代码。JALR 和 LUI 组合在一起可以访问 32bit 地址空间中的任一位置，首先 LUI 把目标地址的高 20bit 搬运到寄存器中，然后 JALR 把低 12bit 加上去就可以算出完整的 32bit 目标地址。同理，AUIPC 和 JALR 也可以跳转到相对于 PC 的任意 32bit 地址。

    需要注意的是，JALR 不会像 branch 指令一样从 imm[1] 开始编码（2 的倍数），JALR 编码的是 imm[11:0]，这样做的好处是可以避免硬件中立即数格式太多的问题。

    JALR 执行的时候，会把计算出的 target address 的 LSB 清零，这样做的好处是可以稍微简化硬件设计，同时还可以空余出 1bit 空间来存储更多辅助信息。虽然这么会导致错误检查的逻辑变复杂，所以有些轻微的性能损失，因为发生了指令地址错误很快就触发异常，所以问题不大。

    当 rs1=x0 时，JALR 可以用来实现单指令调用子程序，在 [-2KB, 2KB] 范围内跳转，可以实现 small runtime lib 的快速调用。

RAS 预测是高性能 IFU 中的常见功能，但是前提是要能准确区分出函数调用和返回，协议规定了 JAL 和 JALR 所使用的寄存器序号可以用来辅助 RAS 预测：

+ 如果 JAL 的 rd=x1 或者是 rd=x5，那么就是函数调用，要把 rd 寄存器的值 push 进 RAS，保存返回地址
+ 如果 JAL 的 rs=x1 或者是 rs=x5，那么就是函数调用返回，要从 RSA pop 出返回地址，用于 IFU 预测下一个地址

遵守协议的规定，compiler 和 core 配合就可以最大化地提高 RAS 的预测准确度。

!!! tip

    有些 ISA 使用了特别的 bit 位来标识辅助 RAS，我们使用隐式的方式（约定寄存器号）来减少对编码空间的占用。

所有的 branch 指令都是 B-type，所以它编码了 imm[12:1] bit，所以可以跳转的范围是 [-4KB, 4KB] 之间。

协议规定软件要假设硬件是 BTFN 算法的方式，依次进行优化，这样可以提高低端 CPU 的预测性能。不同于其他 ISA，RISC-V 规定无条件跳转必须使用 JAL(rd=x0)，而不能用条件设置为 true 的 branch 指令。因为 jump 指令要比 branch 的跳转范围大，而且不会污染条件 branch 的预测表。

!!! tip

    条件 branch 指令被设计成包含两个 register 的算数比较的方式（同 PA-RISC、Xtensa、MIPS R6），没有使用以下方式：

    + 使用条件码 condition code（x86、ARM、SPARC、PowerPC）
    + 只使用一个 register 和 0 做比较（Alpha，MIPS）
    + 两个寄存器只用于比较是否相等（MIPS）

    这样设计的主要原因是把比较和分支合并在一起更加适合常规流水线，不需要使用额外的 condition code，也不需要使用寄存器保存中间结果，可以减少代码体积，降低 IFU 的带宽，还可以在 IF 阶段就被提前检测到，即使是和 0 比较这种设计，也会引入不可忽略的 latency。这样设计付出的硬件代价也很小近似可以忽略。另外融合的指令可以在流水线的上游更早地观测到，更早地预测。

    曾经考虑过在指令编码中加入静态分支提示，但最终并没有加，理由是虽然静态分支提示可以缓解动态预测器的压力，但是代价是会占用更多的编码空间，还需要软件做 profiling 才能获得最好的结果，而一旦 profiling 和实际不一致，性能就很差。

    没有包含类似 ARM 条件码的原因是：条件码需要占用指令的额外 bit，需要额外的指令来设置/清除，增加了硬件复杂度，而和它一起配合使用的静态预测的效果可能并不好。

    有很多方法可以动态地把不可预测的前向分支转化为可预测的代码，以避免流水线的 flush 代价。其中最简单的就是只 flush IFU 中的指令，而不是整个 pipeline 上的所有代码。

### Load and Store Instructions

RISC-V 是一个 load-store 体系结构，即只有 load/store 才可以访问 memory，计算指令只能和寄存器打交道，而且 RISC-V 的端序是 byte 地址不变的。

!!! warning
    + 如果 load 指令的 rd 是 x0，即使读回来的数据被丢弃了，也必须报一个任意类型的 exception

在一个端序是 byte 地址不变的系统中，有下列特性：如果某个 1 byte 的数据被 store 在某种端序的 memory 的某个地址中，那么从那个地址中 load 1 byte 数据返回的数据也是那个值。

+ 小端系统：一个多 byte 数据，LSB 被存储在 memory 的低位地址，剩余数据的地址按照顺序递增，load 指令会把低位地址的数据搬运到 register 的 LSB 中
+ 大端系统：一个多 byte 数据，MSB 被存储在 memory 的低位地中，剩余数据的地址按照顺序递减，load 指令会把高位地址的数据搬运到 register 的 LSB 中

RV32I 的地址空间是 32bit，按照 byte 地址访问，由 EEI 规定合法的地址段。无论端序如何，如果访问地址是天然对齐的，那么就不会产生任何异常，如果访问地址不是天然对齐的（访问数据量是32bit，地址按照 4byte 对齐；访问数据量是16bit，地址按照 2byte 对齐），那么具体行为取决于 EEI。EEI 可以允许非对齐访问，由硬件或者软件处理，也可以不允许非对齐访问，直接抛出异常。

!!! tip
    非对齐访问在移植旧代码、使用 packed-SIMD 扩展的应用程序、处理外部打包的数据结构时很有用。之所以通过 load/store 来允许 EEI 自主选择非对其访问的处理方式，目的就是想简化硬件设计。

    有一种备选方案：在 base ISA 中不允许非对齐访问，额外再设计一个 ISA 来支持非对齐访问，比如某些特殊指令或者是硬件特殊的寻址模式。这个方案的问题在于：

    + 特殊指令使用难度大，导致 ISA 复杂化
    + 要么处理器添加了额外状态（CSR），要么导致现有 CSR 的访问复杂化
    + 面向 for 循环的 packd-SIMD 程序可能要根据数据对齐模式修改多个版本的代码，使得代码生成复杂化，产生额外开销
    + 新的硬件寻址模式必然要消耗大量的指令编码空间，而且也要消耗一些硬件资源来实现

即使实现了非对齐访问，在某些实现中可能性能很差；而且硬件处理非对齐访问时可能会将其拆分成多个子指令来处理，此时需要额外的同步机制来保证访问的原子性。

!!! tip
    标准中非对齐访问的原子性不是必须的，这样 EEI 就可以自由选择是用不可见的 machine trap 还是软件 handler 来处理非对齐访问。如果硬件支持非对齐访问，那么软件只需要直接用简单的 load/store 即可，发生非对齐访问时，硬件会自动优化。

### Memory Ordering Instructions

RISC-V 支持在一个单一的用户地址空间内运行多个 hart，每个 hart 都有自己的 pc 和 register，执行自己的指令流。而由 EEI 来完成 hart 的创建和管理。不同 hart 之间可以通过共享存储器来实现通信和同步，又因为 RISC-V 使用存储器松散一致性模型 `RVWMO`，所以需要 FENCE 指令来定义另外一个 hart 观测到本 hart 的指令执行顺序。从原则上讲，另外一个 hart 只能先观测到 FENCE 前的指令行为，然后才能观测到 FENCE 之后的指令行为，即 FENCE 像一道屏障一样，隔断了前后的指令流。

RISC-V 把数据存储器访问分为了 4 类：

+ I：设备读 device-input
+ O：设备写 device-output
+ R：存储器读 device-read
+ W：存储器写 device-write

配合前后的概念，所以可以实现很多种组合，达到非常精细的控制。

FENCE 的 fm 字段取值为 1000，predecessor = RW，successor = RW时，为 FENCE.TSO 指令，效果为 predecessor 的 load 和 successor 所有 memory 操作保序，predecessor 的 store 和 successor 的所有 store 保序，但是并未规定 predecessor 中的 non-AMO store 和 successor 的 non-AMO load 之间保序。

!!! tip
   FENCE.TSO 是 FENCE 的可选扩展，因为 FENCE 的基本定义要求忽略 I/O 和 memory 字段全局实现 fence，所以 FENCE.TSO 和 FENCE 是兼容的。

FENCE 中没有定义的字段 rs1 和 rd 是为了将来更细精度的扩展而保留的。为了保持前向兼容，硬件应该忽略这些 bit 位，同时软件应该将这些 bit 位设置为全 0。

!!! tip

    我们选择松散一致性模型的目的是让一个简单的微架构能获得高性能的同时也方便未来进行扩展。将 I/O 和 memory 操作分离开来的好处是可以避免不必要的串行化。一个简单的微架构可以忽略 FENCE 中的前序和后序字段，保守地把所有的 FENCE 指令都当成最严格的 FENCE 指令来执行即可。

### Environment Call and Breakpoints

系统指令用来访问系统功能，可能会要求特权访问，全部是 I 类型的指令。大概可以分为两类：

+ 原子性地 read-modify-write CSR 的指令
+ 其他特权指令

!!! tip
    系统指令的目的是使得简单实现中可以触发软件 trap handler 实现功能，而一些高端实现可能直接用硬件实现该指令。

ECALL 和 EBREAK 都向 EEI 会出发一条精确异常来请求复位。

+ `ECALL` 用来向 EEI 发送服务请求，请求的参数则一般放在寄存器文件中
+ `EBREAK` 用来把控制权转移给 debugger

### HINT Instructions

HINT 指令一般用来给微架构传递性能提示。RV32I 给 HINT 预留了大量的编码空间，且全部用 rd = x0 的计算指令来表示。所以 HINT 和 NOP 类似，只会导致 pc 向前移动以及改变性能计数器，除此之外不会改变硬件架构中任何可见的状态。实现中直接把 HINT 忽略也是符合标准的。

!!! tip
    HINT 设计成这样的目的是方便硬件实现。简单实现中既可以把 HINT 当成一条恰好并不会产生任何影响的指令来走完所有 pipeline stage，也可以直接把它丢弃。

    虽然 HINT 编码空间很大，而且划分了 standard 空间和 custom 空间，但是目前还没有定义好 standard HINT。

## Zifencei

!!! tip
    先讨论一个问题：假设一个指令流，前面的有条指令向 memory 某个地址 store 了一个新值，后面需要从该地址取指，那么后面的指令一定可以读取到前面的新值吗？

    答案是：不一定。因为 pipeline 是有深度的，假设前面的这两条指令是背靠背的，那么前序指令结果还没写入的时候后续指令已经完成了取指、译码，开始执行了，所以后续取指的是旧值。

    为了解决这个问题，引入了 FENCE.I 指令，用来约束 store 和取值之间的顺序关系。

这个子集只包含一条指令 `FENCE.I`，它可以实现对同一个 hart 的 instruction memory 的写指令和取指之间的显式的同步控制，目前这是确保 hart 内部 store 对 instruction fetch 可见的唯一标准机制。

这条指令用来同步一个 hart 的 data 和 instruction 之间的关系，如果没有这条指令，RISC-V 就无法保证后续的取值操作能观测到前序的 store 结果。因为 FENCE.I 只用来处理单个 hart 内部的关系，所以如果有多个 hart，为了保证某个 hart 的 store 结果可以被其他 hart 观测到，应该在 FENCE.I 之前先调用一条 FENCE 指令：

+ step1：本 hart 完成 store 指令
+ step2：本 hart 执行 fence 指令
+ step3：本 hart 请求所有 hart 执行 fence.i 指令

!!! tip
    为了支持各种不同的实现，FENCE.I 指令做过精心的设计。简单实现可以直接 flush 流水线，清空 I-cache 即可。复杂一些的实现可能会有更高级的操作。

    FENCI.I 之前是 I 子集的一部分，但是因为下面两个原因，从 I 子集中挪了出来，不再是必须实现的指令了。

    + 在某些系统中，实现 FENCE.I 的代价很大。比如有些设计中 I-cache 和 D-cache 都是 incoherent 形式，一旦遇到 FENCE.I 指令，就必须清空两个 cache。如果在共享 cache / memory 之前有多级独立的 I/D cache，这个问题会更加严重
    + 在类 Unix 系统中，这条指令并没有强大到足以在 user level 使用。因为 FENCE.I 只能处理 hart 内部的同步，但是 OS 可能会在遇到 FENCE.I 时重新调度 user hart，所以现在 Linux ABI 都是要求产生一个系统调用来保证取指的 coherent

## RV32E

RV32E 是专门为嵌入式 Embedded 设计的 ISA，目前还是 draft 1.9 状态，它和 RV32I 的唯一区别就是把 register 的数量减少到了 16 个。

!!! tip

    实际上一开始是拒绝专门设计一个新的子 ISA 的，但是后来考虑到 32bit MCU 的需求，最终定义了这个子集，将来可能还会有 RV64E。

    我们发现在 RV32I 的小核中，高 16 个 register 大概占了 core 面积的 1/4（除 memory 之外），所以去掉这 16 个 register 可以节省大约 25% 的面积和功耗。

    这个改变也对软件的调用惯例和 ABI 提出了要求。

    RV32E 可以和任何标准 extension 组合使用。曾经考虑过 F、D、Q 和 RV32E 配合时，定义一个新版的只有 16-entry 的 FP register file，但是最终放弃了，计划定义一个 `zfinx` 扩展 ，它计算 FP 时直接使用 integer register file，所以省去了 FP register file 和 XRF 之间的搬运操作。

RV32E 的 ISA 和 RV32I 完全一样，但是因为 register 只有 16 个，所有指令中 index 的字段可以释放出几 bit，未来的标准指令扩展都不会用到这些字段，所以可以给自定义扩展指令来使用。

## RV64I

RV64I 是 RV32I 的扩展。和 RV32I 的区别如下：

+ `XLEN` 变成了 64，即 register 的位宽是 64bit。
+ RV64I 的大部分指令的操作数位宽都是 XLEN bit，但是也有一些附加的指令来操作 32bit 的数，这些指令都在 opcode 后面加了 `W` 后缀来区分（W = word）。这些 `*W` 指令会忽略掉高 32bit 数据，产生的结果也只保留 32bit 结果，然后把它符号位扩展到 64bit 后保存在 register 中。

## RV128I

目前还是 draft 1.7 状态，定义这个子 ISA 的目的就是为了支持更大的地址空间。

!!! tip

    目前暂时还不是很清楚我们什么时候需要比 64bit 更大的地址空间，世界上 Top500 的超级计算机拥有超过 1PB 的 DRAM，这需要超过 50bit 的地址线。一些仓储级的计算机已经包含了更大的 DRAM，而且固态硬盘和 interconnect 技术的发展可以能会产生更大地址空间的需求。万亿级别的系统研究需要 100PB 的空间，大概占用 57 根地址线。根据历史增长速度看，大概在 2030 年前就有可能超过 64bit 的范围。

    历史证明，无论何时只要出现地址不够用的情况，architect 设计者们就会重复以前的争论，使用各种技术（如 segmentation，96-bit 地址，软件规避等方法）来扩充寻址范围，但是最终，128bit 的寻址方案将作为最简单、最佳解决方案而被采用。

RV128I 在 RV64I 的基础上定义，就如同 RV64I 在 RV32I 上定义一样，保留了 `*W` 指令不变，只不过新增了如何在 128bit 寄存器中操作 64bit 的指令，用 `*D` 来表示。

!!!note
    为了提升和 RV64I 的兼容性，和 RV64I 兼容 RV32I 的做法相反，可能会把 RV64I ADDI 的指令编码定义为 RV64I ADDD，然后在之前的 opcode `OP-64` 新增一个 RV128I ADDQ，同时将 `OP-64` 的编码定义为 `OP-128`。即目前的编码规则：

    1. 不带后缀的指令操作数位宽为 XLEN
    2. 通过后缀 `W` 和 `D` 指定操作数位宽为 word 或 doubleword

    未来的编码规则：

    1. 不带后缀的指令从操作数位宽为 word
    2. 指令中带后缀 `W` 和 `D` 指定操作数位宽为 word 或 doubleword

## Zicsr

RISC-V 专门定义了一组 (control and status register, CSR) 寄存器来记录配置和运行状态，这些寄存器是内部寄存器，使用专有的 12bit 地址编码空间。

!!! tip

    CSR 主要是在 Priviledge 架构中使用，但是 Unpriviledge 架构中也会用到一些，比如计数器和计时器，浮点状态等。

    因为计数器和计时器等不再是 base ISA 中的必须强制性实现的了，所以访问这些 CSR 的指令就独立出来自成一章。

只要程序中有指令会修改或者是行为受 CSR 影响，那么就会发生隐式或是显式的 CSR 访问。比如说，在某些修改或受 CSR 影响的指令执行完之后，后续修改 CSR 或受 CSR 影响的指令执行之前，会产生 CSR 访问。

+ 对于一条带有 CSR 读的指令，在指令执行之前会先读回 CSR 的状态
+ 对于一条带有 CSR 写的指令，在指令执行之后会更新 CSR 的状态

CSR 的访问是 weakly ordered，所以其他 hart 观测 CSR 的顺序可能和程序中的顺序不一样，特别是，CSR 和 memory 的访问没有顺序关系，除非 CSR 会修改访问 memory 指令的执行，或者是使用特权指令集中定义的 Memory-Ordering PMAs 做了明确的顺序要求。对于 FENCE 指令来说，CSR 读被当作设备输入(I)，CSR 写被当作设备输出(O)。

!!! tip

    CSR 按照第二卷中 Memory-Ordering PMAs 章节定义的 weakly ordered memory-mapped I/O region 模型操作，所以 CSR 的访问顺序和 memory-mapped I/O 访问的约束一致。

    对 CSR 的顺序设置约束的原因是为了支持对主存及 memory-mapped I/O 访问和 time CSR 的排序。除了 `time`, `cycle`, `mycle` 这三类 CSR，两卷标准中定义的其他 CSR 都对其他 hart 不可见，所以这些 CSR 的访问顺序和 FENCE 的顺序可以是任意顺序，都是符合标准的。

## Counter

RISC-V 定义了 32 个 64bit 的 Unpriviledge 只读性能计数器和计时器，其中前 3 个有特定的含义：

| CSR 名字     | 指令                                                          |
| ----------- | ------------------------------------------------------------ |
| **CYCLE**   | 保存了 hart 执行到当前的周期数，使用 RDCYCLE[H] 来读取低/高 32bit    |
| **TIME**    | 保存了 hart 的 wall-clock 周期数，使用 RDTIME[H] 来读取低/高 32bit |
| **INSTRET** | 保存了 hart 退休的总指令数量，使用 RDINSTRET[H] 来读取低/高 32bit    |

可以通过一组非特权指令通过 0xC00-0xC1F 地址访问（高 32bit 可以用 0xC80-0xC9F 访问）这些计数器/计时器。

!!! tip
    这个子集之前是在 RV32I 中，后来分离出来单独作为一个子集，目前是 draft 2.0 版本，目测以后也不会大改。

    在 V2.2 版本中这 3 个 CSR 指令作为 RV32I 的一部分是强制实现的，但是现在独立后就不是强制性的了，但是还是推荐实现这 3 个 CSR，因为它们对于基本的性能分析、自适应、动态优化来说是必不可少的。而且也应该以较小的代价实现其他计数器，来辅助诊断性能问题。

    这些计数器必须是 64bit，即使是 RV32I 也是如此，否则位宽太少软件很难判断是否发生了 overflow。（简单算一下就知道 64bit 的表示范围非常大，可以覆盖任何性能计数的需求）

剩余的 29 个 CSR 名称为 `hpmcounter3`-`hpmcounter31`（hardware performance counter），它们作为可编程的计数器能针对某些事件进行计数，这些计数器可以通过特权指令来配置。

!!! tip
    特权架构中定义了访问和配置这些 hpm 的 privileged CSR，这些 hpm 可以用来对一些硬件指标做统计，比如执行的 F 指令条数，L1 I$ 的 miss 次数等等。

## M Extension

把乘除法指令从 I 子集中分离出来，可以简化低端 core 的实现，或者对于那些乘除法不常出现的应用，用另外一个加速器来实现。

如果同时需要乘法结果的高低两部分，推荐使用下面的顺序，同时保证 rs1 和 rs2 的值和顺序不变，且 rdh != rs1 or rs2：

```asm
MULH[[S]U]  rdh,  rs1,  rs2
MUL         rdl,  rs1,  rs2
```

这样硬件实现时会把它们融合成一条指令来执行，只进行一次乘法。

同理，除法也可以按照类似的方法实现，硬件实现时只会进行一次除法：

```asm
DIV[U]  rdq,  rs1,  rs2
REM[U]  rdr,  rs1,  rs2
```

!!! tip
    + MUL 不需要（没必要）区分 signed/unsigned，MULH 则需要区分 singed/unsigned
    + 大多数 ISA 中除以 0 都会触发异常，跳转到 trap，但是 RISC-V 做了简化，不会产生异常，可以简化硬件实现
    + RISC-V 的所有的计算指令都不会产生 exception
    + RISC-V 虽然不会产生 exception，但是对除 0 的结果做了特殊规定：商的所有 bit 设置为全 1
    + 设置为全 1 是精心设计过的：对于无符号除法来说，全 1 就是最大值，而且也天然符合硬件除法器的输出，无需任何修改；对于有符号除法来说，一般都是先转成无符号来算，所以也没有问题。这样设计可以最大程度地简化硬件设计。

## A Extension

A 子集包含了对 memory 进行原子性的 read-modify-write 操作的指令，目的是同步多个共享同一个 memory 空间的不同 hart。
A 指令支持 unordered, acquire, release, sequentially 等各种内存一致性模型，这些指令可以让 RISC-V 支持 RCsc 存储器一致性模型。

!!! tip
    经过大量辩论后，编程语言社区和体系结构社区最后达成一致：用 release consistency 作为存储器一致性模型，所以围绕着这个模型建立了 A 子集。

    背景知识：3 种最有代表性的内存一致性模型

    | 模型                                      | 含义 | 访问 memory |
    | ---------------------------------------- | ---- | -----------|
    | 按序一致性模型 Sequential Consistency Model | 每个 hart 看到的都是 program-order | 原子性、串行化 |
    | 松散一致性模型 Relaxed Consistency Model | 每个 hart 可以改变自己的存储器访问顺序 | 性能、功能 balance |
    | 释放一致性模型 Release Consistency Model | 支持 acquire-release 机制 | 更加松散 |

已知 base 子集使用的是 relaxed 模型，配合 FENCE 指令可以实现额外的顺序控制。但是为了更加高效地支持 release 模型，每个 A 指令都新增了两 bit：

+ `ac` 字段：表明本指令是 acquire 访问（上锁），屏障了它之后的 memory 操作，即在它之后的 memory 访问必须要等到这条 A 指令结束后才能开始
+ `rl` 字段：表明本指令是 release 访问（释放锁），屏蔽了它之前的 memory 操作，即它必须等前面的 memory 访问都结束了才能开始

如果两个字段都是 0,则这条 A 指令没有任何附加的约束；如果两个字段都是 1，则这条 A 指令是 sequential 模型，只能在前序指令结束、后序指令开始前执行。

A 子集主要包含两大类指令：

+ Atomic Memory Operations, `AMO`

    这些指令都是原子性的，也就是说读回数据和写回结果这段时间内这个地址不允许其他任何指令访问，也就是对这个地址“上锁”。AMO 指令要求必须是对齐访问（32-bit 数据按照 4-byte 对齐；64-bit 数据要求按照 8-byte 对齐），否则会产生异常。

    !!! tip
        设计 AMO 指令的原因是在高度并行系统中它比 LR/SC 指令的伸缩性更好。对于简单实现，可以直接用 LR/SC 来等价实现 AMO；而对于一个复杂实现，可以在 memory 控制器中实现 AMO 指令，而且如果 rd = x0 时可以优化掉读 memory 的操作。

    AMO 可以用来实现 memory 中的 parallel reductions，还可以用来更新 I/O 设备映射出的寄存器（比如 setting, clearing, toggle 比他位）。

    通过设置 ac/rl 字段，AMO 可以用来高效地实现 C11/C++ 11 中的 atmic memory operations。虽然 `FENCE R, RW` 可以实现 acquire 效果，`FENCE RW, W` 可以实现 release 效果，但是和 AMO 比起来引入了不必要的额外约束，所以 AMO 实现更加高效。

    下面是一个自旋锁的例子，整个流程就是 test-and-test-and-set，先测试锁是否可获取，如果可以就尝试获取，尝试完了测试是否获取成功，获取成功了就开始执行核心代码，核心代码执行完后再释放锁。

        #!text
            li          t0, 1   # initialize swap value
        again:
            lw            t1, (a0)      # load lock, lock is stored in a0 address
            bnez          t1, again     # check if lock is held
            amoswap.w.aq  t1, t0, (a0)  # attempt to acquire lock
            bnez          t1, again     # check agian
            # ...
            # critical section.
            # ...
            amoswap.w.r1  x0, x0, (a0)  # release lock

+ Load-Reserved/Store-Conditional, `LR`/`SC`

    和 ARM 中的 Load-Exclusive/Store-Exclusive 类似。


## RVWMO Model

## F Extension

TODO

## D Extension

TODO

## Extending RISC-V

## ISA Extension Naming Conventions
