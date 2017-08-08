
## architecture - ()
设计的理念，特征 - 0.5p

Features


### 系统分为4层 - 1p

整个系统分为4层
 - 基础层是视频缓存层，作为DataLoader的最前端，根据数据请求，从云存储中打开视频，然后读取指定帧，存储在Key-Value中
 - 特征提取层 根据系统需求提取指定特征，特征的提取独立进行
 - CNN训练层 特征提取之后，对应的CNN训练任务会得到信号，进行训练 各个特征的训练资源可以动态分配
 - 结果融合层 通过各种方法对结果进行融合，在验证集上评估得到结果，交付给研究人员做判断
 - 旁路控制系统：控制数据读取，控制消息同步，做系统监控，训练过程监控，结果评估

1 视频缓存层
2 特征提取层
3 CNN训练
4 融合



# 3 System
To design a system that works smoothly with large-scale video training is a non-trivival task. 
First of all, let's have a glapse of the whole video training procedure. We need to know what need to be done. This is necessary when we design a system. Section 3.1 will discuss this. And section 3.2 will discuss challeges we meet in architecture design. Then in section 3.3, we give the whole picture of our system. Then we come into details in section 3.4.

## 3.1 Traditional Video Training System Description
Speak of traditional video training system, we refer to architecture such as two-stream[11] and Temporal Segment Network[50]. These systems are very similar from hierarchy view.
Take TSN as example, it takes video frames and optical flows and other op variants as training input. Training framework is similar to Two-Stream[11], uses a customize CNN. After training, serveral aggregation methods are used to aggregate features, to get accurate result.


## 3.2 Challeges

* Storage
The first challenge we met is the storage problem, I mean, not memory storage problem, but more fundemantally, data storage challege on hard disk. As we known, video data in this task is about 3TB. However, this is just video. When we train on this data, we need to decode them into image frames and feed to training procedure. Then we have two options, one is decode videos and extract features that training process need in advance, and training program just need pick up frames and features from disk. This is what most opensource video training programs do. The advantages is obviously. No need to consider consuming on decoding or extraction, and makes the whole pre-processing more effective, cause we can decode and extract features sequentially, then we can mostly reuse some feature results, such as optical flow stack.
The disadvantage is also deadly, that is the system needs order of magnitudes storage space for frames and features than video storage. In this case, we have 3TB videos, means more than 50TB storage is needed for pre-computed features. This has to be a big cluster, and a long pre-processing phase.
So the opposite way is just having videos on the disk, decode frames and extract features on-the-fly. This is oppsite solution, heavy pressure is on computing, not on storage. However, this is a easy trade-off to make, if latter one fulfills real-time performance, then what we care is which cluster is bigger. In our architecture, we choose later solution according to scalability computing, and we will give out comparison in Section 4.
(TODO - 解决同一个问题的机器数目比较)


* Transmission
When we compute data on distributed cluster, the second problem comes up - Transmission capacity. Single 256 x 256 BGR frame is 256 x 256 x 3 = 192KB, if face with optical flow, it becomes 256 x 256 x 2 x L = 128L KB, here L is optical flow stack width, if L = 10, then it is 1280KB = 1.25MB. If we expect batchsize = 256 for a distributed training system, then the only way is compress transmission data. We encode images, and only transform features need to corresponding training cluster instead of transform whole data package to all training instances. 

* Computing Capacity
Training speed is definitely an eternal topic. In video learning system, we need to train serval CNNs parallelly. For the sake of accomodate different training framework, we use docker container as training instance unit, building distriuted training instances on it.

* Others
There are also more issues besides these fundamental challeges, end-to-end training scopy, hierarchical loose coupling architecture, unifed framework that easy to add/remove/replace algorithm modules. Algorithm researchers brought up many precious requirements, and these requirements promoted our video learning system.


## 3.3 Architecture

We take all these requirements into account, and designed our Hierarchical Elastic Video Learning System (HEV). Fig-2 is the whole arthitecture of the system. 

From left to right, the HEV system is divided into five level, each handles an independent task. The former four is training phase, composed entire training procedure. The last one is particular for inference, it contains action detection to lower prediction noises from irrelevant video segments. A master node is used to coordinate tasks and supervise whole procedure. And a key-value store is used to switch abundant data between different level.


* Pre-Computing Module
In order to make a unified Data Chaching Layer, we made a two-layer chaching module to make video decoding and feature extraction independently.
A Video Decoding worker is used to response decode request only, get required video, offset and video window width, deliver corresponding frames.
And Feature Extraction worker is responsible for extracting whatever features training needs. It takes frames from video decoding worker as input, and instructions from master to identify what to extract, when to extract, when to hold.

* Training Module
Training Module get frames or features from a kv-store that Pre-Computing Module puts data to. As mentioned before, training module usually contains several CNN training, and each one corresponding to different features. So each training submodule just need to catch specified ones, which lowered transmission expenses. Cause different training has different rate of convergence, computing resource between them should be allocated dynamically. For simplarity, We make this allocation on GPU-level, assign different training with different GPU amount. The number is choosed by experience beforehead, and adjusted according to validation loss. We evaluate adjust necessarity after each itration, which avoid most disturbs to trainig.

* Aggregation Module
Aggregation module takes feature-level results from validation set as input, attempts different aggregation strategies and parameters, and involves independent evaluation processes to pick up best one. The target of this module and training module is almost the same - get optimized classification capacity, but emphasis on different aspects. The former module emphasizes on network training, take full advantage of CNN optimization and modeling. However in aggregation part, as CNN have already makes fairly good prediction, training is not necessary, parameter tuning is usally good enough. In the meanwhile, this module still keeps possibility to train a more powerful aggregation model, as new modeling method maybe brought up very soon.


* Inference Modeling
The above four level formed the whole training system. If inference target are all trimmed videos, inference procedure is almost the same as training. But when confront untrimmed videos, which have a large percentage of unmeaning video segments, result would be heaviy confused by irrelevant video segments. Unfortunitely, this is real world. So we involved highlight detection phase in Inference module to improve prediction.

* Master Cluster
A Master Cluster is used to dispatch tasks, supervise independent modules, schedule computing resources between different modules and synchronize task information. Here we used a Cluster for two reasons. First is to manage multiple training procedure in one cluster, which can make provide unified supervision, and provide a easy way to group serveral training tasks as a whole. Second reason is to prevent single node failure disaster, cause Master node is the scheduling core of system, but not computing center of system.


Our Hierarchical Elastic Video Learning System has several contributions to video classification research field. 
First of all, this is a totally industrial solution, modules and components are choosed following industrial standard. Supervision and Error pathes are complete.
Second, system covers every aspect of video training procedure, end-to-end training and hierarchical training can be organized on it conveniently.
Third, hierarchical parts are loose coupling, which means every module can expand or shrink dynamically, which reduced system risk and performance bottleneck. And loose coupling has another important advantage, methods can be attached or detached easily, just maintaining registration table in Master node. And inside individual module, we have many choise to optimize performance, such as distributed training, map-reduced video decoding, and spark-based feature extraction.


## 3.4 Design Details

* Video Decoder
Master Cluster uses random sequence generator to produce random video segment seed. The video learning system rolls data epoch by epoch. So we set sequence generator works once for an epoch, produce video segment seed before epoch begin. 
Following class balance in training[51], we feed video with three random selection, first random choose a class id, then random a video id in choosen class, at last random a frame offset in choosen video.
So the random sequence contains three random elements: video Class ID, video ID in current class, frame begin offset in the video; cause frame stack is always needed in following phase, it comes with another constant parameter, frame count to decode.
Decoding rate and training rate is not matching, one decoding module may deals with multiple training, and followed by arbitrary feature extractors. So we make a independent module for decoding. This module is easy and lightweighted, and elastic to expand or shrink.

The workflow of Video Decorder is (TODO):
a) notify decoder workers are ready
b) receive random sequence
c) decode videos and serialize frames
d) put into key-value store

Video Decoder execute following commands from Master Cluster: start/stop/pause/resume Decoding.


* Feature Extractor
Feature Extractor is an open-registration module. Any appropriate extractor can be attached by registration in Master Cluster. Default feature is single video frame. Which is already extracted by video decorder. Here we take single optical flow as an example. We packed flow extractor into a docker image, the master cluster only need to maintain a feature-image table. The code in docker image are shown in Algorithm 1 (TODO), extract features and put into key-value store. This docker-based microservice framework simplified resource scheduling. To add a new extractor, we build a new image, add a record in the table. To expand specified extractor, we just change corresponding container number, and Load-Balance will take charge of rest.
This scheme reduced our work for attching features, thus we can evaluate all features we are interested in.

	* Optical FLow
(TODO)
	* Frame Difference Stack
(TODO)
	* HOF / HOG
(TODO)
	* MBH
(TODO)
	* Dense Trajectory
(TODO)


* Distributed Trainer
The key factor of distributed trainer is fast. So distributed system is necessary. We also use microservice framework mentioned in extrator module to carry training systems. So Trainer-Registration Table is also needed. There are several training instances, accomodated in a heterogeneous architecture. Training instances can be single-CPU or GPU container, multi-GPU in single node, multi-GPU in distributed configuration. When instantiated, containers should be assigned with specified computing resource. However, distributed can not be assgined directly on docker level, a unified interface is built to accomodate distributed system management.
Mainly used training framework in video learning is two-stream[11] framework. Spatial and temporal cues are trained seperately, final aggeration is carried out in next module, which we will discuss later. Spatial cue used is single frame. We implemented it in PyTorch Framework, followed ImageNet classification scheme, fine-tuned from 22k pretrained model. We trained serveral Versions, including ResNet-50, ResNeXt-50, Inception-V3, DenseNet. We trained models with high variance in structure, which leverages aggregation better.
Temporal cues are represented by optical flow and its variants, and frame difference stacks. The best part of our distributed trainer is we can try as many trainer as we want. 
Each trainer is equiped with a prefetch queue to preload data from memcache cluster, avoid transform bottleneck. If bottleneck still exists, we can expend memcache cluster.


* Aggregator
Aggregation uses lightwighted post-processing methods to find best model combination, good aggregation may promote more than 10 percents on classification accuracy. So we build the same architecture as feature extraction module. Then evaluation is carried out. This module architecture can accomodate as many aggregation experiments as needed.

	* Fisher Vector
	(TODO)
	* Ave pooling
	(TODO)
	* max pooling
	(TODO)
	* top-k pooling
	(TODO)
	* linear weighting
	(TODO)
	* attention weighting
	(TODO)
	* VLAD
	(TODO)
	* compact bilinear pooling
	(TODO)

* Master Design
From the requirement above, Master Cluster take response for following tasks:
a) random sequence generate; b) algorithm image tables maintain; c) use signals to balance different modules; d) online containers management, such as expand or shrink worker containers; e) monitor resources.
	* memcache
	(TODO)
	* Signals
	(TODO)
	* k8s
	(TODO)



[50] Temporal Segment Networks for Action Recognition in Videos
[51] Relay Backpropagation for Effective Learning of Deep Convolutional Neural Networks






