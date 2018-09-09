
**为什么应该了解处理器的设计呢？**

* 从智力方面来说，处理器设计是非常有趣而且很重要的
* 理解处理器如何工作能帮助我们理解整个计算机系统如何工作；
* 虽然很少有人设计处理器，但是许多人设计包含处理器的硬件系统；
* 你的工作可能就是处理器设计；

**如何开始？**

* 首先设计一个基于顺序操作，功能正确但是有点不太实用的Y86-64处理器，它的时钟周期必须足够慢，才可以在一个时钟周期里完成所有动作（包括取指，译码，访存，写回，更新PC）

* 以这个顺序设计为基础，进行一系列的的改造，创建一个流水线化的处理器（pipelined processor）.处理器可以同时执行五条指令的不同阶段。为了使处理器保留Y86-64 ISA 的顺序行为，就要求处理很多冒险或冲突情况；

**定义一个指令集体系结构，包括定义各种状态单元，指令集和它们的编码，一组编程规范和异常处理事件**

### 定义Y86-64指令集体系结构

#### RF：程序寄存器

| **%rax**      | **%rsp**      | **%r8**      | **%r12**      |
|---------------| --------------| --------------| --------------|
| **%rcx**      | **%rbp**      | **%r9**      | **%r13**      |
| **%rdx**      | **%rsi**       | **%r10**    | **%r14**      |
| **%rbx**     | **%rdi**       | **%r11**    |

* Y86-64的状态类似于x86-64，有15个程序寄存器（省略了x86-64的%r15寄存器以简化指令的编码）。每个程序寄存器存储一个64位的字。寄存器%rsp被入栈，出栈，调用和返回指令作为栈指针。除此之外，寄存器没有固定的含义或固定值。有3个一位的条件码：ZF，SF，OF，它们保存着最近的算数或者逻辑指令影响的有关信息。程序计数器（PC）存放当前正在执行指令的地址。

* 内存从概念上来说就是一个很大的字节数组，保存着程序和数据。Y86-64程序用虚拟地址来引用内存位置。硬件和操作系统软件联合起来将虚拟地址翻译成实际或者是物理地址，不过为简化起见，我们只认为虚拟内存系统向Y86-64程序提供了一个单一的字节数组映像。

* 程序状态的最后一个部分是状态码stat，它表明程序执行的总体状态。它会指示是正常运行，还是出现了某种异常。例如当一条指令试图去读非法的内存地址时

## Y86-64指令

**Y86-64指令的一些细节**
* x86-64的movq指令分成了四种不同的指令：irmovq，rrmovq，mrmovq，rmmovq。两个内存传送指令中的内存引用方式是简单的基址和偏移量形式，在地址计算中，我们不支持第二变址寄存器（second index register）和任何寄存器值的伸缩（scaling）。同x86-64一样，我们不允许从一个内存地址直接传送到另一个内存地址。另外，也不允许将立即数传送到内存；
* 有4个整数指令（OPq）。他们是addq, subq, andq, xorq。他们只对寄存器进行操作。这些指令会设置3个条件码ZF,SF,OF(零，符号，和溢出)；
* 7个跳转指令（jmp,jle,jl,je,jne,jge,和jg）；
* 6个条件传送指令，（cmovle, comvl, comve, comvne, covge, comvg）；
* call指令将返回地址入栈，然后跳到目的地址。ret指令从这样的调用中返回；
* pushq， popq指令实现了入栈和出栈；
* halt指令停止指令的执行。x86-64有一条与之相当的指令hlt。x86-64应用不允许使用这条指令，因为它会导致整个系统暂停运行。对于Y86-64来说，执行halt指令会导致处理器停止，并将状态码设置为HLT；

| **字节**          | ** 0**  | ** 1**  | ** 2**  | ** 3**  | ** 4**  | ** 5**  | ** 6**  | ** 7**  | ** 8**  | ** 9**  |
|-----------------|--------|---------|---------|--------|---------|---------|--------|---------|---------|---------|
|halt|0 0|
|nop|1 0|
|rrmovq rA, rB|2 0|rA rB|
|irmovq V,   rB|3 0 |F   rB|  |  |  |V|  |  |  |  |
|rmmovq rA, D(rB)|4 0| rA rB||||D|||||
|mrmovq D(rB), rA|5 0|rA rB||||D|||||
|OPq rA, rB|6 fn|rA  rB|
|jXX Dest| 7 fn| ||||Dest||||
|comvXX rA, rB| 2 fn|rA  rB|
|call  Dest|8 0|||||Dest||||
|ret|9 0|
|pushq rA|A 0| rA  F|
|popq rA|B 0| rA  F|

|整数操作指令|     分支指令     |            传送指令         |
|--------------|----------------|------------------------|
|addq  60      |jmp 70 jne 74 |rrmovq 20 cmovne 24|
|subq  61      |jle 71 jge 75    |cmovle 21 cmovge 25|
|andq  62      |jl  72 jg 76      |cmovl 22 cmovg 26    |
|xorq  63      |je 73                |cmove 23                    |

|数字|     寄存器名字     |数字|     寄存器名字     |
|-----|------------------|---- |------------------|
|0|%rax|8|%r8|
|1|%rcx|9|%r9|
|2|%rdx|A|%r10|
|3|%rbx|B|%r11|
|4|%rsp|C|%r12|
|5|%rbp|D|%r13|
|6|%rsi |E|%r14|
|7|%rdi|F|无寄存器|

**Y86-64的异常**
* 当遇到异常的时候，其简单地让处理器停止执行指令。在更完整的设计中，处理器会调用一个**异常处理程序**（exception handler），这个过程被指定用来处理遇到的某种类型的异常。

|Stage|Opq rA, rB|rrmovq rA, rB|irmovq V, rB|rmmovq rA, D(rB)|mrmovq D(rB), rA|pushq rA|popq rA|jXX Dest|call Dest|ret|
|------|-----------|---------------|--------------|--------------------|------------------|-----------|---------|-----------|----------|---|
|Fetch|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br><br>valP&larr;PC+2|icode:ifun&larr;M1[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br><br>valP&larr;PC+2|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br>valC&larr;M<sub>8</sub>[PC+2]<br>valP&larr;PC+10|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br>valC&larr;M<sub>8</sub>[PC+2]<br>valP&larr;PC+10|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br>valC&larr;M<sub>8</sub>[PC+2]<br>valP&larr;PC+10|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br><br>valP&larr;PC+2|icode:ifun&larr;M<sub>1</sub>[PC]<br>rA:rB&larr;M<sub>1</sub>[PC+1]<br><br>valP&larr;PC+2|icode:ifun&larr;M<sub>1</sub>[PC]<br><br>valC&larr;M<sub>8</sub>PC+1<br>valP&larr;PC+9|icode:ifun&larr;M<sub>1</sub>[PC]<br><br>valC&larr;M<sub>8</sub>PC+1<br>valP&larr;PC+9|icode:ifun&larr;M<sub>1</sub>[PC]<br><br><br>valP&larr;PC+9|
|Decode|valA&larr;R[rA]<br>valB&larr;R[rB]|valA&larr;R[rA]<br>||valA&larr;R[rA]<br>valB&larr;R[rB]|<br>valB&larr;R[rB]|valA&larr;R[rA]<br>valB&larr;R[%rsp]|valA&larr;R[%rsp]<br>valB&larr;R[%rsp]||<br>valB&larr;R[%rsp]|valA&larr;R[%rsp]<br>valB&larr;R[%rsp]|
|Excute|valE&larr;valB OP valA<br>Set CC|valE&larr;0+valA|valE&larr;0+valC|valE&larr;valB+valA|valE&larr;valB+valC|valE&larr;valB+(-8)|valE&larr;valB+8|<br>Cnd&larr;Cond(CC,ifun)|valE&larr;valB+(-8)<br>|valE&larr;valB+8<br>|
|Memory||||M<sub>8</sub>[valE]&larr;valA|valE&larr;M<sub>8</sub>[valE]|M<sub>8</sub>[valE]&larr;valA|valE&larr;M<sub>8</sub>[valA]||M<sub>8</sub>[valE]&larr;valP|valM&larr;M<sub>8</sub>[valA]|
|WriteBack|R[rB]&larr;valE|R[rB]&larr;valE|R[rB]&larr;valE||<br>R[rA]&larr;valM|R[%rsp]&larr;valE|R[%rsp]&larr;valE<br>R[rA]&larr;valM||R[%rsp]&larr;valE|R[%rsp]&larr;valE|
|PCUpdate|PC&larr;valP|PC&larr;valP|PC&larr;valP|PC&larr;valP|PC&larr;valP|PC&larr;valP|PC&larr;valP|PC&larr;Cnd?valC:valP|PC&larr;valC|PC&larr;valM|

|stage|jXX Dest|call Dest|ret|
|------|-----------|---------------|--------------|
|Fetch|icode:ifun&larr;M<sub>1</sub>[PC]<br><br>valC&larr;M<sub>8</sub>PC+1<br>valP&larr;PC+9|icode:ifun&larr;M<sub>1</sub>[PC]<br><br>valC&larr;M<sub>8</sub>PC+1<br>valP&larr;PC+9|icode:ifun&larr;M<sub>1</sub>[PC]<br><br><br>valP&larr;PC+9|
|Decode||<br>valB&larr;R[%rsp]|valA&larr;R[%rsp]<br>valB&larr;R[%rsp]|
|Excute|<br>Cnd&larr;Cond(CC, ifun)|valE&larr;valB+(-8)<br>|valE&larr;valB+8<br>|
|Memory||M<sub>8</sub>[valE]&larr;valP|valM&larr;M<sub>8</sub>[valA]|
|Write||R[%rsp]&larr;valE|R[%rsp]&larr;valE|
|Refersh PC|PC&larr;Cnd?valC:valP|PC&larr;valC|PC&larr;valM|

### 优化程序性能

* **消除循环的低效率**<br>**代码移动**这类优化包括识别要执行多次但是计算结果不会改变的计算，因而可以将计算移动到代码前面不会被多次求值的部分；

* **减少过程调用**<br>

* **消除不必要的内存引用**<br>

### 缓存不命中（cache miss）种类

* 冷缓存（cold cache）会形成**强制不命中**（compulsory miss）或冷不命中(cold miss)<br></br>
* **冲突不命中**（conflict miss）<br></br>
* 工作集（working set）超过缓存大小，会经历**容量不命中**（capacity miss)<br></br>

#### 基于缓存的存储器层次结构行之有效，是因为较慢的存储设备比较快的存储设备更便宜，还因为程序倾向于展示**局部性**

* **利用时间局部性**(temporal locality)：由于时间局部性，同一数据对象可能被多次使用。一旦一个数据对象在第一次不命中时被复制到缓存中，我们就会期望后面对该目标的一系列的访问命中。因为缓存比低一级的存储设备更快，对后面的命中的服务会比最开始的不命中快得多。
* **利用空间局部性**(apatial locality)：块通常包含多个数据对象。由于空间局部性，我们会期望后面对该块中其他对象的访问能补偿不命中后复制该块的花费。

|类型|缓存什么|被缓存在何处|延迟(周期数)|由谁管理|
|----|----------|--------------|-------------|---------|
|CPU寄存器|4字节或8字节字|芯片上的CPU寄存器|0|编译器|
|TLB|地址翻译|芯片上的TLB|0|硬件MMU|
|L1高速缓存|64字节块|芯片上的L1高速缓存|4|硬件|
|L2高速缓存|64字节块|芯片上的L2高速缓存|10|硬件|
|L3高速缓存|64字节块|芯片上的L3高速缓存|50|硬件|
|虚拟内存|4KB页|主存|200|硬件+OS|
|缓冲区缓存|部分文件|主存|200|OS|
|磁盘缓存|磁盘扇区|磁盘控制器|100000|控制器固件|
|网络缓存|部分文件|本地磁盘|10000000|NFS客户|
|浏览器缓存|Web页|本地磁盘|10000000|Web浏览器|
|Web缓存|Web页|远程服务器磁盘|1000000000|Web代理服务器|
