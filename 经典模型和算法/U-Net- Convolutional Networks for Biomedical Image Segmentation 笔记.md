> 

1. 提出一种网络和训练策略，采用数据增强更有效地利用可获得的带注释的样本
2. 网络结构包括：用于捕获上下文的收缩路径和用于实现精确定位的对称的扩展路径
3. 网络可以从很小少的图像进行端到端训练，且推理速度很快


## Introduction
1. 在医学分类任务中，需要对每个像素分配一个标签（定位），而且很难获得大量的训练图像
2. 本文构建了一个全卷积网络（Fully convolutional networks for semantic
segmentation），修改了原始论文的结构，使其只要很少的图片就可以获得精确的分割，网络结构图如下：
![](./image/Pasted%20image%2020221025164830.png)
3. 相比于原结构，一个重要的修改在于，在上采样部分也有大量的特征通道，这允许网络将上下文信息传播到更高分辨率的层。扩张路径与收缩路径对称，并形成u形结构。网络没有任何完全连接的层。该策略允许通过重叠平铺策略对任意大的图像进行无缝分割
4. 数据量很少，采用 elastic deformations 进行数据增强，这也可以有效的模拟组织变形。
5. 总之就是效果很好

## 网络结构

网络结构还是如上图所示，左边是收缩路径，右边是扩展路径。

收缩路径包括重复应用两个3x3卷积，每个卷积之后是一个ReLU和 2x2的 max pooling 用于进行下采样，每个下采样步骤中，都将特征通道的数量加倍。

拓展路径中的每一步都包括对特征图进行上采样，然后进行2x2卷积（“上卷积”），该卷积将特征通道的数量减半，再与收缩路径中相应裁剪的特征图进行级联，最后是两个3x3卷积，每个卷积都后跟一个ReLU。在最后一层，使用1x1卷积将每个64分量特征向量映射到期望数量的类。

## 训练
1. 由于卷积没有进行 padding，输出图像比输入图像边界宽度小一个常数。
2. 采用batch为1的单张图片进行训练，即使用随机梯度下降，相应的动量值也较高（0.99）
3. 预先计算每GT分割的权重图，以补偿训练数据集中某类像素的不同频率，并迫使网络学习我们在触摸单元之间引入的小分离边界
4. 从高斯分布初始化卷积层的权重

### 数据增强
1. 对于显微图像，主要需要移位和旋转不变性以及对变形和灰度值变化的鲁棒性。
2. 在粗略的3x3网格上使用随机位移矢量生成平滑变形。位移从具有10像素标准偏差的高斯分布采样。然后使用双三次插值计算每像素位移。收缩路径末端的dropout层执行进一步的隐式数据增强。

## 实验（略）