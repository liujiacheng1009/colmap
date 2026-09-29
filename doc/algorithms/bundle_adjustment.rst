光束法平差
==========

光束法平差同时调整三维点、相机内参和位姿，使重投影误差变小。增量重建里的局部 BA、全局 BA，以及全局 SfM 的联合 BA，都通过同一个工厂进入。

接口在 ``src/colmap/estimators/bundle_adjustment.h``。``CreateDefaultBundleAdjuster()`` 按选项选择 CERES 或 CASPAR。``CreatePosePriorBundleAdjuster()`` 只有 Ceres 实现。


优化哪些量
----------

``BundleAdjustmentConfig`` 决定这一次问题里有哪些图像、哪些三维点是变量、常数或被忽略，哪些内参固定，以及 ``sensor_from_rig`` 和 ``rig_from_world`` 动不动。规范自由度有两种：``TWO_CAMS_FROM_WORLD`` 和 ``THREE_POINTS``。

``BundleAdjustmentOptions`` 上的开关包括焦距、主点、额外畸变参数、``sensor_from_rig``、``rig_from_world`` 和三维点。``min_track_length`` 以下的轨迹不进问题。``constant_rig_from_world_rotation`` 可以只动平移。

增量重建里两种 BA 的差别：

.. list-table::
   :header-rows: 1
   :widths: 22 39 39

   * - 
     - 局部 ``AdjustLocalBundle``
     - 全局 ``AdjustGlobalBundle``
   * - 图像
     - 参考图加上 ``FindLocalBundle`` 的邻域
     - 全部已注册 frame
   * - 规范自由度
     - 三个点
     - 两个相机；有位姿先验时改走先验 BA
   * - 首轮 loss
     - SOFT_L1，之后改成 TRIVIAL
     - TRIVIAL
   * - 点
     - 变动过且轨迹较短的点，求解后合并、补全、过滤
     - 可选地把冗余点分成两阶段忽略

调用这些函数的循环在 :doc:`incremental`。全局 SfM 里平移阶段和联合阶段的循环在 :doc:`global_sfm`。


Ceres 残差
----------

实现在 ``estimators/bundle_adjustment_ceres.*``。重投影代价在 ``estimators/cost_functions/reprojection_error.h``。

- ``ReprojErrorCostFunctor``：三维点、``rig_from_world`` 和相机一起动。参考传感器用这条。
- ``ReprojErrorConstantPoseCostFunctor``：位姿固定，只动点和内参。
- ``RigReprojErrorCostFunctor``：非参考传感器，多一个 ``sensor_from_rig``。
- ``RigReprojErrorConstantRigCostFunctor``：rig 外参固定。
- ``ScaledRigReprojErrorCostFunctor``：带尺度的 rig 重投影。

这些仿函数可以走解析雅可比，类名是 ``AnalyticalReprojErrorCostFunction``。优化块是三维点、相机内参、7 自由度的 ``rig_from_world``，以及非参考传感器的 ``sensor_from_rig``。

同目录里还有别的代价，给 BA 以外的问题用：

.. list-table::
   :header-rows: 1
   :widths: 34 66

   * - 文件
     - 用途
   * - ``sampson_error.h``
     - Sampson 和切向 Sampson，用于相对位姿。
   * - ``tiny_sampson_error.h``
     - 更轻的 Sampson，用于两视图和焦距求解。
   * - ``calibration.h``
     - 视图图标定里的 Fetzer 焦距代价。
   * - ``pose_prior.h``
     - 绝对位置先验。``AbsolutePosePositionPriorCostFunctor`` 和 Sim3 对齐一起用。
   * - ``motion_averaging.h``
     - 全局定位相关的成对方向约束。
   * - ``alignment.h``
     - 两个点云之间的 ``Point3DAlignmentCostFunctor``。

流形和四元数工具在 ``manifold.h`` 与 ``quaternion_utils.h``。


绝对位姿和相对位姿
------------------

注册新图像时不直接进完整 BA，而是先做一次小的绝对位姿问题。``estimators/pose.h``：

- ``EstimateAbsolutePose``：P3P，可选 P4PF 估计焦距，外面套 RANSAC
- ``RefineAbsolutePose``：Ceres 上的二维到三维精化
- ``EstimateRelativePose``：二维到二维，像素 Sampson 距离的 RANSAC
- ``RefineRelativePose`` 和 ``RefineEssentialMatrix``：相对位姿或本质矩阵的精化

最小求解器在 ``estimators/solvers/``。绝对位姿是 ``P3PEstimator`` 和 ``P4PFEstimator``。广义绝对位姿是 ``GP3PEstimator`` 和 ``GP4PEstimator``，给多相机 rig 和无结构注册用。相对位姿包括五点本质矩阵、七点和八点基础矩阵、单应、共享焦距和单侧焦距。PoseLib 的封装在 ``poselib_utils.*``。

三角化自己的鲁棒估计在 ``IncrementalTriangulator`` 里，用 ``LORANSAC<TriangulationEstimator>``，误差是角度而不是像素。


Caspar
------

``estimators/bundle_adjustment_caspar.*`` 是可选的 GPU 后端。增量流程可以分别给局部 BA 和全局 BA 指定 ``ba_local_backend`` 与 ``ba_global_backend``。

代码里写明的限制：不支持位姿先验，不支持优化 ``sensor_from_rig``，不支持把旋转固定住。全局 SfM 里那个只优化平移的阶段因此会退回 Ceres。使用其他相机模型的观测会被跳过；当前实现针对 ``SIMPLE_RADIAL`` 和 ``PINHOLE``。它也不处理非参考 rig 传感器的 ``sensor_from_rig``，并要求焦距和额外参数的 refine 开关一致。

默认的全局 SfM 后端仍是 Ceres，可以打开 Ceres 自己的 GPU 求解器。那是另一条开关，不是 Caspar。
