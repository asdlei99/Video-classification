# Two-Stream 方法简介
* 将视频分解为spatial 和 temporal 分量
  * Spatial 分量对应每帧的外观，承载了视频中的场景和物体目标信息
  * Temporal 分量对应帧间的运动，包含了摄像头和物体的运动信息


# Two-Stream架构
识别分为两个流，每个流用一个ConvNet
用late fusion 融合两个stream的softmax scores，有两种做法
  * Averaging 融合方式
  * SVM融合方式

## Spartial Stream ConvNet
训练方式与ImageNet的分类任务一致

## Optical Flow ConvNet

### 光流的提取方式
* Optical flow stacking – 将L帧连续图像拼成一个2L通道的输入
* Trajectory stacking – 连续L帧的轨迹描述子，堆叠方式与光流一致
* Bi-directional optical flow – 一半前向光流，一半反向光流
* Mean flow substraction

### 光流的ConvNet整体架构
与spatial网络一致

### 训练方式
通过Multi-task Learning融合两个数据集，解决数据量不够问题
用两个softmax classification layer，分别对应UCF-101和HMDB-51训练

## 实现细节

### 网络配置
类似VGG

### 训练
* 沿袭AlexNet
* SGD momentum=0.9，初始learning rate=0.01
* batchsize=256，在所有类别的视频中均匀采样256类的视频，从中随机提取视频帧，组合成256的batch
* Spatial net：仅采样对应的视频帧
* Temporal net：计算对应帧的光流

###测试
以固定帧间间隔采样固定长度的frames 25帧
每帧通过crop和flip提取10个ConvNet的输入
最终整个视频的分类得分是这些frames组合的平均结果
