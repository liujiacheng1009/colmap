增量式重建
==========

增量式 SfM 从一对图像长出模型：先固定一个初始相对位姿并三角化，再一张一张把新图像注册进来，每次注册后三角化、做局部光束法平差，隔一段时间做一次全局光束法平差。

外层控制器是 ``src/colmap/controllers/incremental_pipeline.cc`` 的 ``IncrementalPipeline``。单次重建的引擎是 ``src/colmap/sfm/incremental_mapper.cc`` 的 ``IncrementalMapper``，选图和初始对的细节在 ``incremental_mapper_impl.*``。


外层怎么循环
------------

``Run()`` 先建 ``DatabaseCache``。小数据集会放宽 ``min_model_size``。然后进入 ``Reconstruct()``。

同一张场景图上可以长出多个子模型，次数由 ``init_num_trials`` 控制。若仍有图像没注册，再放松两轮：先把初始内点数阈值减半，再把初始三角化角阈值减半，各再跑一次 ``Reconstruct()``。

一个子模型在 ``ReconstructSubModel()`` 里：

.. code-block:: text

   BeginReconstruction
   若还没有已注册 frame
     InitializeReconstruction
   循环，直到连续两次都注册失败
     FindNextImages
     RegisterNextImage 或 RegisterNextStructureLessImage
     对该 frame 里每张图 TriangulateImage
     IterativeLocalRefinement
     若 CheckRunGlobalRefinement
       IterativeGlobalRefinement
       FilterFrames
   必要时再做一次最终的全局精化
   EndReconstruction

全局 BA 的触发看已注册 frame 数或三维点数是否跨过 ``ba_global_*_ratio`` 或 ``ba_global_*_freq``。子模型成功结束时，``AlignReconstructionToPriorsOrRigScale()`` 先尝试对齐位姿先验，失败再恢复 rig 的原始尺度。


初始像对
--------

``InitializeReconstruction()`` 的顺序：

1. ``FindInitialImagePair()``。用户指定了图像对时，改为直接 ``EstimateInitialTwoViewGeometry()``。
2. ``RegisterInitialImagePair()``：第一帧相机是单位变换，第二帧写成估计出的 ``cam2_from_cam1``，两帧都 ``RegisterFrame()``。
3. 对两帧里的每张图 ``TriangulateImage()``。这里的最小三角化角用的是 ``init_min_tri_angle``。
4. ``AdjustGlobalBundle()``，然后 ``Normalize()``。
5. ``FilterPoints()`` 和 ``FilterFrames()``。

初始对的搜索在 ``IncrementalMapperImpl``。``FindFirstInitialImage()`` 按焦距先验和对应数量排序。对每个候选，并行地 ``FindSecondInitialImage()`` 再估计两视图几何。留下一对需要同时满足：内点数不少于 ``init_min_num_inliers``，平移的 z 分量绝对值小于 ``init_max_forward_motion``（丢掉纯向前运动），三角化角大于 ``init_min_tri_angle``。多于一台相机的 rig 会走 ``EstimateInitialGeneralizedTwoViewGeometry()``。估计出的焦距可以通过 ``SeedEstimatedInitialCameras()`` 写回。

点数为 0、没有已注册 frame，或者点数小于绝对位姿所需的最少内点时，这次初始化失败。


下一张图
--------

``FindNextImages()`` 只考虑还没注册、可见三维点数达到 ``abs_pose_min_num_inliers``、并且尝试次数没超过 ``max_reg_trials`` 的图像。还没被滤掉、也没重试过的图像排在前面。

默认排序是 ``MIN_UNCERTAINTY``，分数来自 ``Point3DVisibilityScore``，也就是可见性金字塔。另外两种是可见三维点的绝对数量，以及可见三维点占该图全部观测的比例。无结构注册时改按可见对应数排序。

``RegisterNextImage()`` 是带结构的注册，用的是二维到三维：

1. 多传感器 rig 且焦距可靠时，改走 ``RegisterNextGeneralFrame()``，求解器是 ``GP3PEstimator``，接口在 ``estimators/generalized_pose.*``。
2. 否则收集该图二维点通过对应图连到的已三角化三维点。
3. ``EstimateAbsolutePose()`` 做 P3P RANSAC，需要时用 P4PF 同时估计焦距。实现在 ``estimators/pose.h`` 和 ``estimators/solvers/absolute_pose.*``。
4. ``RefineAbsolutePose()`` 用 Ceres 再修一次。
5. 写入 ``cam_from_world``，``RegisterFrame()``，内点调用 ``AddObservation()``，并把被改过的三维点交给三角化器。

可见三维点不够时，``RegisterNextStructureLessImage()`` 作为退路。注释写明它来自 Zheng 和 Wu 的 structure-less resection。它至少需要两张已注册图像，内点阈值是普通绝对位姿的两倍。它收集的是已注册相机之间的二维到二维对应，调用 ``EstimateStructureLessAbsolutePose()``。已有三维点的对应只延长轨迹；没有的则当场 ``EstimateTriangulation()`` 新建点。最后用一个很小的 BA 修这张新位姿。


三角化
------

``src/colmap/sfm/incremental_triangulator.cc`` 的头文件说明三角化包含三件事：新建点、延长已有点、以及一张图把两条轨迹桥接起来时的合并。

``TriangulateImage()`` 对每个二维点先 ``Find()``，按 ``max_transitivity`` 收集传递对应。一个都还没三角化就 ``Create()``。已经有三角化对应时先 ``Continue()``，剩下的未三角化观测再尝试 ``Create()``。

- ``Create()`` 至少要两个未三角化观测。``ignore_two_view_tracks`` 打开时跳过只有两视图的观测。``EstimateTriangulation()`` 使用角度误差，阈值是 ``create_max_angle_error``。外点多时会递归再试一次。
- ``Continue()`` 在已有三维点里选角度重投影误差最小的那个。误差不超过 ``continue_max_angle_error`` 才 ``AddObservation()``。
- ``Merge()`` 沿对应找到另一个三维点，位置取加权平均。轨迹上每一个元素的重投影误差都小于 ``merge_max_reproj_error`` 才合并，并可递归。
- ``Complete()`` 从轨迹出发做广度优先，传递深度从 1 到 ``complete_max_transitivity``，重投影误差足够小就补观测。
- ``Retriangulate()`` 处理漂移。图像对上已三角化对应占全部对应的比例低于 ``re_min_ratio`` 时，这对被视为重建不足，最多重试 ``re_max_trials`` 次。这里会 Continue 和 Create，但两个观测已经属于不同三维点时不做 Merge。

局部 BA 结束时对变动过的点做 Merge 和 Complete。全局精化一开始先 ``CompleteAllTracks`` 加 ``MergeAllTracks``，再 ``Retriangulate()``。


局部 BA 和全局 BA
-----------------

``IterativeLocalRefinement()`` 反复调用 ``AdjustLocalBundle()``，直到变动观测占参与观测的比例低于 ``max_refinement_change``。第二轮起 Ceres 的 loss 换成 TRIVIAL。

局部邻域由 ``FindLocalBundle()`` 选出：和当前图像共享三维点最多的若干张图，张数是 ``ba_local_num_images - 1``。共享点数排序之后，会逐步放宽三角化角的 75 分位和最少共享观测；仍然不够就用重叠最大的图像填满。

``AdjustLocalBundle()`` 的配置：

- 规范自由度用三个点固定，``FixGauge(THREE_POINTS)``
- 加入参考图和局部邻域里所有 frame 的图像
- 遵守 ``fix_existing_frames``、固定 rig、固定相机这些开关
- 只把 track 长度不超过 15、或还没有误差记录的变动三维点设成变量

求解之后 ``MergeTracks``、``CompleteTracks``、``CompleteImage``，再过滤这些图像里的三维点和被改过的点。

``IterativeGlobalRefinement()`` 先补全并合并轨迹、再三角化，然后循环 ``AdjustGlobalBundle()``。全局配置覆盖所有已注册 frame。没有位姿先验时，规范自由度用两个相机固定，注释里写这比固定三个点更稳、步数更少。大模型可以先找出冗余三维点，分两阶段忽略它们。小模型则收紧 Ceres 的收敛条件。每轮之后可以 ``Normalize()``，再补全、合并、过滤。收敛看变动观测占全部观测的比例。

代价函数和后端的差别见 :doc:`bundle_adjustment`。


过滤
----

统计和删除都在 ``src/colmap/sfm/observation_manager.cc``。它维护每张图的观测数、可见对应数、可见三维点数和可见性金字塔，也维护图像对上已三角化对应和全部对应的计数，供 ``Retriangulate()`` 使用。

``FilterAllPoints3D()`` 先删大重投影误差，再删三角化角太小的点。注释强调这个顺序：先删大误差，避免一个坏点因为角度很大而被留下。重投影误差可以按像素、归一化平面或角度来量。轨迹短于 2 就删掉整个点，否则只删外点观测。负深度过滤会删掉落在非球面相机后方的观测。

``FilterFrames()`` 只在已注册 frame 不少于 20 时运行。``FindFramesToFilter()`` 找出内参明显不对、或三维点少到低于最少观测数的 frame，然后 ``DeRegisterFrame()``。注销会减掉可见对应，并删掉该 frame 的全部三维观测。
