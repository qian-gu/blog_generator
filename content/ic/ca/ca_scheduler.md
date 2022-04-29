Title: CPU 关键技术 —— Tomasulo
Date: 2021-01-13 19:21
Category: IC
Tags: CPU, Tomasolu
Slug: cpu_scheduler
Author: Qian Gu
Series: CPU 关键技术
Status: draft
Summary: 总结 Tomasolu 细节

1. 说明静态调度和动态调度算法含义
2. 比较优缺点

## Why need Tomasolu

为了尽可能地获得高性能，人们发明了动态调度算法，比如 scoreboard，但是 scoreboard 无法充分发挥指令的并行性，当前指令的 FU 发生结构冒险时会暂停 issue，导致后续没有结构冒险的指令也无法执行。Tomasolu 算法利用寄存器重命名算法可以避免结构冒险导致的流水线暂停，还可以解决 WAW 和 WAR，一举两得。

## Register Renaming

分析数据冒险 WAW 和 WAR 的产生原因，就可以知道是因为前后两条指令要操作同一个寄存器，而更深层次的根本原因就是因为 ISA 规定的 register 太少，所以不同指令不得不复用相同寄存器。假设如果有无数个寄存器可用，一条新的指令需要写回结果，给它分配一个新的寄存器即可,那么就不存在 WAW 和 WAR 了。现实中我们不可能真的有无数个寄存器，但是我们可以退而求其次，假设我们实际上的寄存器数量比 ISA 规定的多，那么在条件满足的情况下硬件上“偷偷把指令使用的寄存器重命名成另外一个寄存器”，让软件看不到这一细节，那么就能在不改变任何软件的前提下，解决 WAR 和 WAW。

所以寄存器重命名的核心思想就是：**把指令中的寄存器标号看作是位置 locations，而不是名字 names**：

+ locations 比 names 多
+ 动态地把 names 映射到 locations 上
+ 映射表 map table 保存了具体的映射关系
	+ 写寄存器：分配新 location，并且在映射表 map table 中记录相关信息
	+ 读寄存器：在 map table 中查找最近一次该 name 映射的 location
	+ 小细节：必须在合适的时间点回收映射表项，即 de-allocate

## Dynamic Scheduling

调度算法基本可以分为两大类：

+ 静态调度 static scheduling
+ 动态调度 dynamic scheduling

scoreboard 是一种动态调度算法，它可以检测所有类型的冒险，动态地调度不同功能单元执行指令。但是 scoreboard 的问题有两个：

+ 结构冒险会导致流水线暂停
+ 无法消除 WAW 和 WAR，也会导致流水线暂停

为了解决这个问题，动态调度算法（绝大多数是 Tomasolu 及其变种）做了改进。其核心思想是：**前序指令的阻塞不能影响后序指令的执行**。为了实现这个目标，具体实现方式就是：**把译码阶段拆分成两个阶段，译码和读操作数**。译码阶段检查结构冒险，读操作数阶段检查数据冒险，把这两类冒险解偶。为了防止前序有数据冒险的指令阻塞后续不相关指令，就必须找个临时存储的地方把前序指令保存起来，这样就可以把流水线腾出来执行后序指令，而这个临时存储的地方就是 `保留站`。

## What is Tomasulo

Tomasolu 算法是一种硬件动态调度算法，可以实现乱序执行，从而更加充分地使用多个执行单元。它是 Robert Tomasolu 于 1967 在 IBM 发明的，首次使用在 IBM System/360 Model 91 的 FPU 上。他也因为这个发明在 1997 年获得了 Eckert-Mauchly 奖。

Tomasolu 的主要发明包括了三部分，

+ 硬件重命名的方法
+ 给每个执行单元分配一个保留站 (reservation station)
+ 一个广播数据的通用数据总线 (Common Data Bus)

### Common Data Bus

`CDB`(Common Data Bus) 是数据总线，它把保留站和功能单元直接连起来，可以“在提高并行性的同时保持了优先权”。
功能单元访问数据时不再需要涉及寄存器，中间数据不必经过寄存器倒一次手，而是由 CDB 把结果直接广播到总线上，保留站监听总线，得到有效数据后就可以开始执行，避免了寄存器访问的仲裁。

### Reservation Station

`RS` (Reservation Station) 保留站也叫做调度器 (scheduler)，每个功能单元都有自己的保留站，所以它是一种分布式的控制方案，而 scoreboard 是一种集中化管理。当一条指令被发射到保留站时，会同时检查源操作数状态，

+ 如果寄存器中的源操作数已经准备好了，那么直接把源操作数写入到 RS 中（Vⱼ 和 Vₖ）
+ 如果寄存器中的源操作数还没有准备好，那么在保留站中存下产生这个数的 RS 序号（Qⱼ 和 Qₖ）

这样，RS 中就保存了所有的必要信息。只需不断监听 CDB，等待数据 ready 后把数据更新到 RS 中，然后就可以真正发射给功能单元开始执行。

RS 的数据结构是一张表，表的每一个 entry 包含下面 7 个字段：

| 符号 | 含义 |
| ---- | ---- |
| Op | 要执行的操作 |
| Qⱼ, Qₖ | 产生源操作数的 RS 标号（0/blank 表示数据已经 ready，保存在 Vⱼ, Vₖ 中） |
| Vⱼ, Vₖ | 源操作数的值 |
| Busy | 1 = 被占用, 0 = 没有被占用 |
| A | Load / Store 的内存地址 |

其中需要保存 A 字段的原因是有些指令并不是寄存器寻址，而是使用立即数，比如 RISC-V 中的 I 类型指令（比如 Load / Store 指令)，所以要分配一个字段来保存这个立即数。

```
#!text
// RS 的数据结构

  +------+------+------+------+------+------+------+
  | Op   |  Qⱼ  |  Vⱼ  |  Qₖ  |  Vₖ  | Busy |  A   |
  +------+------+------+------+------+------+------+
1 |      |      |      |      |      | 		|  	   |
  +------+------+------+------+------+------+------+
2 |      |      |      |      |      | 	    | 	   |
  +------+------+------+------+------+------+------+
3 |      |      |      |      |      | 		| 	   |
  +------+------+------+------+------+------+------+
```

显然，因为保存了操作数，而且要把所有执行单元的结果和所有已保存的的地址做比较，所以硬件开销很大，无法做得很深。

为了记录每个 register 最终由那条指令写入，每个 register 也需要增加一个字段来记录，

| 符号 | 含义 |
| ---- | ---- |
| Qᵢ | 写入寄存器的 RS 标号（0/blank 表示没有值要写入寄存器） |

```
#!text
// Register File 的数据结构

   +------+-------+
   |  Qᵢ  | value |
   +------+-------+
r0 |      |       |
   +------+-------+
r1 |      |       |
   +------+-------+
r2 |      |       |
   +------+-------+
r3 |      |       |
   +------+-------+
```

### Stage 1: issue

**发射阶段**

```
#!text
Fi[FU] <- dst;
```
### Stage 2: execute

**读操作数**

```
#!text
```

**执行阶段**

```
#!text
```

### Stage 3: write back

**写回阶段**

```
#!text
```

## Improvements

## Ref

[Tomasolu wiki](https://en.wikipedia.org/wiki/Tomasolu_algorithm)

[Revervation Station wiki](https://en.wikipedia.org/wiki/Reservation_station)

[Computer Architecture - Chapter 3.4](https://book.douban.com/subject/6795919/)

[ECE252 duke]()