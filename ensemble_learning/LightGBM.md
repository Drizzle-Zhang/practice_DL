# LightGBM算法梳理

## 1. 简介
LightGBM （Light Gradient Boosting Machine）是一个实现 GBDT 算法的框架，支持高效率的并行训练，并且具有以下优点：<br>
更快的训练速度 <br>
更低的内存消耗 <br>
更好的准确率 <br>
分布式支持，可以快速处理海量数据 <br><br>

## 2. LightGBM的起源
传统的boosting算法（如GBDT和XGBoost）已经有相当好的效率，但是在如今的大样本和高维度的环境下，传统的boosting似乎在效率和可扩展性上不能满足现在的需求了，主要的原因就是传统的boosting算法需要对每一个特征都要扫描所有的样本点来选择最好的切分点，这是非常的耗时。为了解决这种在大样本高纬度数据的环境下耗时的问题，出现了Lightgbm 。<br>
Lightgbm使用了如下两种解决办法：一是GOSS（Gradient-based One-Side Sampling, 基于梯度的单边采样），不是使用所用的样本点来计算梯度，而是对样本进行采样来计算梯度；二是EFB（Exclusive Feature Bundling， 互斥特征捆绑） ，这里不是使用所有的特征来进行扫描获得最佳的切分点，而是将某些特征进行捆绑在一起来降低特征的维度，是寻找最佳切分点的消耗减少。这样大大的降低的处理样本的时间复杂度，但在精度上，通过大量的实验证明，在某些数据集上使用Lightgbm并不损失精度，甚至有时还会提升精度。<br>
<br>

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
<br>

## 3. Histogram VS pre-sorted

### 3.1 Histogram 算法
直方图算法的基本思想是先把连续的浮点特征值离散化成k个整数，同时构造一个宽度为k的直方图。在遍历数据的时候，根据离散化后的值作为索引在直方图中累积统计量，当遍历一次数据后，直方图累积了需要的统计量，然后根据直方图的离散值，遍历寻找最优的分割点。 <br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/histogram.png)<br>

### 3.2 pre-sorted算法
xgboost算法是基于预排序方法，这种构建决策树的算法基本思想是：<br>

* 首先，对所有特征都按照特征的数值进行预排序。<br>
* 其次，在遍历分割点的时候用O(data)的代价找到一个特征上的最好分割点。<br>
* 最后，找到最好的分割点后，将数据分裂成左右子节点。<br>

这样做能精确地找到分割点。但是空间消耗大，时间消耗也大，对cache优化不好。预排序后，特征对梯度的访问是一种随机访问，并且不同的特征访问的顺序不一样，无法对cache进行优化。同时，在每一层，需要随机访问一个行索引到叶子索引的数组，并且不同特征访问的顺序也不一样。<br>

### 3.2 Histogram与pre-sorted的比较
#### 优势
* Pre-sorted 算法需要的内存约是训练数据的两倍(2 * #data * #features* 4Bytes)，它需要用32位浮点(4Bytes)来保存 feature value，并且对每一列特征，都需要一个额外的排好序的索引，这也需要32位(4Bytes)的存储空间。因此是(2 * #data * #features* 4Bytes)。而对于 histogram 算法，则只需要(#data * #features * 1Bytes)的内存消耗，仅为 pre-sorted算法的1/8。因为 histogram 算法仅需要存储 feature bin value (离散化后的数值)，不需要原始的 feature value，也不用排序，而 bin value 用 1Bytes(256 bins) 的大小一般也就足够了。<br>
* 计算上的优势则是大幅减少了计算分割点增益的次数。对于每一个特征，pre-sorted 需要对每一个不同特征值都计算一次分割增益，代价是O(#feature*#distinct_values_of_the_feature)；而 histogram 只需要计算#bins次，代价是O(#feature*#bins)。<br>
* 还有一个很重要的点是cache-miss。事实上，cache-miss对速度的影响是特别大的。预排序中有2个操作频繁的地方会造成cache miss，一是对梯度的访问，在计算gain的时候需要利用梯度，不同特征访问梯度的顺序都是不一样的，且是随机的，因此这部分会造成严重的cache-miss。二是对于索引表的访问，预排序使用了一个行号到叶子节点号的索引表（row_idx_to_tree_node_idx ），来防止数据切分时对所有的数据进行切分，即只对该叶子节点上的样本切分。在与level-wise进行结合的时候， 每一个叶子节点都要切分数据，这也是随机的访问。这样会带来严重的系统性能下降。而直方图算法则是天然的cache friendly。在直方图算法的第3个for循环的时候，就已经统计好了每个bin的梯度，因此，在计算gain的时候，只需要对bin进行访问，造成的cache-miss问题会小很多。<br>
* 最后，在数据并行的时候，用 histgoram 可以大幅降低通信代价。用 pre-sorted 算法的话，通信代价是非常大的（几乎是没办法用的）。所以 xgoobst 在并行的时候也使用 histogram 进行通信。<br>


#### 劣势
* histogram 算法不能找到很精确的分割点，训练误差没有 pre-sorted 好。但从实验结果来看， histogram 算法在测试集的误差和 pre-sorted 算法差异并不是很大，甚至有时候效果更好。实际上可能决策树对于分割点的精确程度并不太敏感，而且较“粗”的分割点也自带正则化的效果，再加上boosting算法本身就是弱分类器的集成。<br>

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法总结](https://blog.csdn.net/weixin_39807102/article/details/81912566)<br>
<br>


## 4. leaf-wise VS level-wise
lightGBM模型采用带深度限制的Leaf-wise的叶子生长策略<br>

### level-wise
绝大多数的GBDT工具使用按层生长的决策树生长策略。level-wise过一次数据可以同时分裂同一层的叶子，容易进行多线程优化，也好控制模型复杂度，不容易过拟合。但实际上level-wise是一种低效的算法，因为它不加区分的对待同一层的叶子，带来了很多没必要的开销实际上很多叶子的分裂增益较低，没必要进行搜索和分裂。level-wise过程如下图所示：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/level_wise.png)<br>

### leaf-wise
leaf-wise是一种更为高效的策略，每次从当前所有叶子中，找到分裂增益最大的一个叶子节点，然后分裂。如此循环。同level-wise相比，在分裂次数相同的情况下，leaf-wise可以降低更多的误差，得到更好的精度。Leaf-wise的缺点是可能会长处比较深的决策树，产生过拟合。因此LightGBM在leaf-wise之上增加了一个最大深度的限制，在保证高效率的同时防止过拟合。leaf-wise过程如下图所示：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/leaf_wise.png)<br>

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法梳理](https://blog.csdn.net/qq_32577043/article/details/86215754#levelwise_37)<br>
<br>

## 5. 特征并行和数据并行

### 5.1 特征并行
特征并行的主要思想是：不同机器在不同的特征集合上分别寻找最优的分割点，然后再机器间同步最优的分割点，示意图如下：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/feature_para.png)<br>
<br>
传统算法中的特征并行，主要是体现在找到最好的分割点，其步骤为：<br>
1.垂直分区数据（不同的线程具有不同的数据集）；<br>
2.在本地数据集上找到最佳分割点，包括特征，阈值；<br>
3.再进行各个划分的通信整合并得到最佳划分；<br>
4.以最佳划分方法对数据进行划分，并将数据划分结果传递给其他线程；<br>
5.其他线程对接受到的数据进一步划分；<br><br>
传统特征并行的缺点：<br>
计算成本较大，传统特征并行没有实现得到"split"（时间复杂度为“O（训练样本的个数)"）的加速。当数据量很大的时候，难以加速；<br>
需要对划分的结果进行通信整合，其额外的时间复杂度约为 “O（训练样本的个数/8）”（一个数据一个字节）<br><br>

由于特征并行在训练样本的个数大的时候不能很好地加速，LightGBM做了以下优化：不是垂直分割数据，而是每个线程都拥有完整的全部数据。因此，因此最优的特征分裂结果不需要传输到其他线程，只需要将最优特征以及分裂点告诉其他线程，随后再本地进行处理。实际上这是一种牺牲空间换取时间的做法。<br><br>

处理过程如下：<br>
1.每个worker在基于局部的特征集合找到最优分裂特征。<br>
2.worker间传输最优分裂信息，并得到全局最优分裂信息。<br>
3.每个worker基于全局最优分裂信息，在本地进行数据分裂。<br><br>

### 5.2 数据并行
数据并行的思路如下图所示：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/data_para.png)<br>
传统算法数据并行旨在并行化整个决策学习。数据并行的过程是：<br>
1、水平划分数据；<br>
2、线程以本地数据构建本地直方图；<br>
3、将本地直方图整合成全局直方图；<br>
4、在全局直方图中寻找最佳划分，然后执行此划分；<br><br>
传统数据并行的缺点：通信成本高。如果使用点对点通信算法，则一台机器的通信成本约为O(#machine * #feature * #bin)。如果使用聚合通信算法（例如“All Reduce”），通信成本约为O(2 * #feature * #bin)。<br><br>

LightGBM中通过下面方法来降低数据并行的通信成本：<br>
1、不同于“整合所有本地直方图以形成全局直方图”的方式，LightGBM 使用分散规约(Reduce scatter)的方式对不同线程的不同特征（不重叠的）进行整合。 然后线程从本地整合直方图中寻找最佳划分并同步到全局的最佳划分中；<br>
2、LightGBM通过直方图的减法加速训练。 基于此，我们可以进行单叶子节点的直方图通讯，并且在相邻直方图上作减法；<br><br>
通过上述方法，LightGBM 将数据并行中的通讯开销减少到O(0.5 * #feature * #bin)。<br><br>

### 5.3 投票并行
基于投票机制的并行算法，是在每个worker中选出top k个分裂特征，然后将每个worker选出的k个特征进行汇总，并选出全局分裂特征，进行数据分裂。有理论证明，这种voting parallel以很大的概率选出实际最优的特征，因此不用担心top k的问题，原理图如下：
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_LightGBM/vote_para.png)<br>


参考资料：<br>
1. [高级算法梳理之LightGBM](https://blog.csdn.net/sun_xiao_kai/article/details/90377282)<br>
2. [LightGBM算法梳理](https://blog.csdn.net/qq_32577043/article/details/86215754#levelwise_37)<br>
<br>


## 6. 顺序访问梯度

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法总结](https://blog.csdn.net/weixin_39807102/article/details/81912566)<br>
3. [XGB算法梳理](https://blog.csdn.net/wangrongrongwq/article/details/86755915#2.%E7%AE%97%E6%B3%95%E5%8E%9F%E7%90%86)<br>
<br>

## 7. 支持类别特征

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法总结](https://blog.csdn.net/weixin_39807102/article/details/81912566)<br>
3. [XGB算法梳理](https://blog.csdn.net/wangrongrongwq/article/details/86755915#2.%E7%AE%97%E6%B3%95%E5%8E%9F%E7%90%86)<br>
<br>

## 8. sklearn参数

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法总结](https://blog.csdn.net/weixin_39807102/article/details/81912566)<br>
3. [XGB算法梳理](https://blog.csdn.net/wangrongrongwq/article/details/86755915#2.%E7%AE%97%E6%B3%95%E5%8E%9F%E7%90%86)<br>
<br>

## 9. CatBoost

参考资料：<br>
1. [机器学习算法梳理-LightGBM](https://blog.csdn.net/mingxiaod/article/details/86233309)<br>
2. [LightGBM算法总结](https://blog.csdn.net/weixin_39807102/article/details/81912566)<br>
3. [XGB算法梳理](https://blog.csdn.net/wangrongrongwq/article/details/86755915#2.%E7%AE%97%E6%B3%95%E5%8E%9F%E7%90%86)<br>
<br>






