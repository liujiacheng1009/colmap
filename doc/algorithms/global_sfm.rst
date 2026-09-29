全局与分层重建
==============

全局 SfM 不逐张注册。它先在整张视图图上求旋转，再同时求所有相机中心和三维点，最后才做光束法平差和增量式的再三角化。分层重建则是把场景切成重叠的块，每块仍用增量式重建，再自底向上合并。


全局流程
--------

控制器是 ``src/colmap/controllers/global_pipeline.cc`` 的 ``GlobalPipeline``。``Run()`` 读入 ``DatabaseCache``，可选地调用 ``MaybeDecomposeRelativePoses()``，把两视图几何分解成相对位姿。``UNDEFINED``、``DEGENERATE``、``WATERMARK`` 和 ``MULTIPLE`` 会被跳过。

少于一半的相机有焦距先验时，代码会建议先跑 ``view_graph_calibrator``。

``multiple_models`` 打开时走 ``ReconstructMultiComponents()``：

1. 载入位姿图。
2. ``ComputeComponentsByRotationAveraging()`` 先在每个连通分量上做旋转平均，再用相对旋转误差滤边，滤完之后新的连通子图各成一个模型。
3. 每个分量单独 ``GlobalMapper::Solve()``。

单分量就是 ``BeginReconstruction``、``Solve()``、再 ``AlignReconstructionToOrigRigScales()``。最后按已注册 frame 数排序，并给图像提取颜色。


Solve 的五步
------------

``src/colmap/sfm/global_mapper.cc`` 的 ``GlobalMapper::Solve()``：

.. code-block:: text

   1. RotationAveraging
   2. EstablishTracks
   3. GlobalPositioning
   4. IterativeBundleAdjustment
   5. IterativeRetriangulateAndRefine

每一步都可以单独跳过。

旋转平均在 ``estimators/rotation_averaging.*``。``GlobalMapper`` 做两遍：第一遍不过滤未注册视图，按相对旋转误差滤边，第二遍只在留下来的图上再求一次。``RotationEstimator`` 用最小生成树做初值，再做 L1，然后做 IRLS，鲁棒核是 Geman-McClure 或 Half norm。可以加入重力约束，也可以分层求解。未知的 ``cam_from_rig`` 由 ``InitializeRigRotationsFromImages()`` 从图像旋转里初始化。

``EstablishTracks()`` 在位姿图的有效边上用并查集合并匹配。同一张图里若多个点的距离不一致，这条轨迹会被丢掉。然后按每条轨迹的视图数做上下限过滤，按长度从长到短选取，直到达到 ``keep_max_num_tracks`` 和每视图所需轨迹数。此时 ``AddPoint3D`` 只写入 track，坐标要等全局定位来填。

全局定位在 ``estimators/global_positioning.*``。``GlobalPositioner`` 用 Ceres 建立点到相机的约束，优化 frame 中心、三维点和尺度。``refine_sensor_from_rig`` 决定 rig 外参动不动。求完之后按角度误差滤点：没有焦距先验时阈值放宽到两倍，有先验时用严格阈值；再滤三角化角，再用十倍阈值做一次归一化平面上的过滤，然后 ``Normalize()``。

迭代 BA 每一轮可以先只优化平移（旋转固定）。Caspar 不支持固定旋转，这一阶段会强制用 Ceres。接着做完整的联合 BA，默认 Huber 核。``Normalize()`` 之后，归一化重投影阈值按 ``max(3 - iteration, 1)`` 逐轮收紧。最后再用归一化阈值和最小三角化角滤一次。

再三角化会先删掉全部二维观测和三维点，然后对每张已注册图像调用增量三角化器的 ``TriangulateImage()``，接着 ``IterativeGlobalRefinement``，再做两轮 BA 和过滤。这一步把全局方法算出的位姿，重新接回增量三角化使用的对应图。


视图图标定
----------

``src/colmap/estimators/view_graph_calibration.cc`` 的 ``CalibrateViewGraph()`` 在全局 SfM 之前估计焦距。它从 ``UNCALIBRATED`` 和 ``CALIBRATED`` 的图像对读取基础矩阵，用 Ceres 和 ``FetzerFocalLengthCostFunctor`` 优化焦距。``CrossValidatePriorFocalLengths()`` 把通过交叉验证的未标定对提升成 ``CALIBRATED``。标定误差太大的对标成 ``DEGENERATE``。``reestimate_relative_pose`` 打开时会重估本质矩阵和相对位姿。


分层重建
--------

``src/colmap/controllers/hierarchical_pipeline.cc``。注释写明：先把场景分层切成互相重叠的簇，每簇用增量式重建，最后合并。

聚类在 ``scene/scene_clustering.h`` 的 ``SceneClustering``，对场景图做 normalized cut。分支因子是 2，叶子大小和图像重叠由选项控制。

每个叶子簇起一个独立的 ``IncrementalPipeline``，拥有自己的 ``DatabaseCache`` 和 ``ReconstructionManager``。``MergeClusters()`` 从叶子往根递归。一个节点上多于一个重建时，反复调用 ``MergeAndFilterReconstructions()``：先用 Sim3 把源模型对齐并合并进目标，再用重投影阈值 8 像素过滤，最小三角化角在这一步是 0。

对齐函数在 ``estimators/alignment.h``。两模型可以按重投影、投影中心或对应三维点来估计 Sim3。增量重建结束时的地理配准也在这里：``AlignReconstructionToPosePriors()`` 和 ``AlignReconstructionToLocations()`` 用 GPS 或投影中心做 Sim3 RANSAC。

有一个明确的限制：簇内若打开了 ``ba_refine_sensor_from_rig``，不同簇估计出的 rig 外参可能发散，合并容易失败。代码在这种配置下会给出警告。
