>译自 [A Recipe for Training Neural Networks (karpathy.github.io)](http://karpathy.github.io/2019/04/25/recipe/)

## 引子 

1. 很多时候，即使模型的参数配置错误或者一些代码编写错误，神经网络也能很正常地训练和推理，但是最后只能在 “沉默中灭亡”
2. 因此我们需要特别关注神经网络的细节，而不是相信一些所谓的即插即用的骗术，作者认为，与成功的深度学习最相关的品质是耐心和对细节的关注

## 训练神经网络的流程

根据上面的说法，作者给出了一些训练神经网络遵循的准则。

其中的哲学思想是：从简单到复杂，在每一个小的步骤中做出假设并确保其准确性，因为大量且复杂的未知 势必会导致错误，而且这需要很长的时间来 debug。

### 数据之天人合一

抛掉一切和神经网络相关的代码，从零开始彻底检查数据：
+ 浏览数据样本
+ 了解数据的分布和模式

具体到细节，可以：
+ 关注数据的不平衡和一些偏差
+ 把自己当作一个分类器来对数据进行分类，这有助于后期模型架构的选择，例如，可以考虑：
	+ 局部特征够用吗， 是否需要全局的上下文特征
	+ 数据之间的差异有多大，是以何种形式出现的
	+ 数据的空间信息是否重要，是否能够 进行 average pooling
	+ 数据细节的重要程度是多大，能否承受降采样
	+ 标签的噪声有多大
	+ 等等

如果神经网络给的预测和实际在样本中观测的不一致（如关注了不应该关注或者不重要的特征），那么这中间很可能存在一些问题。

最后，可以编写简单的代码来计算、查看、搜索、过滤数据集中的一些东西（例如，标签的大小、注释的大小数量等），同时也可以进行一些可视化的操作来发现数据中的异常值。

### 模型之以小见大

建立完整的训评估框架，通过一系列实验验证模型的正确性。例如，可以选择一些简单的小模型进行训练，同时可视化损失、一些其他的度量（如精度等）和 模型的预测输出，这一过程中也可以进行一系列的消融实验。

此过程的一些技巧：
+ 固定随机数种子：确保两次运行代码得到的结果一致
+ 尽量简化：例如，一定不能使用数据增强（后续可能加入，但不是现在）
+ 在画测试集损失曲线的时候，要在整个测试集中进行评估，而不要只绘制当前 batch 的损失然后进行平滑
+ 在初始化时验证损失的值：如果网络层被正确初始化，这时候的损失值应该在一个随机猜测的损失值附近
+ 正确初始化：在对网络的最后一层进行初始化时，如果回归的均值是 50，那么可以初始化偏差为 50，如果数据样本不均衡，那么可以初始化使得其融入这种偏差（例如，如果数据的正负的比例为1:10，那么网络在初始化时使其预测概率为0.1），因为网络在前几次的迭代中，学习的基本只是这些偏差
+ 把人类的测试结果作为 baseline，然后监控训练过程中人类可知的指标（如精度）。或者一个方式是，为每个样本生成两个标签，其中一个标签作为预测，另一个作为 ground truth。
+ 零输入baseline：将输入变成0，这应该比输入样本的效果更差，从而可以判断模型是否学习到从输入中提取信息
+ 过拟合一个batch：对部分少数样本进行过拟合，可以通过增加网络层数和滤波器数量来验证此时所能达到的最低损失（最好是 0），同时在一张图中可视化标签和预测结果，确保其完美对应
+ 由于这个阶段使用的是 toy model，可能会对数据集欠拟合，此时可以增加模型的 capacity，理论上 损失应该减少
+ 在数据输入到模型（通常是代码 y=model(x) 中的 x）之前，对输入的数据进行可视化，这可以发现出一些数据预处理和数据增强方面的问题
+ 可视化预测曲线：在训练过程中，对固定的一部分测试数据持续预测，观察预测结果的动态变化，可以发现一些网络不稳定或者学习率设置不当的问题
+ 反向传播来获得数据传递的关系：深度学习的代码通常是复杂的、矢量化的，且存在大量的广播操作（broadcast），一个非常非常常见的 bug 是，在批处理中混淆了维度（例如，下面代码中错误使用 view 函数和 transpose或者 permute函数会造成难以察觉的问题），且这种情况下网络还是可以继续训练而不会报错，但是最终的结果却是完全错误的！**一个debug的方法是，将损失设为样本  i 的所有输出的和，然后进行反向传播确保能在第 i 个输入中得到非零的梯度**
```python
a=torch.Tensor([[1,2,3],[4,5,6]])
print(a.shape)
print(a[0,:])
print(a.view(3,2)[:,0])
print(a.permute(1,0)[:,0])
# 输出，两种改变维度的方法得到截然不同的结果
torch.Size([2, 3]) 
tensor([1., 2., 3.]) 
tensor([1., 3., 5.]) 
tensor([1., 2., 3.])
```

+ 从 special case 中进行推广：不要尝试从零开始编写那种能够满足所有的功能的代码，相反，可以先从某个具体的功能下手，能够正确实现功能之后再慢慢推广到更多的功能，同时确保现有的功能不变

### 过拟合

在完成上面两个阶段之后，我们对数据集已经有充分的了解，同时也有完整的训练+评估流程，那么下一步就可以动手迭代训练一个好的模型。

两个步骤可以帮助我们找到一个更好的模型：首先得到一个足够大的模型，以至于它可以过拟合数据（这个时候只关注 训练损失），然后适当地进行调整（这个时候需要关注验证损失），
> 这两个步骤基于以下假设，如果我们无法通过任何模型达到一个较低的错误率，那么很有可能出现了一些问题或者错误的配置。

一些技巧包括：
+ 选择架构：切忌使用各种花里胡哨的架构。在项目的早期阶段，只需要找到最简单的论文，然后复现模型以获得良好的性能，在这之后可以做一些修改以进一步提高性能
+ 放心使用 Adam 优化器：在 训练 baseline 的早期阶段，放心使用 Adam，学习率 为 3e-4，因为 Adam 几乎对超参数不怎么敏感。
+ 每次只使一个模块复杂化：每次只增加一个复杂的模块，然后确保它能够提升模型的性能
+ 不要相信默认的 learning rate decay：一定要特别小心 learning rate decay，这不仅是因为对于不同的问题应该使用不同的衰减策略，更因为 scheduler 是和 epoch（或 step）有关的，根据数据集的大小和 batch size 的大小差别会很大！如果不小心这一点，scheduler 可能过早地将学习率降到0从而影响模型的收敛。可以考虑先暂时禁用 这个功能

### 正则化

到这一步，已经有了可以用的模型，有了合适的数据集，此时可以考虑一些正则化的技巧，用训练精度来换验证精度：
+ 获取更多的数据：花大量的时间试图榨干小数据集是非常错误的做法，增加数据是几乎唯一能保证提高神经网络性能的方法，且可以说没有上限
+ 数据增强：考虑 half fake data 甚至 fake data，这一步可以尽情发挥想象进行各种数据增强和扩充，fake data 生成方面，可以考虑：
	+ [domain randomization](https://openai.com/blog/learning-dexterity/)
	+ use of [simulation](http://vladlen.info/publications/playing-data-ground-truth-computer-games/)
	+ clever [hybrids](https://arxiv.org/abs/1708.01642) such as inserting (potentially simulated) data into scenes
	+ 甚至是 GANs
+ 预训练：如果可以的话，即使有充足的数据也可以使用预训练网络
+ 不忘初心，牢记监督学习：不要对无监督的预训练过于兴奋（这一条存疑，毕竟是 19 年的博客，作者无法预知最近两年无监督的发展之快，日新月异）
+ 使用较小的维度：删除一些可能包含虚假线索的特征（很可能导致过拟合），如果 low level 的细节不重要的话，可以将输入特征进行下采样
+ 尝试使用更小的模型：可以通过一些域相关的知识（或者先验知识）来减少模型的某些参数或去掉某些层
+ 减少 batch size：如果使用了 batch norm，较小的 batch size 在某种程度上对应了较强的 regularization
+ 添加 dropout：需要谨慎使用，因为 dropout 在 batch normalization 中似乎不起作用（[参考这篇论文](https://arxiv.org/abs/1801.05134)）
+ weight decay：增加 weight decay 的惩罚（注意不要和 learning rate decay 搞混了）[pytorch学习笔记-weight decay 和 learning rate decay - 简书 (jianshu.com)](https://www.jianshu.com/p/8980c6e576ba)
+ 提前终止：根据验证损失，在模型即将过拟合时停止训练
+ 尝试更大的模型：结合提前终止的大模型最终的性能通常比小的模型好很多

最后，可以可视化模型第一层的权重，来确保能够获得良好的有意义的边缘（对于图像可以，但是其他的输入好像就不太行）。

### 微调

这一小节指导如何在数据集中探索各种配置和模型，一些技巧包括：
+ 随机网格搜索：使用网格搜索看起来很美好，但是很费时。所以可以使用随机搜索，因为神经网络对某些参数的敏感程度远大于其他参数
+ 超参数优化：可以使用贝叶斯工具箱，手动调参也可以只要你有时间，或者丢给师弟师妹调（bushi）

### 完美收官

一旦找到了最佳配置和超参数，仍有一些技巧来提高性能：
+ 模型融合：在任何情况下都可以确保获得 2% （？） 的性能提升
+ 保持定力，持续训练：当验证损失趋于稳定时，可以不用停止训练，说不定训练一个月后模型就变成了 SOTA 呢：）

## 总结

不需要总结，动手去做吧。Talk is cheap, show me the code.