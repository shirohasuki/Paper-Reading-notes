### Overview of 《Runahead Execution: An Alternative to Very Large Instruction Windows for Out-of-order Processors》

[TOC]

#### 摘要

问题：当今的处理器通过OoO来解决长延迟操作（long latency operations）。然而，随着延时的增加，instruction window的大小将以更快的速度增加。

贡献：This paper proposes runahead execution as an effective way to increase memory latency tolerance in an out-of-order processor, without requiring an unreasonably large instruction window. 本文提出的runahead execution可以在乱序处理器中提高内存延迟容忍度，避免使用大instruction window。runahead execution可以解决被长延迟操作阻塞的指令窗，使处理器在程序路径中超前执行。这使得数据在需要时早已被预置到缓存中。

实验结果：增加runahead execution可将 IPC提高 22%。runahead execution+128 条目instruction window，其IPC相比没有runahead execution和 384 条目instruction window仅低 1%。



#### Introduction（第1，3节）

##### 哪些是长延迟操作？

cache miss可以视为一种长延迟操作，其他类型的长延迟操作还包括:

- 浮点运算等复杂运算导致的长延迟

- 同步和互斥操作导致的延迟

- IO操作导致的延迟等

本文中只讨论cache miss的情况，具体的是只讨论L2 Cache的miss情况

##### What’s the architectural state

architectural state 表示架构可见状态,也就是 Instruction Set Architecture (ISA) 定义的处理器状态。它主要包括:

- 通用寄存器的值(如 x86 中的 EAX,EBX 等寄存器的值)
- 状态寄存器的值(如 EFLAGS)
- 内存的状态
- 程序计数器的值

这些状态组成了从 ISA 视角可见的处理器状态。

 microarchitectural state 表示微架构内部的状态,对 ISA 不可见,如重命名表(RAT)、预取队列等。

在乱序处理器中,为了支持 speculative execution,会保持两个状态:

- Architectural state: 架构可见状态,由 Reorder Buffer 维护,用于 retire 指令时更新。
- Speculative state: 猜测执行状态,由各种重命名表如 RAT 维护,用于指令执行。

其中 speculative state 先改变,在 retire 时再更新 architectural state。

Runahead 模式下,进行的是 speculative execution,所以不直接更新 architectural state,称为 “pseudo-retire”。

architectural register and physical register

#####  the instruction window and the scheduling window

指令窗口包含所有已解码但尚未提交到architectural state的指令。它的主要目的是保证指令的乱序执行，以支持精确的异常处理。

调度窗口保存指令窗口中指令的子集。它的主要目的是在每个周期内搜索指令，查找准备执行的指令，并调度执行。

长延时操作会阻塞指令窗口，直到执行完毕。尽管后面的指令可能已经执行完毕，但它们无法退出指令窗口。如果操作的延迟足够长，而指令窗口又不够大，指令就会在指令窗口中堆积，导致指令窗口满载。机器就会停滞，停止前进。

##### 如果有某条指令的执行需要1000个周期？

为了保证效率、使指令能够持续地译码，指令窗的大小也需要为1000。

指令窗指的是，机器中能存储的译码但还没交付的指令数目，它决定了乱序执行对延迟的最大容忍程度。

##### runahead execution方案概述

当指令窗口被长延时操作阻塞时，architectural register file的状态将被checkpoint存下。

处理器随后进入 "“runahead mode"。它会为阻塞操作分配一个假结果，并将其扔出指令窗口。

阻塞操作之后的指令将被获取、执行，并从指令窗口中伪退出（pseudo-retired）。

##### pseudo-retired机制

伪退出，是指指令的执行和完成与传统意义上的指令一样，只是不更新architectural state。阻塞操作完成后，处理器重新进入 "normal mode"。它会恢复checkpoint的状态，并从阻塞操作开始重新读取和执行指令。





#### Related work

Memory access is a very important long-latency operation.

为了解决这个问题相关工作的贡献有：

##### 引入cache
##### Software prefetching techniques

使用编译器静态预测哪些内存引用会导致Cache miss，编译器通过在代码中插入预取指令(prefetch instructions)来实现预取的。

其基本思想是:

1. 编译器通过静态分析,预测哪些内存访问可能产生cache miss。
2. 在这些内存访问指令之前,插入专门的预取指令,提前发出内存读取请求。
3. 当真正的内存访问指令执行时,由于提前预取,数据已经在缓存中,可以快速获取。

举个例子:

```c
// 原代码
for (i=0; i<n; i++) {
  a[i] = 1; 
}

// 插入prefetch指令
for (i=0; i<n; i++) {
  prefetch(&a[i+1]); // 预取下个元素
  a[i] = 1;  
}
```

在这个例子中,编译器预测到数组访问a[i]可能cache miss,所以在其前面插入prefetch指令,提前预取a[i+1],来隐藏miss延迟。

这种方法的优点是可以利用编译器的静态分析优化内存访问。但需要修改代码,增加了指令的数量。且对一些访问模式难以预测。

##### Hardware prefetching techniques

硬件预取的主要思想是通过硬件来预测可能的内存访问,提前发出内存请求,获取数据放入缓存,这样当指令真正需要该数据时就可以快速获取,避免等待内存访问的延迟。

常见的硬件预取技术包括:

- 基于标记的预取:根据历史访问模式,给每个缓存块打标记,根据标记提前预取。
- 基于顺序流的预取:检测到顺序访问模式时,按顺序预取下个缓存块。
- 基于相关性的预取:分析数据地址相关性,根据当前访问预测可能的下个访问地址。
- 组相联预取:同时预取多个缓存组中的多个缓存块。

硬件预取的优点是可以自动发现访问模式，无需修改代码，更容易实现。但预测准确性较差时也可能带来缓存污染,消耗过多带宽。

**与runahead 的区别**：文中提出的runahead 也可以发出预取请求,但它是通过实际执行指令来完成的,预测准确性更高。文章还分析了两者的协同效应。硬件预取可以看作是解决内存延迟问题的另一类技术。

##### Thread-based prefetching techniques

利用多线程处理器上的空闲线程上下文来运行帮助主线程的线程。这些辅助线程执行为主线程预取的代码。该技术的主要缺点是需要空闲的线程上下文和备用资源，而这些资源在处理器被充分利用时是不可用的。

##### Balasubramonian et al.的”future thread“

1. 当一个长延迟指令造成处理器管线阻塞时，将RegisterFile中的一部分动态分配给future thread。
2. 在future thread中执行那些依赖长延迟指令的后续指令。这些指令不能立即改变architectural state，但可以做prefetch等操作。
3. 当长延迟指令完成时,处理器切换回原来的主线程(primary thread)继续正常执行。

这种机制和Runahead都是在长延迟指令出现时执行其后续指令来实现预取。但是该技术也存在以下问题:

1. 资源被分割给两个线程，导致单线程无法全力运行。
2. 需要额外的硬件支持两个线程的context switch。

而超前执行在正常模式和超前模式下都可以充分利用处理器所有资源,没有线程间的资源分割。并且超前指令更有可能在正确程序路径上,因此预取更准确。

##### Lebeck et al.的“分治结构”，scheduling window和WIB
将依赖于长延时操作的指令从（相对较小的）调度窗口中移出，放入（相对较大的）等待指令缓冲区（WIB）。这种机制结合了大指令窗口的延迟容忍能力和小调度窗口的高循环周期时间的优点。

**注：**是WIB，不是Instruction Window






#### Methodology （第4节）

##### 实现Runahead添加的结构：

![img1](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Runahead/img/Runahead.png)

图中虚线表示从缓存流出的miss的情况。阴影结构是支持Runahead所添加的部分。



##### Entering runahead mode

A processor can enter runahead mode at any time. **A data cache miss, an instruction cache miss, and a scheduling window stall** are only a few of many possible events that can trigger a transition into runahead mode.

**触发条件**：在本文中进入Runahead的情况为：当L2 Cache中的内存操作发生miss，且该内存操作到达指令窗口的头部时，处理器就会进入Runahead mode。

**保存”上下文“**：需要保存的”上下文“内容有：

- 导致进入Runahead 模式的指令地址（即，入口地址，此处不是数据的地址，是指令地址）

- checkpoint存入architectural RegisterFiles的值

- 保存branch history register和 the return address stack

  （Branch history register 记录了最近的分支历史,用于动态预测分支的方向。Return address stack 用于预测函数返回的地址。

  保存这两个硬件结构的状态是为了在退出runahead模式并恢复正常模式时,可以快速恢复分支预测的状态,避免分支预测的warm up对性能造成影响。）
  
  综上，即保存入口地址和checkpoint的三个结构。

**设置状态**：指令窗口中的所有指令都被标记为 "runahead operation."



###### what‘s the second level checkpoint mechanism?

<details>
<summary>checkpoint</summary>
在乱序执行的处理器中,为了支持指令乱序执行和分支预测,通常会维护如下checkpoint:</br>
1.Reorder Buffer 中的退休寄存器重命名表(Retirement Register Alias Table, Retirement RAT)。</br>
  记录已提交指令写入架构寄存器的映射,用于在分支误预测或异常时回滚至正确的架构状态。</br>
2.分支历史寄存器(Branch History Register)。</br>
  记录最近的分支历史,用于动态分支预测。</br>
3.返回地址栈(Return Address Stack)。</br>
  用于预测函数返回地址。</br>
在正常执行模式下,上述结构记录了当前处理器的架构状态checkpoint。当发生分支误预测或异常时,可以通过回滚这些表来恢复到正确的程序状态。
</details>

Retirement RAT 在正常模式下指向architectural register状态，但在Runahead模式下指向pseudo-architectural register状态，并反映伪退休指令更新的状态。

在正常模式下,Reorder Buffer上的Retirement RAT记录了架构状态的checkpoint。而在Runahead模式下,Retirement RAT记录了pseudo-retired指令更新的“speculative状态”,不再反映架构状态。

即,现在有两个Checkpoint:

1. Checkpointed architectural register file: 在进入Runahead模式时保存的架构寄存器状态。
2. Retirement RAT: 在Runahead模式下,由pseudo-retired指令更新,反映speculative状态。

所以Retirement RAT的语义在两种模式下是不同的:

- 正常模式:指向committed架构状态
- Runahead模式:指向speculative状态

这种机制的目的是:

1. 保存架构状态以便退出时恢复。
2. Runahead下也要有地方(Retirement RAT)保存pseudo-retired指令的speculative状态。
3. 两级checkpoint可以独立地服务于两种模式。



##### Execution in runahead mode

###### INV位（存在于寄存器、runahead cache和store buffer中）

每个物理寄存器都有一个INV 位。

任何指向INV 位被设置的寄存器的指令都是无效指令。

INV 位用于防止错误的prefetch(bogus prefetches and resolution of branches using bogus data).

**INV 值的传播**:

第一条引入 INV 值的指令是使处理器进入Runahead mode的指令。

如果该指令是一条load指令，则会将其物理目标寄存器标记为 INV。（标记RegFiles中对应字节的INV位）

如果是store指令，则在runahead cache中分配一行，并将其目标字节标记为 INV。（标记runahead cache中对应字节的INV位）

任何写入寄存器的无效指令在调度或执行后都会将寄存器标记为 INV。（寄存器写入无效值标记INV）

任何写入寄存器的有效操作都会重置其目标寄存器的 INV 位。（寄存器写入有效值后清除INV）



###### runahead cache

**读指令触发runahead后两种情况**

- 写后读，写指令和读指令都还在指令窗中（即之前的写指令还没退休）。那么直接从store buffer读出来给读指令就行。**这种情况只修改store buffer中的INV位**
- 写后读，触发runahead的读指令依赖的写指令已经伪退休，则此时数据已经不在store buffer中。**为了解决这种情况引入了runahead cache！** **这种情况修改runahead cache中的INV位**

runahead cache是一个缓存，但它的目的不是缓存数据，而是提供指令之间的数据和INV状态的**通信**（只是用来标记哪些store的data是INV的，这个cache本质用于通信）。在runahead执行期间，只有runahead的store操作才会访问这个缓存。在正常模式下，没有指令会访问它。

被剔除的缓存行也不会存储回其他地方（即没有什么写通/写回的操作）



###### STO位（存在于runahead cache）

指示是否有store操作写入该字节，作用类似于常规cache的valid位，用来区别是写过还是初始化后没写过。



###### INV位和STO位的设置

- 当一个有效的runahead store指令执行完毕时，它会将数据写入其store buffer条目（就像在普通处理器中一样），并重置该条目相关的 INV 位（=0 有效）。与此同时，它还会查询数据缓存，如果数据缓存中的数据丢失，就会向下发送预取请求。

- 当一个无效的runahead store指令被调度时，它会设置其相关存储缓冲区条目的 INV 位（=1 无效）。

- 当有效的runahead store指令退出指令窗口时，会将其结果写入runahead cache，并重置已写入字节的 INV 位（=0 有效）。它还会设置所写入字节的 STO 位（=1 表示该位被写过了，不是初始化的值）。

- 当无效的runahead store指令退出指令窗口时，它会设置 INV 位（=1）和写入字节的 STO 位（=1）（如果其地址有效）

如果这里runahead store操作的地址无效时会怎么样呢？在这种情况下，存储操作会被简单地视为 NOP

  

###### Runahead load操作可能因三种不同原因而无效
1. 它可能源于一个 INV 物理寄存器。（寄存器的INV位）

2. 它可能依赖于store buffer中标记为 INV 的存储。（store buffer的INV位）

3. 它可能依赖于一个已经伪退休的 INV 存储器。（runahead cache的INV位）

   

###### Runahead load operations执行的过程

有效load指令执行时，会并行访问三个结构：data cache、runahead cache和store buffer。

- 如果load指令在store buffer中命中，且命中的条目被标记为有效（INV=0），则load指令从存储缓冲区中获取数据。

- 如果load指令在store buffer中命中，且命中的条目标记为无效（INV=1），则load指令将其物理目标寄存器标记为 INV。

- 只有当load指令访问的runahead cache有效，且load指令访问的runahead cache中任何字节的 STO 位被置位，load指令才被视为在runahead cache中命中。

如果load指令在store buffer中未命中，而在runahead cache中命中，则会检查其在runahead cache中访问的字节的 INV 位，有以下两种情况：

- 如果 INV 位均未被设置，则加载使用runahead cache中的数据执行。

- 如果任何来源数据字节被标记为 INV，则加载会将其目的地标记为 INV。

如果load指令在runahead cache和store buffer中均未命中，但在data cache中命中，则使用：data cache中的值，并被视为有效。（然而，由于以下两个原因，它实际上可能是无效的： 1) 它可能依赖于一个具有 INV 地址的存储空间，或者 2) 它可能依赖于一个 INV 存储空间，该存储空间在runahead cache中将其目标字节标记为 INV，但由于冲突，runahead cache中的相应行已被取消分配。不过，这两种情况都比较少见，对性能影响不大。）

如果load指令在三个结构中都未命中，它就会向L2 Cache发送获取数据的请求。如果该请求在L2 Cache中命中，数据就会从L2 Cache传输到L1 Cache，然后load完成执行。

如果请求未命中L2 Cache，则load指令会将其目标寄存器标记为 INV，并从调度程序中移除，就像导致进入runahead mode的load一样。请求会像错过L2 Cache的普通load请求一样被发送到内存。（*这部分锅又甩回去了？樂*）



###### Runahead下的分支的预测

在Runahead模式下，分支的predicted and resolved与正常模式下完全相同，只有一点不同：与所有分支一样，具有 INV 源的分支也会被预测并speculatively地更新global branch history register，但与其他分支不同的是，它永远不会被resolved。（*PB这块咱实在不熟*




###### Runahead下的伪退休
在Runaheadm模式下，指令按程序顺序离开指令窗口。如果一条指令到达指令窗口的头部，它将被视为伪退休指令。

- 如果被视为伪退休的指令是 INV，则会立即移出窗口。

- 如果是有效指令，则需要等待执行（此时可能变为 INV），并将执行结果写入物理寄存器。

在伪退休时，指令会释放为其执行分配的所有资源。当指令离开指令窗口时，有效指令和无效指令都会更新退休 RAT。Retirement RAT 不需要存储与每个寄存器相关的 INV 位，因为物理寄存器已经具有与之相关的 INV 位。



##### Exiting runahead mode
退出Runahead模式的操作
1. 清空机器状态 - 清空所有指令的缓冲区和已分配资源。
2. 恢复架构寄存器状态 - 将checkpointed architectural register file中的值恢复到物理寄存器中。（对应checkpoint存入architectural RegisterFiles）
3. 修复RAT - 修复Frontend RAT和Retirement RAT,使它们重新指向保存了架构寄存器值的物理寄存器。
4. 恢复分支历史 - 恢复保存的branch history register和return address stack的状态。（对应进入runahead时两者的存入）
5. 失效Runahead Cache - 使所有Runahead Cache的行无效（STO=0，回归初始化状态）。
6. 开始重取指令 - 从导致进入Runahead的指令地址开始,重取和重新执行指令。（对应进入runahead时保存的入口地址）

#### 思考
##### 如果将所有INV位初始置为1，是否可以不用STO位
 <img src="https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Runahead/img/1.jpg" width = "150" height = "350" align=center />
