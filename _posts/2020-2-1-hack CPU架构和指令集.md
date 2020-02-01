---
title: 实现hack：计算机架构、内存、CPU和指令集
tags: 计算机基础
---

经过前两篇我们已经实现了所有的组合逻辑芯片和时序芯片，具备了实现hack计算机的基础，本篇是项目硬件部分的最后一篇，我们将在本篇定义hack能够执行的操作，即hack的指令集；实现hack的内存、CPU，然后将它们组合起来形成完整的计算机硬件架构。

### 计算机架构

hack是基于经典的冯·诺伊曼体系架构，它的关键组成部分是将中央处理单元和存储器，通过存储器中存储的程序指令控制中央处理器的执行，即存储程序计算机。它主要有以下几个特点：
* 以运算单元为中心
* 采用存储程序原理
* 存储器是按地址访问、线性编址的空间
* 控制流由指令流产生
* 指令由操作码和地址码组成
* 数据以二进制编码

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbgwoausqdj31b00p20t8.jpg" width="800"></center>

### 指令集（机器语言）
我们可以从两个角度来理解CPU，一个是从机器语言的角度，一个是从物理实现的角度。**机器语言是CPU的功能抽象，物理实现是CPU的具体实现，同时机器语言还是计算机系统硬件层和软件层的接口。** 

想要实现CPU，我们首先要定义CPU能执行哪些操作，然后把这些操作抽象为CPU的指令集。CPU的指令集就是一系列的操作约定，它们以格式化的指令，描述了如何用CPU来操作内存。我们已x86的指令集为例，x86的指令集非常复杂，有几百条指令，包括数据传输指令、算数运算指令、逻辑运算指令、位移指令、字符串操作指令、处理器控制指令、控制转移指令等

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh0f43rzqj312h0u0wk4.jpg" width="800"></center>

我们会抛弃诸多复杂的指令，只实现一个CPU所需的最必须的功能，以方便实现，但更重要的是方便我们理解计算机系统硬件设计运行的最基本原理。我们可以梳理出最必须的4点功能：
* 从内存中读取某地址的数据（包括指令数据），传送到CPU
* CPU进行基本的算数&逻辑运算
* CPU把数据写入内存或寄存器
* CPU跳转到某地址执行

在实现时，我们只需设计两条指令就可以满足上面的4点功能，你没听错，只需要两条。再介绍这两条指令之前，我们先介绍下CPU中需要用到的2个寄存器，A寄存器：用于暂存将要用到的内存地址的值；D寄存器：用于暂存计算过程中的中间数据。回顾一下前两篇的内容，我们实现的芯片的最大字长都16位，所以我们的指令也是16位的。

#### A指令
A指令，即Address指令。它的功能很简单，就是把一个数值传送到CPU中的A寄存器即可，而这个数字后续会当做地址使用，用于定位内存中的存储单元。我们来看一下A指令的格式，A指令把16位数据分成两部分：第一位是标志位，为0时就代表该指令是A指令，剩下的15位就是需要传入A寄存器的数值。可以看到，A指令实现了上面4点功能的第一点。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh0rj8yj6j30we0480sn.jpg" width="600"></center>

#### C指令
C指令较为复杂，分成4部分。A指令实现了上面4点功能的后三点
* 第一位是标志位：为1时就代表该指令是C指令
* 3-10位是comp位：：这几位的排列用于表示执行不同的元素&逻辑运算
* 11——13位是dest位：指示需要把计算出的数据传送到哪里（寄存器或内存）
* 14-16位是jump位：指示CPU执行跳转指令的条件

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh0v8yf1cj311604st8o.jpg" width="600"></center>

comp位，copm位主要作用于CPU中的ALU，指示ALU来进行一些算数运算和逻辑运算，操作数就是A寄存器，D寄存器，或者由A寄存器中的地址指向的内存单元中的数据。操作列表如下：

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh153x9nkj30w20rawf4.jpg" width="600"></center>

通过comp位的指示ALU计算出的数值，再传送到由dest位指示的存储单元，这个存储单元是A寄存器，D寄存器，或者由A寄存器中的地址指向的内存单元，或者是他们的组合。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh18hc1zaj31040d2ab1.jpg" width="600"></center>

大多数情况下，CPU是顺序执行的，但也需要进行跳转执行。jump的执行逻辑是，根据comp位输出的数据，用jump位判断是否需要跳转，如果需要，则跳转到A寄存器中地址的指令，否则按顺序执行下一条指令。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh1e9lupsj30vg0e8mxu.jpg" width="600"></center>

#### 一些符号
同时，我们提前介绍一些预定义的符号，符号的作用是便于我们后续编写hack的汇编语言。所需的符号类型如下：
* 预定义符号：有汇编语言中特殊含义的符号
* 虚拟寄存器：代表一些虚拟寄存器中的地址
* 预定义指针：指向一些特定的内存
* I/O指针：指向内存中I/O的其实地址
* 标签符号：用于表示指令的位置（方便编写跳转逻辑）
* 变量符号：方便在汇编代码中声明变量

### 内存
我们在时序芯片里已经实现了RAM芯片，所以实现内存的工作已经基本完成。本篇主要是对内存的使用、组合和划分。内存按照功能划分，主要分为数据内存和指令内存两部分，正常时情况数据内存和指令内存是在一块大内存下划分的，是操作系统帮开发者分配管理的，这里我们为了更方便的实现，把hack的内存拆成量块：一块数据内存Memory，一块ROM，ROM专门用于存储程序，在程序运行期间不支持写入操作。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh3erthk6j312e0o2mxq.jpg" width="600"></center>

同时把Memory划分为3部分：RAM（16K），用于存储数据；Screen（8K）内存映射到显示器；Keyboard（16bit）内存映射到键盘。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh2ausrj0j30uo0lu0tf.jpg" width="600"></center>

HDL实现：Memory
```
CHIP Memory {
    IN in[16], load, address[15];
    OUT out[16];

    PARTS:
    DMux4Way(in=true, sel[0..1]=address[13..14], a=a, b=b, c=c, d=d);

    Or(a=a, b=b, out=outa);

    And(a=outa, b=load, out=load0);
    And(a=c, b=load, out=load1);

    RAM16K(in=in, load=load0, address[0..13]=address[0..13], out=R0);
    Screen(in=in, load=load1, address[0..12]=address[0..12], out=R1);
    Keyboard(out=R2);

    Mux4Way16(a=R0, b=R0, c=R1, d=R2, sel[0..1]=address[13..14], out=out);
}
```

### CPU
定义完指令集，我们接着需要做的就是按照指令功能实现CPU。hack的CPU主要由以下四部分组成：
* ALU：主要用于执行算数&逻辑运算
* 寄存器：A寄存器、D寄存器
* PC：程序计数器，保存下一条指令地址
* 控制逻辑：解析指令类型，选择寄存器，更新PC等

CPU的电路示意图如下，完整电路图比较难画，暂时不画。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh319ez5ij31800rct9n.jpg" width="700"></center>

HDL实现：CPU
```
CHIP CPU {

    IN  inM[16],         // M value input  (M = contents of RAM[A])
        instruction[16], // Instruction for execution
        reset;           // Signals whether to re-start the current
                         // program (reset==1) or continue executing
                         // the current program (reset==0).

    OUT outM[16],        // M value output
        writeM,          // Write to M? 
        addressM[15],    // Address in data memory (of M)
        pc[15];          // address of next instruction

    PARTS:
    // 指令类型解析
    And(a=instruction[15], b=true, out=ins15);          // ins15指令标志位
    Not(in=instruction[15], out=ins15Not);       // ins15Not:指令标志位取反

    // A指令和A寄存器
    Mux16(a=instruction, b=ALUOut, sel=instruction[5], out=ARSel);
    Mux16(a=instruction, b=ARSel, sel=ins15, out=ARIn);
    Or(a=ins15Not, b=instruction[5], out=ARLoad);
    ARegister(in=ARIn, load=ARLoad, out=AROut);

    // D寄存器
    And(a=ins15, b=instruction[4], out=DRLoad);
    DRegister(in=ALUOut, load=DRLoad, out=DROut);

    // C指令类型区分：计算A还是计算M
    Mux16(a=AROut, b=inM, sel=instruction[12], out=AMOut);

    // ALU逻辑
    ALU(x=DROut, y=AMOut, zx=instruction[11], nx=instruction[10], zy=instruction[9], ny=instruction[8], f=instruction[7], no=instruction[6], out=ALUOut, zr=ALUzr, ng=ALUng);

    // outM & writeM & addressM
    And16(a=true, b=ALUOut, out=outM);
    And(a=ins15, b=instruction[3], out=writeM);
    And16(a=true, b=AROut, out[0..14]=addressM[0..14]);

    // jump逻辑
    And(a=ALUng, b=instruction[2], out=j1Out);
    And(a=ALUzr, b=instruction[1], out=j2Out);
    Or(a=ALUng, b=ALUzr, out=po);
    Not(in=po, out=ALUpo);
    And(a=ALUpo, b=instruction[0], out=j3Out);
    Or8Way(in[0]=j1Out, in[1]=j2Out, in[2]=j3Out, in[3..7]=false, out=jump);
    And(a=ins15, b=jump, out=jumpOut);

    // PC
    PC(in=AROut, load=jumpOut, inc=true, reset=reset, out[0..14]=pc[0..14]);
}
```

### COMBINATION

最后我们按照整体架构图，把CPU、ROM、RAM连接起来即可。

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh27xo7nej31bg0ok0tw.jpg" width="800"></center>

HDL实现
```
/**
 * The HACK computer, including CPU, ROM and RAM.
 * When reset is 0, the program stored in the computer's ROM executes.
 * When reset is 1, the execution of the program restarts. 
 * Thus, to start a program's execution, reset must be pushed "up" (1)
 * and "down" (0). From this point onward the user is at the mercy of 
 * the software. In particular, depending on the program's code, the 
 * screen may show some output and the user may be able to interact 
 * with the computer via the keyboard.
 */

CHIP Computer {

    IN reset;

    PARTS:
    CPU(inM=memoryOut, instruction=ROMOut, reset=reset, outM=outM, writeM=writeM, addressM=addressM, pc=pc);
    ROM32K(address=pc, out=ROMOut);
    Memory(in=outM, load=writeM, address=addressM, out=memoryOut);
}
```

### 总结

至此，我们就完了hack硬件部分的所有工作。我们基于两个基本们：Nand和DFF，一步步实现了诸多芯片，最终实现了整个硬件架构。hack虽小，五脏俱全，我们只保留了计算机架构中最核心的部分，**减少了我们的工作量，更重要的是方便我们拨开众多复杂的技术细节，清楚的了解到计算机体系最核心的设计思想**。完成了hack的硬件系统和指令集的定义，我们就可以编写一个用0101写的hello world程序了，但这貌似有些痛苦，所以我们会在下一篇定义hack的汇编语言，并编写汇编器。😀

<center><img src="https://tva1.sinaimg.cn/large/006tNbRwly1gbh3i0sxnoj310s0u03zp.jpg" width="700"></center>