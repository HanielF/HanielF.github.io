---
title: GNN和图聚类
comments: true
mathjax: false
date: 2021-05-09 13:11:01
tags:
  [
    GNN,
    Clustering,
    Graph,
    DeepLearning,
    UnstructuredData,
  ]
categories: MachineLearning
urlname: gnn-graph-clustering
---

<meta name="referrer" content="no-referrer" />

{% note info %}
遇到一个聚类问题，感觉可以建模到GNN上面。

**特点：**
1. 无监督的多模态数据
2. 聚类成N个类别，这个类别数目不确定
3. 会有新节点加入
4. 如果训练端到端的模型，成本可能很高，每次都得聚类一次
5. 图的数量很大，并且大小不确定，从几十到大几百上千不等
6. node之间的link很稀疏并且是顺序连接，部分连边可能并不代表真实的联系
7. 如果是自己设置图的连边，那阈值也都不同，如果不建模到图上，就不存在连边问题，直接是一堆样本进行聚类
8. 或许可以用正则出来的集数作为连边信息，只要是集数连着就有边，没有集数就没有边，semi-supervised

**几个方向：**
1. ~~Link Prediction~~
   1. 不适用，因为这个是在有命名实体的情况下预测连接
   2. 想想知识图谱
2. Graph Clustering
   1. (DAEGC)《Attributed Graph Clustering via Adaptive Graph Convolution》
   2. 比较像
3. Community Detection
   1. 发现共享相似属性或功能的顶点组来了解网络数据
   2. 有很多基于顶点embedding的聚类算法
   3. 《CommunityGAN: Community Detection with Generative Adversarial Nets》
   4. 《GEMSEC: Graph Embedding with Self Clustering》这个不太行，nodes are clustered into a fixed number of groups，而且需要图结构。但是，它是在sequence based graph（拥有相似邻居的节点在空间上更接近）上做的
4. ~~Graph Reconstruction~~
   1. 《Multi-View Self-Constructing Graph Convolutional Networks With Adaptive Class Weighting Loss for Semantic Segmentation》直接通过输入feature构建基础图，而不是依赖先验知识。这居然是用在图像的语义分割上...
5. ~~dynamic graph clustering~~
   1. 《Interpretable Clustering on Dynamic Graphs with Recurrent Graph Neural Networks》用一个动态随机块模型来捕获这些变化，并提出一种基于衰减的简单聚类算法，该算法基于节点之间的加权连接对节点进行聚类，其中权重随时间以固定速率降低。还基于半监督图聚类提出了两种新的RNN-GCN结构
   2. 节点之间的连接以及节点的聚类成员身份可能会随时间而变化，但是并不是新加入节点，和合集的场景不太一样
6. 或许不应该focus在端到端的模型上面，应该看看传统的机器学习算法，毕竟可以得到确定性的结果同时迭代新的数据。
7. 可以利用上正则的匹配结果，作为部分label进行semi-supervised learning，这样的目的其实是为了提高召回，而不是precision

{% endnote %}

<!--more-->

## 基于Graph

### Chinese Whisper

基于图的聚类算法，代表性的基础算法是Chinese Whisper

- 初始化：将所有的样本点初始化为不同的类，自下而上的进行聚类
- 建图：**根据样本点之间的距离建图**，距离较远的两个点之间的边权重值较低，而距离较近的两个点之间的边权重值较高。是否连边需要**设置阈值**
- 迭代：
  1. 随机选取一个节点i开始，在其相连节点中选取边权重最大者j, 并将i归为节点j类（若相连节点中有多个节点属于同一类，则将这些权重相加再做比较）
  2. 遍历所有节点后，重复迭代至满足迭代次数
- 优点：不用设定k，只需指定相似度阈值
- 缺点：
  1. 对向量的要求度较高，需要向量能够增大类间距离，减小类内距离
  2. 由于随机初始化，每次聚类的结果有可能不一致
- 优化
  1. CW需要自行设置相似度阈值，且该阈值敏感度较高，后续优化方向是自动选择阈值
  2. Linkage Based Face Clustering via GCN（CVPR2019）

### 谱聚类spectral clustering

- [刘建平-谱聚类原理](https://www.cnblogs.com/pinard/p/6221564.html)
- 解决的问题是kmeans中无法对非欧式空间的分布进行聚类的问题，主要原理是对聚类数据进行变换，然后进行k-means聚类，之后再还原到原空间。
- 算法思路：它的主要思想是把所有的数据看做空间中的点，这些点之间可以用边连接起来。距离较远的两个点之间的边权重值较低，而距离较近的两个点之间的边权重值较高，通过对所有数据点组成的图进行切图，让切图后不同的**子图间边权重和尽可能的低**，而**子图内的边权重和尽可能的高**，从而达到聚类的目的。
- 度矩阵
  - 度$d_i$定义为和它相连的所有边的权重之和，对于没有边连接的两个点之间$w=0$。权重可以自己输入
  - 得到$nxn$的度矩阵𝐷，是个对角阵
- 邻接矩阵
  - 所有点之间的权重值，我们可以得到图的邻接矩阵
  - $nxn$，第i行的第j个值对应我们的权重 $w_{ij}$
  - 通过样本点距离度量的相似矩阵𝑆来获得邻接矩阵𝑊。构建邻接矩阵𝑊的方法有三类。
    - 𝜖-邻近法：
      - 用节点间欧式距离和阈值𝜖比，取 $w_{ij} = \epsilon\ if\ dis\ >\ \epsilon\ else\ 0$
      - 两点间的权重要不就是𝜖,要不就是0，距离远近度量很不精确，因此在实际应用中，我们很少使用𝜖-邻近法。
    - K邻近法：
      - KNN算法遍历所有的样本点，取每个样本最近的k个点作为近邻。只有距离最近的k个点之间的 $w>0$，图中用的RBF高斯核
      - RBF高斯核，$K(x,x') = \exp -\frac{||x-x'||_2^2}{2\sigma^2}$ 值随距离增大而减小，介于0-1之间。也会一种现成的相似性度量表示
      - 重构之后的邻接矩阵W非对称，但算法需要对称邻接矩阵
      - ![0nS2HD](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/0nS2HD.png)
    - 全连接法：
      - 所有的点之间的权重值都大于0，因此称之为全连接法
      - 可以选择不同的核函数来定义边权重，常用的有多项式核函数，高斯核函数和Sigmoid核函数。最常用的是高斯核函数RBF，此时相似矩阵和邻接矩阵相同
      - $w_{ij} = \exp -\frac{||x_i - x_j||^2_2}{2\sigma ^2}$
- 拉普拉斯矩阵
  - $L = D - W$，其中D是度矩阵对角阵， W是邻接矩阵
  - L是对称矩阵，这可以由𝐷和𝑊都是对称矩阵而得
  - L是对称矩阵，则它的所有的特征值都是实数
  - 对于任意的向量 $f$，$f^T L f = \sum_{i,j=1}^n w_{ij}(f_i-f_j)^2$。可以通过 $f^T L f = f^T D f - f^T W f$得到
  - L是半正定的，且对应的n个实数特征值都大于等于0
  - L是半正定的，且对应的n个实数特征值都大于等于0，即 $0 = \lambda_1 \le \lambda_2 \le .. \lambda_n$， 且最小的特征值为0
- 子图
  - 点集$V$的的一个子集$A \in V$
  - $vol(A) := \sum_{i\in A} d_i$
- 切图
  - 就是把 $G(V, E)$切成相互没有连接的k个子图，然后合起来就是完整的图G
  - 任意两个切图AB之间权重是： $W(A, B) = \sum_{i\in A, j\in B} w_{ij}$
  - 定义切图cut： $cut(A_1, A_2...A_k) = \frac{1}{2} \sum_{i=1}^k W(A_i, \overline{A_i})$，其中 $\overline{A_i}$其实就是V中A的补集
  - 直接最小化切图，不是最优的切图
  - 谱聚类有两种切图方式：RatioCut和cut
  - RatioCut：
    - 除了最小化切图，还考虑最大化每个子图点的个数
    - ![px7w07](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/px7w07.png)
    - ![eTmcmA](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/eTmcmA.png)
  - NutCut：
    - Ncut切图和RatioCut切图很类似，但是把Ratiocut的分母|𝐴𝑖|换成𝑣𝑜𝑙(𝐴𝑖). 由于子图样本的个数多并不一定权重就大，我们切图时基于权重也更合我们的目标，因此一般来说Ncut切图优于RatioCut切图
    - ![rpF8Yi](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/rpF8Yi.png)
    - ![8nzn56](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/8nzn56.png)
  - 总结：
    - ![J5TpAk](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/J5TpAk.png)
- 流程
  - 输入：n个样本, 类别k
  - 根据输入的相似度的衡量方式对样本建图，根据相似图建立邻接矩阵W
  - 计算出拉普拉斯矩阵L, 其中L=D-W， D为度矩阵
  - 计算L的最小的k个特征向量u1, u2,...,uk（此步相当于降维),将这些向量组成为n*k维的矩阵U
  - 将U中的每一行作为一个样本，共n个样本，使用k-means对这n个样本进行聚类
  - 得到簇划分C(c1,c2,...ck).
- 缺点是要设置K，和k-means一样
- 谱聚类算法的主要优点有：
  - 谱聚类只需要数据之间的相似度矩阵，因此对于处理稀疏数据的聚类很有效。这点传统聚类算法比如K-Means很难做到
  - 由于使用了降维，因此在处理高维数据聚类时的复杂度比传统聚类算法好。
- 算法的主要缺点有：
  - 如果最终聚类的维度非常高，则由于降维的幅度不够，谱聚类的运行速度和最后的聚类效果均不好。
  - 聚类效果依赖于相似矩阵，不同的相似矩阵得到的最终聚类效果可能很不同。

## 基于GNN

### 基于GCN聚类

paper： Learning to Cluster Faces on Affinity Graphy(CVPR2019)

流程：
1. 将样本点建图(如何建图可以参照前面图聚类算法，实际上有很多种构建方式，主要取决于相似度如何衡量以及如何建边）
2. Cluster Proposal: 从前面建好的图中选择出多个sub-graph，也就是proposal，就是子图的概念。主要是为了降低计算消耗，在子图上去做聚类。每一个cluster proposal里的向量数是比较少的。比如说一个包含1M向量的全图可以转化为50K包含20向量的proposal
3. Cluster Detection: 将上面的Cluster Proposal利用GCN提取特征后聚类，计算预测类别与gt的差异，评价指标为，$IoU(P)=$和$IoP(P)$
4. Cluster Segmentation: 上面那个模块是检测，这个模块是剔除负样本点，评价指标和上面那一步是一样的。使用的loss均为MSE，训练然后梯度下降

### CDP(ECCV2018)

是针对人脸识别提出的优化算法，解决的是在大数据集下传统算法聚类性能过差的问题

- 是**半监督**方式的
- 该算法有3个部分
  - base model, 建一个大的**KNN图**
  - committe model(决策者模型),多个model, 对大图每一条边，判断是否断开，得到多个子图
  - Mediator(融合模型), 根据多个committe的结果来判断两个节点之间的边是保留还是断开，多数投票，最后输出聚类结果
- 各模块**都用GCN训练**，**非设置阈值**
- 优点：
  - 只探索局部关系，因为它的主要计算量集中在两个节点组成的pairs的关系，而不是整个图的关系，计算效率较高，可以用于大规模数据集

> We first provide an overview of the proposed approach. Our approach consists of three stages:
>
> 1) Supervised initialization - Given a small portion of labeled data, we separately train the base model and committee members in a fully-supervised manner. More precisely, the base model B and all the N committee members {Ci|i = 1, 2, . . . , N} learn a mapping from image space to feature space Z using labeled data Dl. For the base model, this process can be denoted as the mapping: FB : Dl 􏰉→ Z, and as for committee members: FCi : Dl 􏰉→ Z, i = 1,2,...,N. 
>
> 2) Consensus-driven propagation - CDP is applied on unlabeled data to se- lect valuable samples and conjecture labels thereon. The framework is shown in Fig. 1. We use the trained models from the first stage to extract deep features for unlabeled data and create k-NN graphs. The “committee” ensures the diver- sity of the graphs. Then a “mediator” network is designed to aggregate diverse opinions in the local structure of k-NN graphs to select meaningful pairs. With the selected pairs, a consensus-driven graph is created on the unlabeled data and nodes are assigned with pseudo labels via our label propagation algorithm.
>
> 3) Joint training using labeled and unlabeled data - Finally, we re-train the base model with labeled data, and unlabeled data with pseudo labels, in a multi-task learning framework.

### DAEGC(IJCAI2019)

《Attributed Graph Clustering: A Deep Attentional Embedding Approach》

![MGUaZE](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/MGUaZE.png)

强调的是goal-directed的embedding方法，用attention based网络捕捉相邻节点之间的importance，并且结合了structure和node content信息作为compact representation。通过graph embedding产生的soft labels进行自监督聚类. 自监督过程和graph embedding优化过程是joint learned and optimized。

- Require:
  - Graph G with n nodes;
  - Number of clusters k;
  - Number of iterations Iter;
  - Target distribution update interval T;
  - Clustering Coefficient γ.
- Output:
  - k disjoint groups

### AGC

graph-clustering SOTA论文

《Attributed Graph Clustering via Adaptive Graph Convolution》

![in0VDA](https://cdn.jsdelivr.net/gh/HanielF/ImageRepo@main/others/in0VDA.png)

在使用k-order GCN之后，用了谱聚类。问题是谱聚类也要指定k值，所以论文里面是通过实验找到最好的k值...

## 思路

1. 算法不能提前指定类别数
2. 边的确定
   1. ep是merge了pid_ep和photo_ep
   2. 根据ep来建立连接关系，每个1都和2相连，每个2都和3相连
   3. 所有的-1，按照时间顺序，连接前后W个视频，W是窗口大小，设为3
   4. 