> T5 模型，Google，2020

1. 引入一个统一的框架，探索 NLP 中的 迁移学习
2. 将所有基于文本的语言问题转换成 文本到文本（Text to text）的格式
3. 在多个任务中取得了 SOTA 结果

## Introduction（略）

## 方法

基本思想是，把每个文本处理问题都看成是 “text to text” 问题，从而可以将相同的模型、目标函数、训练过程、解码过程应用到所有的任务中。

![](image/Pasted%20image%2020230507103214.png)
本文的重点不是提出新的方法，而是提出统一框架。
