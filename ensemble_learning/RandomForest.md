# 随机森林算法梳理


## 1. 集成学习的概念

集成学习(ensemble learning)通过构建并结合多个学习器来完成学习任务，集成学习的一般结构为:先产生一组"个体学习器" (individual learner) ，
再用某种策略将它们结合起来。示意图如下：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/ensemble_learning/Supp_RF/ensemble.png)

## 2. 个体学习器的概念
个体学习器通常由一个现有的学习算法从训练数据产生，例如C4.5 决策树算法、BP 神经网络算法等，此时集成中只包含同种类型的个体学习器，
例如"决策树集成"中全是决策树"；神经网络集成"中全是神经网络，这样的集成是"同质"的(homogeneous) 。同质集成中的个体学习器亦称"基学习器" (base learner),
相应的学习算法称为"基学习算法" (base learning algorithm). 集成也可包含不同类型的个体学习器，
例如同时包含决策树和神经网络，这样的集成是"异质"的(heterogenous) 。
异质集成中的个体学习器由不同的学习算法生成，这时就不再有基学习算法;相应的，
个体学习器一般不称为基学习器，常称为"组件学习器" (component learner) 或直接称为个体学习器.

## 3. boosting bagging的概念、异同点
### Boosting
**Boosting** 是一族可将弱学习器提升为强学习器的算法.这族算法的工作机
制类似:先从初始训练集训练出一个基学习器，再根据基学习器的表现对训练
样本分布进行调整，使得先前基学习器做错的训练样本在后续受到更多关注，
然后基于调整后的样本分布来训练下一个基学习器;如此重复进行，直至基学
习器数目达到事先指定的值T ， 最终将这T 个基学习器进行加权结合.<br>

### Bagging
**Bagging** 是井行式集成学习方法最著名的代表.它直接基于自助来样法(bootstrap sampling).
给定包含m 个样本的数据集，我们先随机取出一个样本放入采样集中，再把该
样本放回初始数据集，使得下次采样时该样本仍有可能被选中，这样，经过m
次随机采样操作，我们得到含m 个样本的采样集，初始训练集中有的样本在采
样集里多次出现，有的则从未出现.初始训练集中约有63.2%
的样本出现在来样集中.
照这样，我们可采样出T 个含m 个训练样本的采样集，然后基于每个采样
集训练出一个基学习器，再将这些基学习器进行结合.这就是Bagging 的基本
流程.在对预测输出进行结合时， Bagging 通常对分类任务使用简单投票法，对
回归任务使用简单平均法.若分类预测时出现两个类收到同样票数的情形，则
最简单的做法是随机选择一个，也可进一步考察学习器投票的置信度来确定最
终胜者.<br>

### Boosting与Bagging的异同点
**样本选择上：** Bagging采取Bootstraping的是随机有放回的取样，Boosting的每一轮训练的样本是固定的，改变的是买个样的权重。<br>

**样本权重上：** Bagging采取的是均匀取样，且每个样本的权重相同，Boosting根据错误率调整样本权重，错误率越大的样本权重会变大。<br>

**预测函数上：** Bagging所以的预测函数权值相同，Boosting中误差越小的预测函数其权值越大。<br>

**并行计算：** Bagging 的各个预测函数可以并行生成;Boosting的各个预测函数必须按照顺序迭代生成。<br>


## 4. 理解不同的结合策略(平均法，投票法，学习法)

### 平均法
对数值型输出![](http://latex.codecogs.com/gif.latex?\$$h_{i}(x)\in\mathbb{R}$$)，最常见的结合策略是使用平均法<br>
* 简单平均法<br>
![](http://latex.codecogs.com/gif.latex?\$$H(x)=\frac{1}{T}\sum_{i=1}^{T}h_{i}(x)$$)<br>
* 加权平均法<br>
![](http://latex.codecogs.com/gif.latex?\$$H(x)=\sum_{i=1}^{T}\omega_{i}h_{i}(x)$$)<br>

### 投票法
对分类任务来说，学习器将从类别标记集合中预测出一个标记, 最常见的结合策略是使用投票法(voting).<br>
* 绝对多数投票法<br>
若某标记得票过半数，则预测为该标记;否则拒绝预测.<br>
* 相对多数投票法<br>
预测为得票最多的标记，若同时有多个标记获最高票，则从中随机选取一个.<br>
* 加权投票法<br>
对各个学习器的结果进行加权后，使用相对多数投票法。<br>

### 学习法
当训练数据很多时，一种更为强大的结合策略是使用"学习法"，即通过另一个学习器来进行结合.Stacking是学习法
的典型代表.这里我们把个体学习器称为初级学习器，用于结合的学习器称为次级学习器。<br>
Stacking 先从初始数据集训练出初级学习器，然后"生成"一个新数据集
用于训练次级学习器.在这个新数据集中，初级学习器的输出被当作样例输入
特征，而初始样本的标记仍被当作样例标记.<br>


## 5. 随机森林的思想
随机森林(Random Forest ，简称RF) 是Baggi吨的一个
扩展变体.RF在以决策树为基学习器构建Bagging 集成的基础上，进一步在
决策树的训练过程中引入了随机属性选择.具体来说，传统决策树在选择划分
属性时是在当前结点的属性集合(假定有d 个属性)中选择一个最优属性;而在
RF 中，对基决策树的每个结点，先从该结点的属性集合中随机选择一个包含k
个属性的子集，然后再从这个子集中选择一个最优属性用于划分. 这里的参数
k 控制了随机性的引入程度: 若令k = d ， 则基决策树的构建与传统决策树相同;
若令k = 1 ， 则是随机选择一个属性用于划分; 一般情况下，推荐值k = log2d.<br>


## 6. 随机森林的推广
由于RF在实际应用中的良好特性，基于RF，有很多变种算法，应用也很广泛，不光可以用于分类回归，还可以用于特征转换，异常点检测等。下面对于这些RF家族的算法中有代表性的做一个总结。<br>

### extra trees
extra trees是RF的一个变种, 原理几乎和RF一模一样，区别仅有：<br>
  1） 对于每个决策树的训练集，RF采用的是随机采样bootstrap来选择采样集作为每个决策树的训练集，而extra trees一般不采用随机采样，即每个决策树采用原始训练集。<br>
  2） 在选定了划分特征后，RF的决策树会基于基尼系数，均方差之类的原则，选择一个最优的特征值划分点，这和传统的决策树相同。但是extra trees比较的激进，他会随机的选择一个特征来划分决策树。<br>
从第二点可以看出，由于随机选择了特征的划分点位，而不是最优点位，这样会导致生成的决策树的规模一般会大于RF所生成的决策树。也就是说，模型的方差相对于RF进一步减少，但是偏倚相对于RF进一步增大。在某些时候，extra trees的泛化能力比RF更好。<br>
    
### Totally Random Trees Embedding
Totally Random Trees Embedding(以下简称 TRTE)是一种非监督学习的数据转化方法。它将低维的数据集映射到高维，从而让映射到高维的数据更好的运用于分类回归模型。我们知道，在支持向量机中运用了核方法来将低维的数据集映射到高维，此处TRTE提供了另外一种方法。<br>
TRTE在数据转化的过程也使用了类似于RF的方法，建立T个决策树来拟合数据。当决策树建立完毕以后，数据集里的每个数据在T个决策树中叶子节点的位置也定下来了。比如我们有3颗决策树，每个决策树有5个叶子节点，某个数据特征x划分到第一个决策树的第2个叶子节点，第二个决策树的第3个叶子节点，第三个决策树的第5个叶子节点。则x映射后的特征编码为(0,1,0,0,0,     0,0,1,0,0,     0,0,0,0,1), 有15维的高维特征。这里特征维度之间加上空格是为了强调三颗决策树各自的子编码。
映射到高维特征后，可以继续使用监督学习的各种分类回归算法了。<br>

### Isolation Forest
Isolation Forest（以下简称IForest）是一种异常点检测的方法。它也使用了类似于RF的方法来检测异常点。<br>
对于在T个决策树的样本集，IForest也会对训练集进行随机采样,但是采样个数不需要和RF一样，对于RF，需要采样到采样集样本个数等于训练集个数。但是IForest不需要采样这么多，一般来说，采样个数要远远小于训练集个数？为什么呢？因为我们的目的是异常点检测，只需要部分的样本我们一般就可以将异常点区别出来了。<br>
对于每一个决策树的建立， IForest采用随机选择一个划分特征，对划分特征随机选择一个划分阈值。这点也和RF不同。<br>
另外，IForest一般会选择一个比较小的最大决策树深度, 原因同样本采集，用少量的异常点检测一般不需要这么大规模的决策树。<br>

**Reference:**<br>
1. [Bagging与随机森林算法原理小结 ](https://www.cnblogs.com/pinard/p/6156009.html)<br><br>


## 7. 随机森林的优缺点

### 优点
1. 表现性能好，与其他算法相比有着很大优势。<br>
2. 随机森林能处理很高维度的数据（也就是很多特征的数据），并且不用做特征选择。<br>
3. 在训练完之后，随机森林能给出哪些特征比较重要。<br>
4. 训练速度快，容易做成并行化方法(训练时，树与树之间是相互独立的)。<br>
5. 在训练过程中，能够检测到feature之间的影响。<br>
6. 对于不平衡数据集来说，随机森林可以平衡误差。当存在分类不平衡的情况时，随机森林能提供平衡数据集误差的有效方法。<br>
7. 如果有很大一部分的特征遗失，用RF算法仍然可以维持准确度。<br>
8. 随机森林算法有很强的抗干扰能力（具体体现在6,7点）。所以当数据存在大量的数据缺失，用RF也是不错的。<br>
9. 随机森林抗过拟合能力比较强（虽然理论上说随机森林不会产生过拟合现象，但是在现实中噪声是不能忽略的，增加树虽然能够减小过拟合，但没有办法完全消除过拟合，无论怎么增加树都不行，再说树的数目也不可能无限增加的。）<br>
10. 随机森林能够解决分类与回归两种类型的问题，并在这两方面都有相当好的估计表现。（虽然RF能做回归问题，但通常都用RF来解决分类问题）。<br>
11. 在创建随机森林时候，对generlization error(泛化误差)使用的是无偏估计模型，泛化能力强。<br>


### 缺点
1. 随机森林在解决回归问题时，并没有像它在分类中表现的那么好，这是因为它并不能给出一个连续的输出。当进行回归时，随机森林不能够做出超越训练集数据范围的预测，这可能导致在某些特定噪声的数据进行建模时出现过度拟合。（PS:随机森林已经被证明在某些噪音较大的分类或者回归问题上回过拟合）。<br>
2. 对于许多统计建模者来说，随机森林给人的感觉就像一个黑盒子，你无法控制模型内部的运行。只能在不同的参数和随机种子之间进行尝试。<br>
3. 可能有很多相似的决策树，掩盖了真实的结果。<br>
4. 对于小数据或者低维数据（特征较少的数据），可能不能产生很好的分类。（处理高维数据，处理特征遗失数据，处理不平衡数据是随机森林的长处）。<br>
5. 执行数据虽然比boosting等快（随机森林属于bagging），但比单只决策树慢多了。<br>


**Reference:**<br>
1. [随机森林（Random forest,RF）的生成方法以及优缺点 ](https://blog.csdn.net/zhongjunlang/article/details/79488955)<br>
1. [随机森林的优缺点 ](https://blog.csdn.net/xingxing1839381/article/details/81273793)<br><br>


## 8. 随机森林在sklearn中的参数解释

### 决策树的参数
1. criterion: ”gini” or “entropy”(default=”gini”)是计算属性的gini(基尼不纯度)还是entropy(信息增益)，来选择最合适的节点。<br>
2. splitter: ”best” or “random”(default=”best”)随机选择属性还是选择不纯度最大的属性，建议用默认。<br>
3. max_features: 选择最适属性时划分的特征不能超过此值。当为整数时，即最大特征数；当为小数时，训练集特征数x小数。<br>
4. max_depth: (default=None)设置树的最大深度，默认为None，这样建树时，会使每一个叶节点只有一个类别，或是达到min_samples_split。<br>
5. min_samples_split:根据属性划分节点时，每个划分最少的样本数。<br>
6. min_samples_leaf:叶子节点最少的样本数。<br>
7. max_leaf_nodes: (default=None)叶子树的最大样本数。<br>
8. min_weight_fraction_leaf: (default=0) 叶子节点所需要的最小权值<br>

### 随机森林特有的参数
1. n_estimators=10：决策树的个数，越多越好，但是性能就会越差，至少100左右（具体数字忘记从哪里来的了）可以达到可接受的性能和误差率。<br>
2. bootstrap=True：是否有放回的采样。  <br>
3. oob_score=False：oob（out of band，带外）数据，即：在某次决策树训练中没有被bootstrap选中的数据。多单个模型的参数训练，我们知道可以用cross validation（cv）来进行，但是特别消耗时间，而且对于随机森林这种情况也没有大的必要，所以就用这个数据对决策树模型进行验证，算是一个简单的交叉验证。性能消耗小，但是效果不错。  <br>
4. n_jobs=1：并行job个数。这个在ensemble算法中非常重要，尤其是bagging（而非boosting，因为boosting的每次迭代之间有影响，所以很难进行并行化），因为可以并行从而提高性能。1=不并行；n：n个并行；-1：CPU有多少core，就启动多少job。<br>
5. warm_start=False：热启动，决定是否使用上次调用该类的结果然后增加新的。  <br>
6. class_weight=None：各个label的权重。  <br>

**Reference:**<br>
1. [sklearn中随机森林的参数](https://www.cnblogs.com/harvey888/p/6512312.html)<br><br>
 

## 9. 随机森林的应用场景
随机森林在现实分析中被大量使用，它相对于决策树，在准确性上有了很大的提升，同时一定程度上改善了决策树容易被攻击的特点。该算法适用于数据维度相对低（几十维），同时对准确性有较高要求时。因为不需要很多参数调整就可以达到不错的效果，基本上不知道用什么方法的时候都可以先试一下随机森林。

**Reference:**<br>
1. [各种机器学习的应用场景分别是什么？](https://baijiahao.baidu.com/s?id=1586018185986909021&wfr=spider&for=pc)<br><br>


