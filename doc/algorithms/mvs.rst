多视图立体
==========

稀疏重建给出相机和稀疏点。多视图立体给每张图一张深度图和法线图，再融成稠密点云，可选地做成网格。

自动流程在 ``src/colmap/controllers/automatic_reconstruction.cc`` 的 ``RunDenseMapper()``：

.. code-block:: text

   COLMAPUndistorter
     去畸变图像，并把稀疏模型拷进 dense/{i}/
   PatchMatchController
     photometric，然后通常再跑 geometric
   StereoFusion
     fused.ply 和 fused.ply.vis
   PoissonMeshing 或 DenseDelaunayMeshing 或 AdvancingFrontMeshing

PatchMatch 需要 CUDA。Delaunay 和推进前沿需要 CGAL。去畸变实现是 ``controllers/undistorters.h`` 里的 ``COLMAPUndistorter``。


PatchMatch
----------

CPU 侧的包装是 ``src/colmap/mvs/patch_match.h`` 的 ``PatchMatch`` 和 ``PatchMatchController``。CUDA 调度在 ``mvs/patch_match_cuda.h``，核在 ``mvs/patch_match_cuda.cu``。

一个 ``PatchMatch::Problem`` 指定参考图、源图，以及可选的已有深度图和法线图。第二遍几何一致性就靠这些先验。

核里做的是：

- 光度项是双边加权的归一化互相关。代价定义成 ``1 - NCC``，计算在 ``PhotoConsistencyCostComputer``。
- ``SweepFromTopToBottom`` 按列扫描。消息在相邻像素之间传递，形式是 HMM。源视图按选择概率做蒙特卡洛采样。
- 每个像素的状态是深度加法线。随机扰动用 ``curand``。
- 先验包括三角化角、入射角，以及单应诱导的分辨率。
- ``geom_consistency`` 打开时，再加上相对已有深度和法线的重投影代价，权重是 ``geom_consistency_regularizer``。
- 过滤使用最小 NCC、最小三角化角、几何一致性和多视图一致性掩膜。

写出的文件：

.. code-block:: text

   dense/{i}/stereo/depth_maps/{name}.{photometric|geometric}.bin
   dense/{i}/stereo/normal_maps/{name}.{photometric|geometric}.bin
   dense/{i}/stereo/consistency_graphs/{name}.{photometric|geometric}.bin

常见顺序是先光度，再用光度结果做几何一致性。控制器可以按 GPU 索引把多个 problem 摊到多块卡上，也可以在同一块卡上重复索引来开多条线程。


融合
----

``src/colmap/mvs/fusion.cc`` 的 ``StereoFusion::Fuse()`` 遍历深度图上的像素。一个点要在若干邻近视图里同时满足重投影误差、深度误差和法线夹角，才被收进点云。邻近视图的传递检查走一致性图，张数是 ``check_num_images``。

产物是二进制 PLY ``fused.ply``，以及每个点被哪些图像看到的 ``fused.ply.vis``。写文件的函数是 ``WriteBinaryPlyPoints`` 和 ``WritePointsVisibility``。


网格
----

三种网格互不替代，由自动重建的 mesher 选项选一个：

.. list-table::
   :header-rows: 1
   :widths: 34 66

   * - 函数
     - 实现
   * - ``PoissonMeshing()``
     - ``mvs/poisson_meshing.h``，调用仓库里打包的 PoissonRecon，再做表面裁剪。输出 ``meshed-poisson.ply``。
   * - ``DenseDelaunayMeshing()``
     - ``mvs/delaunay_meshing.h``。CGAL，对应 Labatut 等人 2009 的方法。输出 ``meshed-delaunay.ply``。
   * - ``AdvancingFrontMeshing()``
     - ``mvs/advancing_front_meshing.h``，同样依赖 CGAL。输出 ``meshed-advancing-front.ply``。

网格生成之后，``mesh_texturer`` 可以按投影面积和视角给每个面选一张相机图，烘焙成带逐面 UV 的纹理图集。它的输入是去畸变工作区，不是原始图像。


和稀疏模型的关系
----------------

深度估计用的是去畸变后的针孔图像和对应的稀疏相机，不是原始畸变图。融合和网格都不改相机位姿。位姿若在之后又被 ``mapper`` 或 ``bundle_adjuster`` 改过，稠密结果的坐标系就对不上，需要从头再跑稠密阶段。
