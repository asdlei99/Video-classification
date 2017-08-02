# Temporal Segment Networks Brief

## Algorithm abstract
* 介绍TSN框架和架构，讨论了Segment based sampling的重要性
* 在TSN框架内尝试并分析了多种aggregating函数
* 调研了TSN在实用过程中遇到的一些特定问题

## Segment based sampling
* 将整个视频划分为多个snippets组成的序列
* 每个snippet是从其对应的segment上随机采样得到的，
* 每个snippet会推理得到自己的snippet-level上的classification结果，然后用consensus function将这些预测结果aggregate成video-level scores
* Video-level的score整合了整个视频上的long-range信息，所以更加可靠
* 训练的优化目标定义在video-level上，迭代更新模型参数来优化

## 建模
F是ConvNet
G是consensus 函数，将snippet-level的预测融合为video-level
G的形式非常重要，需要具有高建模能力，且要可求导
前者保证能够从snippet-level预测中提取video-level预测
后者保证能通过Bp算法优化
H是最终的预测函数，本文采用了softmax

## Aggregation function
本节对结果融合函数的设计进行讨论；并分析了不同类型的融合函数，给出了一些建模的经验。
提出了5种聚合函数：max-pool，ave-pool，topK-pool，加权平均，attention weighting
* Max pooling
TODO 
* Average pooling
TODO 
* top-K pooling
TODO 
* attention weighting
学习一个聚合函数，根据视频内容自动分配每个snippet的权重

## TSN架构
研究了一系列TSN的实际问题，包括框架，输入，训练等

### 架构
研究了inception-V2和V3，ResNet等架构

### 输入
在速度和精度两方面扩展Two-Stream方法，研究了warp光流和RGB帧差
* Warped光流 – 能对抗摄像头运动，更加关注人的运动
* RGB帧差 – 减少了optical flow 计算时间

### 训练
设计了多种策略，为的是对抗数据集不足情况下的over-fitting
* Cross modality initialization - 将ImageNet预训练的网络迁移过来训练光流
* Regularization – 引入partial BN 和 额外的dropout layer，都为降低over-fitting
* Data augmentation – 除了random cropping和horizontal flipping，还引入了corner cropping 和 scale jittering

## Action Recognition in Untrimmed Video

### Challenges
* 事件定位
* 持续时间
* Irrelevant content

### 将基于检测的方法用于动作模型
* 先再整个视频上均匀采样snippets，以覆盖视频的所有位置
* 为了覆盖不同持续时间的视频，用一系列不同size的滑动时间窗口作用在frame scores上，取窗口中得分最高的类别
* 为了降低irrelevant content，相同长度的windows进行topk-pooling，不同长度的窗口之间的结果进行投票，得到最终整个视频的预测结果
