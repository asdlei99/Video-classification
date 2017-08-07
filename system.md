
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
特征提取细节
CNN训练细节
融合细节



[50] Temporal Segment Networks for Action Recognition in Videos






