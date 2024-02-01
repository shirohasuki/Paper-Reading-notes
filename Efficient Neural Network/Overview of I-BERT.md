## Overview of 《I-BERT: Integer-only BERT Quantization》

### 摘要

问题：现有Transformer based models如BERT and RoBERTa存在内存占用多、推理延迟高和功耗高的问题。之前的量化方法使用浮点的方式，不适合在很多硬件上运行。

贡献：提出了一个I-BERT量化方案（INT8），用纯整数运算对整个推理进行量化

实验结果：比FP32快个2.4-4倍

### Introduction

详细的贡献：

- 提出一个新的kernel用于计算GELU和Softmax，使用轻量级二阶多项式来近似 GELU 和 Softmax
- 使用平方根整数计算算法计算LayerNorm

### Related Work
##### Efficient Neural Network的几种方式
- pruning
- knowledge distillation 
- efficient neural architecture design 
- hardware-aware NN co-design 
- quantization（本文中只涉及量化）



##### DNN Accelerators Generators的Related Work

##### ![related work.png (1719×406) (raw.githubusercontent.com)](https://raw.githubusercontent.com/shirohasuki/Paper-Reading-notes/main/ML Accelerators/img/related work.png)



### Methodology

##### Architectural Template

![image-20240122223039547](C:\Users\32027\AppData\Roaming\Typora\typora-user-images\image-20240122223039547.png)

![image-20240122223139703](C:\Users\32027\AppData\Roaming\Typora\typora-user-images\image-20240122223139703.png)

![image-20240122223242027](C:\Users\32027\AppData\Roaming\Typora\typora-user-images\image-20240122223242027.png)

##### Programming Support



##### System Support



#####代码解读！