匹配与几何验证
==============

匹配回答两件事：哪两张图要互相比，以及哪些特征是同一三维点的投影。原始匹配先写入 ``matches`` 表。几何验证再从中挑内点，写入 ``two_view_geometries``。重建只使用后者。


谁来产生图像对
--------------

工厂在 ``src/colmap/controllers/feature_matching.h``，配对逻辑在 ``controllers/pairing.cc``。

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - 生成器
     - 实际做法
   * - ``ExhaustivePairGenerator``
     - 按图像 id 排序后做块状两两配对，只保留 ``image_id1 < image_id2``。
   * - ``SequentialPairGenerator``
     - 与前后 ``overlap`` 张，以及二次重叠 ``2^o`` 张配对。可按 rig 扩展。每隔 ``loop_detection_period`` 张，用词袋再找回环。
   * - ``VocabTreePairGenerator``
     - 先把库中图像放进 ``VisualIndex``，再对每张查询取前 ``num_images`` 个检索结果。细节在 :doc:`retrieval`。
   * - ``SpatialPairGenerator``
     - 用位姿先验的位置做 FAISS 最近邻，距离和邻居数由选项限制。
   * - ``TransitivePairGenerator``
     - 从已经验证过的边出发，把 A–B 和 B–C 补成 A–C，重复 ``num_iterations`` 次。
   * - ``ImportedPairGenerator``
     - 直接读入外部给出的图像对列表。

``FeatureMatcherThread`` 在 ``controllers/feature_matching_utils.cc`` 里取下一批图像对，交给 worker。默认接着做几何验证。若打开 ``guided_matching``，验证之后再做一次引导匹配。最后 ``WriteMatches`` 和 ``WriteTwoViewGeometry``。内点太少时，原始匹配会被清掉。一次运行结束时，还可以做 frame 级别的 ``RigVerification()``。


描述子怎么比
------------

抽象接口是 ``src/colmap/feature/matcher.h`` 里的 ``Match()`` 和 ``MatchGuided()``。``FeatureMatcher::Create()`` 按 ``FeatureMatcherType`` 分发。

``Match()`` 只看描述子。SIFT 用 L2 距离和 ratio test，可选交叉验证。CPU 路径默认走 ``FeatureDescriptorIndex::Search()``，也就是 ``feature/index.cc`` 里的 FAISS IVF；也可以退回暴力点积矩阵。GPU 路径是 ``SiftMatchGPU``。ALIKED 暴力匹配用余弦相似度的 ONNX 核。LightGlue 和 LoMa 的专用网络也是 ONNX，在 ``onnx_matchers.cc`` 和 ``loma.cc``。

``MatchGuided()`` 要求已经有一份两视图几何。它按当前的本质矩阵、基础矩阵或单应矩阵丢掉违反极线或单应约束的描述子对，再重新做 ratio test，结果写进 ``two_view_geometry->inlier_matches``。SIFT 的引导距离分别是：本质矩阵用切向 Sampson，基础矩阵用像素对称极线距离，单应用重投影误差。自动流程里，ALIKED 和 LoMa 没有接上引导匹配。


两视图几何
----------

验证 worker 调用 ``EstimateTwoViewGeometry()``，实现在 ``src/colmap/estimators/two_view_geometry.cc``。配置枚举在 ``scene/two_view_geometry.h``。

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - 配置
     - 含义
   * - ``CALIBRATED``
     - 用本质矩阵。两边焦距都已知时走这里。
   * - ``CALIBRATED_RIG``
     - 已标定 rig 的相对位姿。
   * - ``UNCALIBRATED``
     - 用基础矩阵，或同时估计焦距。
   * - ``PLANAR`` / ``PANORAMIC``
     - 单应矩阵更能解释匹配，场景接近平面或纯旋转。
   * - ``WATERMARK``
     - 边界上的纯平移，当成水印一类退化。
   * - ``DEGENERATE`` / ``MULTIPLE``
     - 几何退化，或一次 RANSAC 里存在多个模型。

非多模型时的分流：

.. code-block:: text

   强制单应
     → EstimateCalibratedHomography
   只有一侧有焦距
     → EstimateOneSidedFocalTwoViewGeometry
   球面相机
     → EstimateSphericalTwoViewGeometry
   同一相机、没有焦距先验
     → EstimateSharedFocalTwoViewGeometry
   两侧都有焦距先验
     → EstimateCalibratedTwoViewGeometry
   非针孔且没有焦距
     → DEGENERATE
   其余
     → EstimateUncalibratedTwoViewGeometry

标定情形下会比较本质矩阵和基础矩阵的内点数，再和单应矩阵比，用来识别平面或纯旋转。基础矩阵可以走 DEGENSAC。可选 Sampson 误差做非线性精化。估计完矩阵后，``EstimateTwoViewGeometryPose()`` 分解出 ``cam2_from_cam1``，并计算三角化角度。

鲁棒估计用的是 ``optim/loransac.h`` 里的 LO-RANSAC，不是每次都从零拟合。最小求解器在 ``estimators/solvers/``：五点本质矩阵、七点和八点基础矩阵、单应、共享焦距或单侧焦距的相对位姿。``optim/sprt.h`` 里的 SPRT 有实现和测试，当前估计器没有调用它。


从匹配到轨迹
------------

``CorrespondenceGraph::AddTwoViewGeometry()`` 只吃 ``inlier_matches``。每条内点写成双向对应，图像对上再记下几何。``Finalize()`` 把邻接表压平，方便后面缓存访问。``DatabaseCache::Load()`` 读库时把所有两视图几何加进这张图。

轨迹本身不在这张图里建。增量三角化时，``IncrementalTriangulator::Find()`` 调用 ``ExtractTransitiveCorrespondences()``，从当前二维点出发做有限深度的搜索，收集多视图上的同一条链。两条互相只指向对方的观测会被 ``IsTwoViewObservation()`` 认出来，选项可以丢掉这种只有两视图的轨迹。
