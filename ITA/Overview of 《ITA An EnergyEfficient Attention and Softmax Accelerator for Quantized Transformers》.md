# Overview of 《ITA: An Energy-Efficient Attention and Softmax Accelerator for Quantized Transformers》

提出了加速器架构--ITA，该架构通过利用 8 位量化和创新的Softmax硬件实现（只对整数值进行操作）

*结构和Softermax系列很像，疑似水论文*

**Softmax硬件实现存在的问题**

Transformer面临的一个特殊挑战是对注意力矩阵的行进行softmax 运算，由于其非线性和非元素性，softmax 运算成为低精度架构的瓶颈。

- softmax 的非线性限制了它在量化值上的执行，而使用浮点运算单元则会产生巨大的面积和功耗成本。

- 此外，softmax 操作的非元素性要求对注意力矩阵的行矢量进行多次传递，从而导致系统内大量的数据移动和功耗。

**Related Work**

- Zafrir等人，Bhandare等人，首先对 softmax 的输入进行去量化，计算 softmax，然后再对输出进行量化。涉及浮点运算单元，对硬件不友好

- I-BERT提出了一种使用二阶多项式逼近 softmax 的方法，完全消除了去量化的需要。不过，该方法的精度较高，为 32 位，而网络其他部分采用的是 8 位量化，因此需要 32 位乘法器和除法器。

- softmax并不是element-wise运算，需要对注意力矩阵的某一行进行最大值搜索和求和。这就需要对该行进行多次搜索，并从内存中进行多次读取，从而导致数据移动和功耗增加。**通常的做法是逐行计算注意力矩阵，并对该行进行累加求和。完成一行后，再进行除法运算以获得概率 **（Wang et al， Keller et al）。然而，这种方法对于权重静态加速器并不可行，因为注意力矩阵不是逐行产生的。本文解决了这个问题。

- sparsity问题 

  - 工作一 OPTIMUS（全整型量化，但没有提供有关其 softmax 实现的实现细节和误差的信息）

  - 工作二 Wang等人（全整型量化，但没有提供有关其 softmax 实现的实现细节和误差的信息）

  - 工作三 SpAtten（伪量化）

  - 工作四 ELSA（伪量化）

存在的缺点：Transformer的sparsity 仅限于注意矩阵，而且取决于网络本身，在这些加速器中支持sparsity 需要付出面积成本。例如 SpAtten 中需要额外的 top-k 引擎，ELSA 中需要哈希和规范计算单元。**因此，ITA 没有利用注意力稀疏性，以实现更高的面积效率** *（确定不是偷懒？）*

- Keller 等人使用 Softermax 算法，该算法使用定点运算，并用 2 取代了基数 e，以简化硬件。在本文中，提出了另一种方法，以最小的硬件面积开销计算整数中的 softmax，而无需用基数 2 逼近 softmax。**ITA不使用基数 2 逼近 softmax**



**计算流程**

1. 初始化

- MAX缓冲器清零
- Σ累计和缓冲器清零

2. 计算第一部分行向量元素

- 找到这部分元素的最大值max1,存入MAX
- 对每个元素x,计算exp(x - max1),累加到Σ中

3. 计算下一部分行向量元素

- 找到这部分元素的最大值max2
- 比较max1和max2
  - 若max2 > max1
    - 将新的max2存入MAX
    - 对之前累加的Σ值,右移(max2 - max1)位,相当于乘以修正因子exp(-(max2 - max1))
    - 对新部分元素x,计算exp(x - max2),累加到Σ
  - 若max2 <= max1
    - MAX不变
    - 对新部分元素x,计算exp(x - max1),累加到Σ

4. 重复步骤3,直到整个行向量计算完毕

- 此时Σ中存的是该行exp(x - maxrow)之和,maxrow是该行的最大值,存在MAX

5. 计算Σ中累加和的倒数,得到归一化因子sigma_inv
6. 对该行每个元素x

- 计算(maxrow - x) >> (8 - log2(8))
  - 相当于(maxrow - x)/256完成近似除法
- 将sigma_inv右移上述值,得到softmax(x)

7. 重置MAX和Σ,进入下一行的计算

   

**ARCHITECTURE**

![ITA2.png (1750×1000) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ITA2.png)



![ITA3.png (643×114) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ITA3.png)



*问题*

- 串行除法器是指一次算不完所有加法？
- Q、K、V不能一把算？减少数据搬运
- 一把右移？(no，节约掉了分子存储的开销)



**改进的Softmax实现**

- Softmax直接对量化值进行计算。

- 为防止出现下溢(underflow)，分子（numerator）和分母（denominator）均以整数值缩放。因此，分母的累加和反转分别以 15 位和 16 位整数格式执行。

  <details><summary>什么是underflow</summary>
  因为计算机没有足够的bit位，所以不能描述一个极小的数，计算机认为这个数是0 。这种误把一个极小的数看作0的现象叫做underflow。
  计算机也不能描述一个极大的数，计算机会认为这不是一个数字。这种误把一个级大的数看作+∞的现象叫做overflow。</details>

- 用最小的内存开销存储最大值和求和值。max值和sum值缓冲区都包含 M 个元素，等于一个tile的行数。

- 不使用任何指数运算模块和乘法器，因为它们在面积和功耗方面都很昂贵。Softmax 是即时计算的，不会增加任何计算延迟。

 - 通过在数据流上计算 softmax，我们避免了多次读取同一个向量，从而减少了数据移动和功耗。



作者观察到当缩放因子超过一定值时，除了输入的最大值外，所有输入的 softmax 都会量化为零。这就意味着，输入的范围可以截取到最终 softmax 大于 0 的输入，而缩放因子可以在训练时进行相应的调整。（前面的部分全部变为0）

![ITA1.png (621×228) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ITA1.png)

可以将系数$log_2e$隐藏在缩放因子$\epsilon$中，并将基数改为 2，以简化硬件。直接把划到$\epsilon$里面去了，真的大丈夫？就这么简单？
$$
e^x = e^{\epsilon x_q} = (2^{log2 e})^{\epsilon x_q} = 2^{((log_2e)\epsilon)x_q} = 2^{\epsilon^{′}{x_q}}
$$
作者计算出最合适的$\epsilon=B/(2^Blog_2e)$，B 是量化表示中使用的比特数，本文中为8。所以

$$
\epsilon^{′}= (log_2e)\epsilon = B/2^B
$$

所以Softmax的计算就变成了这样：

![ITA4.png (518×235) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ITA4.png)

**实验部分**

使用了Compact Transformer输入来和I-BERT比精度
