
# Hierarchical Elastic Video Learning System (HEVLS) 系统架构

## 描述
该系统是多层级的弹性视频学习系统。分4个层级分别完成视频学习所需的特征提取、网络训练、特征融合、结果评估阶段。
作为弹性的学习系统，“弹性”包括两个方面：一方面是系统的特征提取、网络训练和特征融合环节之间是一个松散耦合的组织形式，从而可以方便地添加、删除所需要的特征提取方法和训练融合算法；另一方面，弹性还体现在系统的所有算法都部署在分布式的平台上，并可以以微服务的形式进行方便的扩缩容，根据问题的规模动态地调节系统的规模，以及根据不同模块的计算和收敛速度动态地调整计算资源在各个模块之间的分配。这种架构的好处是多方面的，一方面使系统性能和数据容纳能力能够满足超大规模视频算法训练的需求，另一方面也将某一节点瘫痪带来的损失降到最低。最大的好处是，该系统能实现从视频读取到模型结果评估的端到端训练，使研究人员能够将更多精力放到算法本身。

* 视频解码 - Video Decoding Module
视频解码模块，接受来自Master Group的解码命令。视频解码的输入为视频ID，帧偏移，需要的帧数。
Master Group 根据训练系统的输入需要给解码层发送请求。通常在用卷积网络进行单独特征的训练时，视频输入的方式为每次迭代时随机选一个类别，然后从该类别中随机选择一个视频，最后从该视频中随机选择一个位置，在该位置进行视频帧提取和对应的特征提取。因此最直观的形式是，每次迭代的开始，Master Group 会传给视频解码模块三个随机数，和一个所需的解码窗口宽度。Cpp形式的接口函数定义如下：

	'''
	int HEV_DecodeRequest(int classId, int videoId, float frameIdx, int WinWidth);

	'''
由于训练和解码存在速度的不匹配，因为训练可能同时attach不止一个，而且训练需要的特征也可能存在自由发挥的情况。因此我们将视频解码这一过程抽象为独立的处理模块。Master Group预先生成视频采样的随机序列，然后视频解码模块独立运行解码工作，将解码后的图片序列化后存储在kv-store中，等待训练模块和特征提取模块调用。为了降低kv-store爆仓的情形发生，让Master Group以一个iteration为单位分配生成一次本iteration的随机序列。每当kv-store剩余容量达到告警值时，Master Group会给解码模块发送“暂停”指令。

	//Master Group发送给解码模块的命令
	'''
	int HEV_DecodeCommand(int code);

		'''
		code definition:
			1 - start
			2 - pause
			3 - resume
			4 - stop

	'''

	//解码模块的frame stack 序列化
	struct VideoSegmentSerialStructure{
		int videoIdx,
		int classIdx,
		int startFrmIdx,
		int endFrmIdx,
		int fWidth,
		int fHeight,
		int vLength,
		char frames[]
	}


* 特征提取 - Feature Extraction Module
特征提取模块是一个开放协议模块。由于视频处理的特征可能多种多样，存在需要随时扩展的能力。因此HEVLS的特征提取层是独立一层，并没有和视频解码或训练模块绑定。它的任务是从kv-store中获取对应的视频帧，然后根据Master Group给的特征列表处理得到对应的特征，每个特征提取模块自行序列化特征并加入kv-store中。与视频处理模块一样，当kv-store剩余容量低于告警值时，Master Group会给特征提取模块发送“暂停”指令。
所以，当需要扩展新的处理特征时，只需要编写对应的处理模块，将对应的特征处理指令添加到Master Group的列表中；Master Group负责维护序列化后的特征的kv-store。


	//特征提取函数调用接口
	'''
	int HEV_ExtractionRequest(kvPos FrameStack, int ExtractionMethods);
		'''
		ExtractionMethods:
			Optical Flow
			bidirectional Optical Flow
			warped Optical Flow
			RGB Frame Difference
			HOF - Histogram of Gradient and Histogram of Flow
			HOG - 3D Histogram of Gradient
			MBH
			dense trajectories


	'''

	//Master Group发送给特征提取模块的命令
	'''
	int HEV_ExtractionCommand(int code);

		'''
		code definition:
			1 - start
			2 - pause
			3 - resume
			4 - stop

	'''

	//Master Group发送给特征提取模块的命令
	'''
	struct VideoSegmentSerialStructure{
		int videoIdx,
		int classIdx,
		int startFrmIdx,
		int endFrmIdx,
		int fWidth,
		int fHeight,
		int vLength,
		char frames[],
		char OptFlow[],
		...
	}


* 网络训练 - Training Module

* 特征融合 - Fusion Module
	BoW
	Fisher vector
	Ave pooling
	max pooling
	top-k pooling
	linear weighting
	attention weighting
	VLAD
	compact bilinear pooling

* 主控集群 - Master Group

