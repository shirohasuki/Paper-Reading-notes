
## Overview of 《A Survey of Quantization Methods for Efficient Neural Network Inference》

Sehoon Kim佬的又一篇大作，膜~

### 量化的基本概念

#### 均匀量化uniform quantization & 非均匀量化non-uniform quantization

![量化综述1.png (464×382) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Quantization Survey/img/量化综述1.png)

非均匀量化使我们能够通过非均匀地分配比特和离散化参数范围来更好地捕获信号信息，但通常难以在通用计算硬件（例如 GPU 和 CPU）上有效部署。

**目前，硬件映射基本都用均匀量化**

#### 对称量化symmetric quantization & 非对称量化asymmetric quantization

![量化综述2.png (464×382) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Quantization Survey/img/量化综述2.png)



量化因子变为：
$$
S = \frac{β−α}{2^b−1}
$$



#### 静态量化static quantization & 动态量化dynamic quantization

动态量化在每一个激活值中动态计算clipping range，因此代价较大

静态量化提前输入不同的校正量来计算clipping range，inference的时候保持不变

​	



#### 不同粒度的量化

Layerwise Quantization：实现简单但是因为每层的量化比例固定，所以同一层中不同参数范围不同，使用同一个缩放因子，会损失量化分辨率。

Groupwise Quantization：将一些通道的处理分在一层，作为整体进行量化，不同缩放因子会带来额外成本。

Channelwise Quantization：给每个通道分配一个缩放因子

Sub-channelwise Quantization：？


![量化综述3.png (464×382) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/Quantization Survey/img/量化综述3.png)

**Channelwise Quantization是目前用于量化卷积核的标准方法**



#### Fine-tuning Methods 微调方法

##### Quantization-Aware Training（QAT）

前递和反向传播都使用浮点数来做，模型的参数在每次梯度更新之后被量化。

如何处理前传量化时不可微分的问题：使用STE（Straight Through Estimator），使用与量化函数相似的连续函数来代替参数更新。文中也介绍了些Non-STE的方法。

QAT缺点：重新训练神经网络模型的计算成本高

##### Post-Training Quantizaiton(PTQ)

无需重新训练模型，相比QAT时间开销会小很多（问题：根据数据集和模型进行量化？）

缺点：准确率会比QAT低，尤其是低精度的量化。文中有介绍减少损失的方法

##### Zero-shot Quantization（ZSQ）

QAT和PTQ为了在量化后实现最小的精度下降，都需要访问训练数据的全部或一小部分。

然而有些特殊场景无法做到，如：训练数据集要么太大而无法分发，出于安全或隐私问题。所以提出ZSQ方法不依赖dataset样本进行量化。

ZSQ + PTQ

*问题：没太看懂，不要数据再量化的？直接摁调？*

ZSQ + QAT

使用GAN来学习训练数据，并且生成数据来进行QAT。缺点在于GAN不能学习到内在的统计信息（量化因子不准？）

##### Stochastic Quantization

支持观点：微小的weight变化可能并不会导致结果的变化，因此可以增加一些随机因子，使其escape出来（なに？）

介绍了两种随机量化的方法

- 传统方子：

$$
Int(x) =\begin{cases}
 [x] & \text{with probability  } [x]-x \\
 [x] & \text{with probability  } x-[x]
\end{cases}
$$


-  QuantNoise：在每次正向传递过程中量化权重的不同随机子集，并用无偏梯度训练模型。在许多计算机视觉和自然语言处理模型中，这允许较低位精度的量化，而没有显著的精度下降。缺点是每次权重更新都要生成随机数，会带来较大的开销。



### 更Advanced的量化

#### 伪量化Simulated  Quantization & 全整型量化Integer-only Quantization

Simulated Quantization：先用量化值生成浮点值，然后做计算，最后再量化回来。适合bandwidth-bound计算

Integer-only Quantization：全程使用整数运算，速度更快。适合computer-bound计算。

Dyadic quantization：使用二进制量化来进行模拟。适合computer-bound计算。（二元数是分子为整数、分母为 2 的幂的有理数。这使得计算图只需要整数加法、乘法和位移，而不需要整数除法）
#### Mixed-Precision Quantization

motivation：面对极端的低bit压缩uniform quantization会导致性能的衰减。

该方法中不同的网络层会使用不同的精度进行量化。

对网络层选择量化的精度的搜索是指数级别的，因此大量工作着手于如何对其进行搜索：

- 基于强化学习（RL）的方法或基于神经架构搜索（NAS）的方法
- 使用阶段性的正则化函数来直接判断某一层应该使用的精度
- HAWQ 引入了一种基于模型二阶灵敏度自动查找混合精度设置的方法。理论上表明，二阶算子Hessian的迹可用于测量层对量化的敏感度。在 HAWQv3 中，引入了仅整数、硬件感知的量化。速度比RL快100x。（听起来很nb）

#### Hardware Aware Quantization

motivation：量化加速依赖于硬件有哪些资源

使用RL算法来决定硬件感知的混合精度设置，利用软件模拟硬件的延迟信息（*太玄幻了，万物皆可RL是吧*）。

之后还有直接部署在硬件上的操作。

#### Distillation-Assisted Quantization

这个不大懂，后面看了蒸馏再说。

#### Extreme Quantization极端量化

也就是超低精度量化

二值化是最极端的量化方法。

一些work：

- Binarized NN：二值网络-1与+1，同时二值化偏置和权重，这样可以将耗时的浮点矩阵乘操作变为位操作
- BWN：使用scaling factor来代替-1与+1
- Ternary-Binary Network (TBN)：三值网络，-1,0,1

由于极端的二值化会导致准确率衰减，提出了大量减少衰减的解决方法（略）。

#### Vector Quantization

- 经典算法是使用聚类将权重分为不同的组，然后使用组的质心(centroil)向量作为量化的值来进行推断。使用KNN可以加速8x
- Product量化是先将权重矩阵分成若干子矩阵，然后对每个子矩阵进行向量量化。更加细粒度的聚类也可以很好的保留信息


### 未来量化研究方向

#### Quantization Software

INT8的量化效果不错，但是INT8以下量化的软件实现暂时还比较稀缺

#### Hardware and NN Architecture Co-Design

- 有研究证明修改NN架构的宽度可以减小量化前后的性能差异
- 或者修改单据卷积核或者卷积核的深度但是不影响性能
- One line of future work is to adapt jointly other architecture parameters
- Another line of future work is to extend this co-design to hardware architecture

#### Coupled Compression Methods

- 与剪枝，知识蒸馏或者硬件设计结合（这个方面目前研究还很少）

#### Quantized Training

量化低精度加速神经网络训练
