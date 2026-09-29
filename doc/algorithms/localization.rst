定位与高斯
==========

这两条路径都坐在已经做好的 COLMAP 稀疏模型上，不再从头做 SfM。稀疏定位重新提取学习特征，几何仍调用本仓库编译出的 ``colmap``。高斯定位用同一套位姿训练辐射场，再靠渲染定位。Gerrard Hall 上的数字和页面在 :doc:`/experiments`。


稀疏定位
--------

几何后端是 ``third_party/Hierarchical-Localization/hloc/colmap_backend.py``。``colmap_binary()`` 取环境变量 ``COLMAP_BIN``，否则用本仓库的 ``build/src/colmap/exe/colmap``。所有几何步骤都是 ``run_colmap()``，不导入 pycolmap。

Gerrard 的入口是 ``scripts/hloc_gerrard.py``。参考模型是 ``dense/sparse``，图像是 ``dense/images``。按文件名排序后，下标能被 8 整除的是查询，其余是地图。

建图阶段由 ``hloc/triangulation.py`` 的 ``triangulation.main()`` 串起来：

.. code-block:: text

   export_subset_model
     model_converter 到文本，按图像名过滤，清空 points3D，再转回二进制
     图像 id 与参考模型保持一致
   create_database
   fill_database_from_model
     按同一套 id 写入 cameras、rigs、frames、frame_data、images
   write_keypoints
     SuperPoint 的 h5 写入 keypoints，坐标加 0.5，对齐 COLMAP 的像素中心
   write_matches
     LightGlue 的 h5 写入 matches，pair_id 用较小 id 在前的编码
   guided_geometric_verification
     colmap guided_geometric_verifier
     用参考模型的已知位姿做引导，不重新估计两视图几何
   triangulate
     colmap point_triangulator --clear_points 1
     内参精化关闭

图像对不是检索猜的。``covisible_pairs()`` 从参考模型里已经三角化的共视关系取出地图图像对。特征是 ``superpoint_max``，匹配是 ``superpoint+lightglue``。

查询阶段在 ``hloc/localize_sfm.py`` 的 ``main()``：

1. 从建图数据库读出现有的名字到 id。
2. ``add_query_images()`` 把还没有位姿的查询图追加到同一个 rig。
3. ``write_keypoints()`` 写入查询图的 SuperPoint。
4. ``write_query_geometries()`` 把查询和地图的 LightGlue 匹配直接写成 ``config`` 为 ``CALIBRATED`` 的两视图几何。这一步故意绕过引导验证器，因为查询还没有位姿。
5. ``register_queries()`` 调用 ``colmap mapper``，带上 ``Mapper.fix_existing_frames 1``，所以已有 frame 的位姿不动。绝对位姿的最大误差和最少内点分别是 12 像素和 15。``multiple_models`` 关掉。
6. 再把局部化模型转成文本，写出 ``poses.txt``。

查询图像对来自 NetVLAD：``pairs_from_retrieval`` 给每张查询取 10 张地图图，再跑 LightGlue。

``QueryLocalizer.localize()`` 仍是旧的 pycolmap 绝对位姿接口，``main()`` 不走它。


高斯训练
--------

入口是 ``third_party/gaussian-splatting/train.py`` 的 ``training()``。

``Scene`` 通过 ``readColmapSceneInfo()`` 读 ``sparse/0`` 的外参、内参和 ``points3D``，像素来自 ``images/``。``GaussianModel.create_from_pcd()`` 用这些稀疏点做初始高斯。位姿和内参在训练中固定。

每一轮：随机取一张训练相机，``gaussian_renderer.render()`` 得到 RGB 和逆深度，损失是 L1 加上一项 SSIM。反向传播之后，在 densify 截止步之前做 ``densify_and_prune`` 和透明度重置，再 ``optimizer.step()``。

``--eval`` 时 ``llffhold=8``，排序后每 8 张留一张做测试。这和稀疏定位的 holdout 是同一种分法。

渲染器是子模块 ``submodules/diff-gaussian-rasterization``。Python 侧 ``GaussianRasterizer.forward()`` 调 CUDA：

- ``FORWARD::preprocess`` 把三维均值投到屏幕，由尺度和旋转（或预先算好的协方差）得到二维圆锥曲线，用球谐算出颜色，并做视锥剔除。
- ``FORWARD::render`` 按图块排序做 alpha 合成，输出 RGB、逆深度和每个高斯的屏幕半径。
- ``BACKWARD::render`` 把像素损失传回二维位置、圆锥曲线、透明度、颜色和逆深度。
- ``BACKWARD::preprocess`` 再链式传到三维均值、尺度、旋转、球谐和透明度。

训练靠这条反向更新高斯参数。


高斯定位
--------

``third_party/gaussian-splatting/localize.py`` 的 ``main()`` 载入最后一次迭代。每张测试相机单独做四步：

1. 检索。把训练图和查询图缩到 48×64，按像素 L2 找最近的训练相机，把它的位姿当作初值。
2. 在这个位姿上 ``render_pose()``。内部建 ``MiniCam``，调用同一个 ``render()``，得到 RGB 和逆深度。
3. ``lift_and_pnp()``。查询图和渲染图都提 SIFT，``BFMatcher`` 用 0.75 的 ratio。渲染图上的关键点到逆深度图里取值，用检索位姿反投影到世界坐标，再 ``cv2.solvePnPRansac``。这一步只有前向渲染，梯度不进光栅器。
4. ``refine()`` 做 60 步 Adam。优化的是李代数 ``phi`` 和位移 ``rho``，但实现方式是去改高斯的 ``_xyz`` 和 ``_rotation``，相机固定在单位位姿上渲染。损失是 0.8 的 L1 加 0.2 的 SSIM。``finally`` 里把高斯恢复原状，所以下一次查询仍从同一模型开始。

因此光度细化的梯度走的是光栅器反向，但视图矩阵本身没有参数。等价的效果是把点云刚体变到查询相机前。
