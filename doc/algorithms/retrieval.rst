图像检索
========

词袋用来在大图集里找出可能重叠的图像对，而不是把每对都做一次匹配。顺序匹配里的回环检测也走同一套索引。

接口是 ``src/colmap/retrieval/visual_index.h`` 的 ``VisualIndex``。具体类是 ``visual_index.cc`` 里的 ``FaissVisualIndex<128, 64>``。注释里对应的论文是 Schönberger 等人的 vote-and-verify（ACCV 2016），以及 Arandjelovic 和 Zisserman 的突发性和 IDF（ACCV 2014）。


建索引
------

``VisualIndex::Build()`` 做四件事：

1. 用 FAISS 工厂串 ``IVF{num_centroids},ITQ{dim},SH`` 把训练描述子量化成视觉单词。
2. ``InvertedIndex::Initialize(num_words)``。每个视觉单词一条 ``InvertedFile``，定义在 ``retrieval/inverted_file.h``。
3. ``GenerateHammingEmbeddingProjection()``：随机高斯矩阵做 QR，取前 ``kEmbeddingDim`` 行，作为汉明嵌入的投影。
4. ``ComputeHammingEmbedding()``：每个单词上，用投影后描述子的中位数当二值化阈值。

``Add()`` 对每个特征调用 ``FindWordIds()``，把描述子分到 ``num_neighbors`` 个视觉单词，再 ``InvertedIndex::AddEntry()``。条目里除了描述子，还有 ``FeatureGeometry``：x、y、尺度、方向。


查询
----

``Query()`` 的顺序：

1. 查询描述子同样分到视觉单词。
2. ``InvertedFile::ScoreFeature()`` 把描述子投影并二值化，和倒排表里的条目算汉明距离，再用 ``HammingDistWeightFunctor`` 加权。
3. 分数按突发度做 ``score / sqrt(num_votes)``，并乘上 IDF。图像之间再按自相似和每张图的常数归一化。
4. 若 ``num_images_after_verification > 0`` 且提供了关键点，就取出与排名靠前图像的汉明匹配，用斐波那契堆做成一对一，再调用 ``VoteAndVerify()``。空间验证的内点数加回检索分数，然后重排。

``VocabTreePairGenerator::Query()`` 在需要空间验证时走第 4 步。


Vote-and-verify
---------------

实现在 ``src/colmap/retrieval/vote_and_verify.cc``。输入是带几何的特征匹配 ``FeatureGeometryMatch``。

1. 在多分辨率的 ``VotingBin`` 里对相似变换投票。箱子覆盖尺度、转角和平移。
2. 得票最高的若干变换送进 ``RANSAC<AffineTransformEstimator>``。估计器在 ``estimators/solvers/affine_transform.*``。
3. 可选：在内点上重拟合仿射，做一次局部优化。
4. 可选：按空间分箱计算有效内点数，避免内点挤在一小块区域里。

返回值是内点数。它加到检索分数上，用来把几何上说不通的近邻压下去。这一步只决定图像对，不代替后面的本质矩阵或基础矩阵估计。
