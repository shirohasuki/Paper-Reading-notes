
## Overview of 《A Survey of Quantization Methods for Efficient Neural Network Inference》

Sehoon Kim佬的又一篇大作，膜~

### 量化的基本概念

#### 均匀量化uniform quantization & 非均匀量化non-uniform quantization

![image-20240217195618430](C:\Users\32027\AppData\Roaming\Typora\typora-user-images\image-20240217195618430.png)

#### 对称量化symmetric quantization & 非对称量化asymmetric quantization

![image-20240217195626907](C:\Users\32027\AppData\Roaming\Typora\typora-user-images\image-20240217195626907.png)

量化因子变为：
$$
S = \frac{β−α}{2^b−1}
$$


#### 静态量化 & 动态量化
