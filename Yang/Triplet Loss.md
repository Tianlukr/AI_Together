# 论文阅读笔记
## 论文一《Deep feature learning with relative distance comparison for person re-identification》
### 简介
每个triplet单元包括3张图片，分别是anchor(query image),positive,negative，目的是使每个triplet中matched pair和mismatched pair的相对距离最大化，前者比后者小，采用L2 distance。由于每张图片会在很多个triplet出现，triplet的数量很多，为解决这个问题，采用一种triplets generation scheme和一种梯度下降算法，使得计算量主要取决于图片数量而不是triplet数量。采用这个模型的原因，其一，解决类内差异、类内相似问题，其二，生成多于图片数量的triplets，加入了更多的限制constraints，缓解过拟合。
### 网络结构如下：
![](https://github.com/Tianlukr/AI_Together/blob/master/Yang/net.PNG)
* 1st卷积层：32kernels，5*5*3，s=2；       2nd卷积层：32kernels，5*5*32，s=1；       fc:400
* 第二层卷积后会有一个对特征进行归一化的操作，目的是使得loss计算的triplet 距离不易超过C，使得训练过程中加入更多的triplet constraint，因为不超过C的tripet loss是没有梯度的，因此只有难到一定程度的triplets才能贡献梯度。
### triplet-based gradient descent algorithm 
![推导](https://github.com/Tianlukr/AI_Together/blob/master/Yang/algorithm1.PNG) 
* 其中C是为了防止容易识别的triplet造成loss太小，给出一个下界，这里c=-1.
![伪代码](https://github.com/Tianlukr/AI_Together/blob/master/Yang/Algorithm_1.PNG)
* 对于每个triplet，需要计算Fw和Fw对w的偏导，共6项（FW是网络的输出）
### image-based gradient descent algorithm 
* 基于triplet的梯度算法中，每个triplet有3次前向和3次后向，而同一张图片会在不同的triplet中出现，于是有了改进的基于image梯度下降算法，来减少重复计算。
传统的方法，对于单张图片：loss对每层的W偏导=loss对每层的特征图X求偏导*X对W求偏导，然后把所有图片的偏导值相加除以总图片数即可
![推导](https://github.com/Tianlukr/AI_Together/blob/master/Yang/image_based.PNG)
* 用算法3计算f对Fw的偏导。偏导的形式与图片Ik在triplet中的位置有关 （这里不懂！）
![算法3](https://github.com/Tianlukr/AI_Together/blob/master/Yang/Algorithm_3.PNG)
* 然后求得的值带入算法2
![算法2](https://github.com/Tianlukr/AI_Together/blob/master/Yang/Algorithm_2.PNG)
### triplets generation scheme
若有M个人，每个人有N张图，那么可生成M(M-1)NN(N-1)个triplets。如果每次迭代都是随机地取triplets，有些简单的triplets由于c的存在而不贡献梯度，那么只能在这些triplets的图片上加很少的限制，效率低。因此，每次迭代，随机选取一小部分的persons,只用这些persons生成triplets，那么可以少量的图片却加了很多的限制（这样为什么就可以增加难样本的比例而加更多的限制了呢？）。再结合我们的算法来避免计算相同图片的梯度，这样计算量主要取决于图片数量而不是triplet数量。
 ### 测试与训练
 把数据集平均分成训练集和测试集，然后把测试集分成gallery set（每个人只对应一张图片）和probe set，对probe的每张图，从gallery中选出最接近的n张。
 CMC曲线，横坐标是n，纵坐标是所有图片的rank-n准确率取平均值
参考博客：https://www.cnblogs.com/jermmyhsu/p/8257981.html
         https://blog.csdn.net/qq_28659831/article/details/80805291 
## 论文二《In Defense of the Triplet Loss for Person Re-Identification》
