
## Overview of 《I-BERT: Integer-only BERT Quantization》

### 摘要

问题：现有Transformer based models如BERT and RoBERTa存在内存占用多、推理延迟高和功耗高的问题。之前的量化方法使用浮点的方式，不适合在很多硬件上运行。

贡献：提出了一个I-BERT量化方案（INT8），用纯整数运算对整个推理进行量化

实验结果：比FP32快个2.4-4倍

### Introduction

详细的贡献：

- 提出一个新的kernel用于计算GELU和Softmax，使用轻量级二阶多项式来近似 GELU 和 Softmax
- 使用平方根整数计算算法计算LayerNorm
- 使用 INT8 乘法和 INT32 累加来处理矩阵乘法 (MatMul)
- 具体方案
![I-BERT](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient%20Neural%20Network/img/I-BERT1.png)

### Related Work
#### Efficient Neural Network的几种方式
- pruning（剪枝）
- knowledge distillation（蒸馏） 
- efficient neural architecture design 
- hardware-aware NN co-design 
- quantization（本文中只涉及量化）

#### Quantization（parameters and/or activations are represented with low bit precision）

##### 基于CNN的量化

##### Transformer模型的量化工作

- Bhandare等人 Zafrir等人为Transformer-based的模型提出了一种 8 bit量化方案，并将模型大小压缩到原始大小的 25%。

- Shen等人采用统一精度和混合精度（uniform and mixed-precision）量化 BERT 模型。其中混合精度设置使用了二阶灵敏度方法（second-order sensitivity method）。

- Fan等人在每次训练迭代中量化不同的权重子集，使模型更加robust。

##### 更低精度的量化

- Zadeh等人提出了一种无需微调的基于中心点的 3/4 位量化方法。

- Bai等人，Zhang等人利用蒸馏。

- Hinton等人对权重进行三元化/二元化。

- Jin等人将知识蒸馏和学习步长量化方法结合起来，实现了最高 2 位的 BERT 量化。

##### Ours

之前所有基于变换器模型的量化工作都是使用模拟量化（simulated quantization），即全部或部分运算都使用浮点运算。这就需要将量化参数和/或激活状态解量化回 FP32，以便进行浮点运算。

Shen 等人，Zadeh 等人使用浮点运算执行整个推理。

Bai等人，Zhang等人，Bhandare等人，Zafrir等人使用用整数算术处理Embedding和MatMul，但他们将其余操作（即GELU、Softmax和LayerNorm）保留在FP32中。

然而，Ours方法 I-BERT 在整个推理过程中只使用整数量化，即在整个推理过程中不使用任何浮点运算，也不进行任何去量化。

##### 非Transformer的纯整型量化

- Jacob 等人通过用整数运算取代所有浮点运算（如卷积、MatMul 和 ReLU），为流行的 CNN 模型引入了纯整数量化方案。

- Yao 等人的最新研究也将这种方法扩展到了低精度和混合精度的二元量化，这是纯整数量化的扩展，其中不使用整数除法。

然而，这两项研究都仅限于只包含线性和片断线性算子的 CNN 模型，它们无法应用于基于 Transformer 的非线性算子模型，例如 GELU、Softmax 和 LayerNorm。

### Methodology

#### Basic Quantization Method 基础知识补课

##### 均匀对称量化（uniform symmetric quantization）（量化值quantized values与其对应的实值real values映射之间的间距是恒定的）：

$$
r = Sq
$$

$$
q = Q(x, b, S) = Int(\frac{clip(x, -\alpha,\alpha)}{S}) \\ q ∈ [−2^{b−1}, 2^{b−1}-1]
$$

其中，$Q$是量化运算，$x$ 被映射到一个整数 $q$，$b$ 指定了量化的位精度，$Int$ 表示整数映射（四舍五入到最接近的整数）。$clip$为截断函数，$\alpha$ 为$clip$函数的参数用于控制离群值。$S$是量化缩放因子 $S = \alpha/(2^{b−1}−1)$

反量化（dequantization）将一个量化后的值 $q$ 变成实数
$$
\tilde x = DQ(q, S) = Sq ≈ x
$$

##### 非均匀量化（non-uniform quantization）（间距不恒定）

非均匀量化方法可能比均匀量化方法更好地捕捉参数/激活的分布，但它们一般很难在硬件上部署（因为它们通常需要一个查找表，从而导致开销）。因此，在本文这项工作中只关注均匀量化。

##### 非均匀量化（asymmetric quantization）（映射的范围不对称）

Why not？文中没说

本文中使用静态量化的均匀对称量化，即所有的缩放因子$S$在推理过程中都是固定的，以避免计算这些因子的运行时开销。

#### 非线性函数的几种全整型算法

对于线性运算（如 MatMul）或分段线性运算（如 ReLU）量化很简单
$$
MatMul(Sq) = S · MatMul(q)
$$
但是对于非线性运算
$$
MatMul(Sq) \neq S · MatMul(q)
$$
所以，对非线性函数的量化方法通常有几种：

- 其一，是查找表法。缺点在于当这种方法部署在片上内存有限的芯片上时，可能会产生开销，并且会产生与执行查找表的速度成正比的瓶颈。

<details>
  <summary>对应论文</summary> 
  Lai, L., Suda, N., and Chandra, V. CMSIS-NN: Efficient neural network kernels for arm cortex-m cpus.
</details>

- 其二，是将激活去量化并转换为浮点数，然后用单精度逻辑来计算这些非线性运算（Bhandare 等人，Zafrir 等人）。然而，这种方法并非整数运算，不能用于不支持浮点运算的专用高效硬件。（与本文目的不符）

- 其三，是用多项式近似的方法。文中使用这种方法。过程如下

  ![ibert推导1.png (652×346) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient%20Neural%20Network/img/ibert推导1.png)

  演示使用整型计算二次多项式$a(x + b)^2 + c$。

  ![ibert手推1.png (1349×1000) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient%20Neural%20Network/img/ibert手推1.png)

#### 非线性函数多项式近似方法

本文使用的方法为使用插值多项式（interpolating polynomials）

给定一组 n + 1 个不同数据点的函数值 {(x0, f0), . , (xn, fn)}，试图找到一个至多 n 阶的多项式与这些点的函数值完全匹配。众所周知，存在一个阶数最多为 n 的唯一多项式，它通过所有数据点。

多项式形式为：
$$
L(x)=\sum^n_{i=0}f_il_i(x) \quad where\quad  l_i(x) = \prod_{0\leq j\leq n \ \ j\neq i} \frac{x-x_j}{x_i-x_j}
$$
有两个方式调节拟合的效果

- 寻找合适的插值点
- 调节多项式的阶数

存在两个问题

- 高阶多项式意味着更高的计算和存储开销。-> 需要找到一个好的低阶多项式
- 使用低精度的仅整数算法，相乘时可能会发生溢出。-> 需要使用双精度，以避免溢出

#### GELU

GELU的表达式：
$$
GELU(x):=x·\frac{1}{2}[1+erf(\frac{x}{\sqrt{2}})] \quad where\quad erf(x) := \frac{2}{\sqrt{\pi}} \int_{0}^{x} exp(−t^2)dt.
$$
*(这是人想出来的函数？通信原理死去的记忆开始攻击我)*

- 近似方法一：使用Sigmoid函数近似GELU
  $$
  GELU(x)≈xσ(1.702x)
  $$
  然而，sigmoid 函数本身也是一个需要浮点运算的非线性函数。一种方式就是用所谓的 hard-sigmoid(h-sigmoid) 来近似 sigmoid，来获取对 GELU函数的仅整数近似：

  
  
- 近似方法二：使用h-Sigmoid函数代替Sigmoid函数近似GELU 
$$
h-GELU(x):=x\frac 
{ReLU6(1.702x+3)}{6}≈GELU(x)
$$

  *ReLU6是啥？*

  缺点：在 Transformer 中用 h-GELU 来替代 GELU 会导致结果有很大的精度丢失

- 近似方法三（Ours）：使用i-GELU函数近似GELU（使用$L(x)= a(x + b)^2 + c$去近似$erf$函数，已知$ a(x + b)^2 + c$可以整型计算。

  则可以得到如下“损失函数”：
  $$
  \min_{a,b,c} \frac{1}{2}∥GELU(x)−x·\frac{1}{2}[1+L(\frac{x}{\sqrt{2}})]∥_2^2,\\s.t. \quad L(x) = a(x + b)^2 + c
  $$
  即在满足L(x)为二次多项式的约束下,调整a,b,c的值,使损失函数最小化,也就是L(x)尽可能逼近erf函数。损失函数为L(x)和GELU(x)之间差值的L2范数的平方（最小二乘法拟合）。

  **存在问题**：L(x) 是一个用于近似 erf 函数的二次多项式。直接优化上式会导致一个很差的近似，因为 erf 的定义域包含了所有的实数。

  **解决方案**：只优化 L(x) 使函数在一个限定的范围内，因为 erf 函数的 x 很大时，值接近1（-1）。利用 erf 是奇函数的性质，也就是只需要考虑在正数域来近似它然后对称出另一半即可。当发现最佳的插值点后，对多项式 L(x)进行一些调整：
  $$
  L(x)=sgn(x)[a(clip(\left|x\right|,max=-b)+b)^2+1]
  $$
  最终的i-GELU函数如下：
  $$
  i-GELU(x):=x⋅ \frac{1}{2}[1+L(\frac{x}{\sqrt{2}})]
  $$
  ![ibert推导2.png (658×505) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient%20Neural%20Network/img/ibert推导2.png)

*S/根号二怎么不加取整？*

![ibert手推2.png (1000×1124) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient%20Neural%20Network/img/ibert手推2.png)

#### Softmax

Softmax 函数规范化一个输入向量，然后转换成一个概率分布：
$$
Softmax (x)_i := \frac{exp(x_i)}{\sum^k_{j=1}\exp(x_j)}, x=[x_1,...,x_k]
$$
![softmax.jpg (994×785) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/softmax.jpg)

先前的研究要么浮点运算来处理这一层，要么查表法，要么使用的多项式阶数很高，而且只适用于有限的定义域。

首先每个$x_i$减去$x_{max}$，缩小所有指数值大小，$\tilde{x_j}=x_j-x_{max}$都变为非正数
$$
Softmax (x)_i = \frac{exp(x_i-x_{max})}{\sum^k_{j=1}\exp(x_j-x_{max})}, x=[x_1,...,x_k]
$$
已知可以将所有非正数表示为
$$
\tilde{x_j}=(-ln2)z+p
$$
则每个新的$\tilde{x_j}$可以表示为
$$
exp(\tilde{x_j})=2^{−z}exp(p)=exp(p)>>z,
$$
这个表达式就很硬件友好了，之后的工作就是去近似$exp(p)$，论文中结果如下：
$$
L(p) = 0.3585(p + 1.353)^2 + 0.344 ≈ exp(p)
$$
则
$$
i-exp(\tilde{x}) := L(p)>>z
$$
*So easy啊，朴实无华*

![ibert推导3.png (512×438) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ibert推导3.png)

![ibert手推3.png (1308×1000) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ibert手推3.png)

#### LayerNorm

BN是对batch的维度去做归一化，也就是针对不同样本的同一特征做操作。

LN是对hidden的维度去做归一化，也就是针对单个样本的不同特征做操作。因此LN可以不受样本数的限制。

BN就是在每个维度上统计所有样本的值，计算均值和方差；

LN就是在每个样本上统计所有维度的值，计算均值和方差（指的简单的MLP情况，输入特征是（bsz，hidden_dim））。

所以BN在每个维度上分布是稳定的，LN是每个样本的分布是稳定的。

![BNvsLN.png (850×508) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/BNvsLN.png)

LayerNorm:
$$
\tilde{x}= \frac{x−μ}{σ}\quad where \quad \mu =\frac{1}{C}\sum_{i=1}^{C}x_i\quad and \quad \sigma =\sqrt{\frac{1}{C}\sum_{i=1}^{C} (x_i − μ)^2}
$$
这里$\mu$和$\sigma$分别为输入的跨通道维度的平均值和标准差

计算$\mu$很简单

计算$\sigma$的平方根函数这里使用基于牛顿迭代法搜索$[\sqrt{n}]$的准确值，全整数运算。该算法在计算上是轻量级的，对于任何INT32输入，它最多可以在4次迭代中收敛，并且每次迭代只包含一个整数除法、一个整数加法和一个位移操作。*（but整数除法也很不友好啊，假设两个INT32，一轮跑32个周期）*

LayerNorm 中的其他非线性运算，如除法和平方运算，则直接用整数算法计算。

![ibert推导4.png (528×290) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ibert推导4.png)

![ibert手推4.png (1447×1000) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Efficient Neural Network/img/ibert手推4.png)



*后面的实验就略了~*
