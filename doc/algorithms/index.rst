算法总览
========

COLMAP 把一张张照片变成相机位姿、稀疏点、深度图和网格。这一组文档按源码里的调用顺序写，不按菜单项写。函数名和文件路径与仓库一致。

数据从左到右流过这些模块：

.. code-block:: text

   图像
     │  FeatureExtractorController          feature/  controllers/feature_extraction.cc
     ▼
   关键点 + 描述子                         database.db
     │  FeatureMatcherController            controllers/feature_matching.cc
     │  EstimateTwoViewGeometry             estimators/two_view_geometry.cc
     ▼
   内点匹配 + 两视图几何                    correspondence graph
     │
     ├── IncrementalPipeline                sfm/incremental_mapper.cc
     │     初始对 → 注册 → 三角化 → 局部/全局 BA
     │
     └── GlobalPipeline                     sfm/global_mapper.cc
           旋转平均 → 建轨迹 → 全局定位 → BA → 再三角化
     │
     ▼
   稀疏 Reconstruction                      cameras / images / frames / points3D
     │  COLMAPUndistorter                   controllers/undistorters.h
     ▼
   去畸变图像
     │  PatchMatchController                mvs/patch_match_cuda.cu
     │  StereoFusion                        mvs/fusion.cc
     │  Poisson / Delaunay / 推进前沿        mvs/*meshing*
     ▼
   depth / normal / fused.ply / mesh

本仓库在这条链之外还有两条定位路径，见 :doc:`localization`：

- 稀疏定位：SuperPoint + LightGlue 写入数据库，几何仍调用本仓库的 ``colmap`` 二进制。
- 高斯定位：用稀疏模型和去畸变图像训练 3D Gaussian Splatting，再渲染、PnP、光度细化。


运行时的几块数据
----------------

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - 对象
     - 作用
   * - ``Database``
     - SQLite。存 rig、相机、帧、图像、关键点、描述子、原始匹配和两视图几何。接口在 ``src/colmap/scene/database.h``。
   * - ``DatabaseCache``
     - 把数据库读进内存，并建成 ``CorrespondenceGraph``。增量重建和全局重建都从这里开始。
   * - ``CorrespondenceGraph``
     - 只收两视图几何里的内点。每个二维点连到其他图像上的对应点。三角化沿这张图做传递闭包。
   * - ``Reconstruction``
     - 运行中的模型：相机、rig、图像、frame、三维点和 track。
   * - ``Frame`` / ``Rig``
     - 同一时刻的一次采集，以及传感器之间固定的 ``sensor_from_rig``。位姿优化的是 ``rig_from_world``，不是每张图各算一套。
   * - ``PoseGraph``
     - 全局 SfM 的视图图。边上来自两视图几何分解出的相对旋转和平移。


位姿怎么写
----------

代码里的变换一律是 ``target_from_source``。相机位姿是 ``cam_from_world``，rig 位姿是 ``rig_from_world``。像素以角点为原点，左上角像素中心是 ``(0.5, 0.5)``。透视模型先做透视除法，再加畸变，最后乘焦距、加主点。


各页对应的代码
--------------

.. list-table::
   :header-rows: 1
   :widths: 24 76

   * - 文档
     - 先看这些文件
   * - :doc:`features`
     - ``feature/extractor.cc``、``feature/sift.cc``、``feature/aliked.cc``、``feature/loma.cc``、``controllers/feature_extraction.cc``
   * - :doc:`matching`
     - ``controllers/pairing.cc``、``controllers/feature_matching_utils.cc``、``feature/matcher.cc``、``estimators/two_view_geometry.cc``、``scene/correspondence_graph.cc``
   * - :doc:`retrieval`
     - ``retrieval/visual_index.cc``、``retrieval/inverted_file.h``、``retrieval/vote_and_verify.cc``
   * - :doc:`incremental`
     - ``controllers/incremental_pipeline.cc``、``sfm/incremental_mapper.cc``、``sfm/incremental_triangulator.cc``、``sfm/observation_manager.cc``
   * - :doc:`global_sfm`
     - ``controllers/global_pipeline.cc``、``sfm/global_mapper.cc``、``estimators/rotation_averaging.cc``、``estimators/global_positioning.cc``、``controllers/hierarchical_pipeline.cc``
   * - :doc:`bundle_adjustment`
     - ``estimators/bundle_adjustment.h``、``estimators/bundle_adjustment_ceres.cc``、``estimators/cost_functions/reprojection_error.h``
   * - :doc:`mvs`
     - ``controllers/automatic_reconstruction.cc``、``mvs/patch_match_cuda.cu``、``mvs/fusion.cc``
   * - :doc:`localization`
     - ``third_party/Hierarchical-Localization/hloc/colmap_backend.py``、``third_party/gaussian-splatting/train.py``、``localize.py``


一条自动重建实际跑什么
----------------------

``AutomaticReconstructionController`` 把上面的阶段串起来。稀疏阶段结束之后，稠密阶段在 ``RunDenseMapper()`` 里是固定顺序：去畸变，PatchMatch，融合，再按选项做 Poisson、Delaunay 或推进前沿网格。PatchMatch 需要 CUDA。Delaunay 和推进前沿还需要 CGAL。
