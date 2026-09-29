定位与高斯
==========

这两条路径都从已经做好的 COLMAP 稀疏模型出发，不再从头做 SfM。稀疏定位换一套学习特征，几何仍交给本仓库的 ``colmap``。高斯把同一套位姿变成可微的辐射场，再用渲染来定位。

.. contents::
   :local:
   :depth: 1


稀疏定位在算什么
----------------

hloc 把「在哪找重叠」和「点怎么对应」从 COLMAP 的 SIFT 里拆出来，三角化和绝对位姿仍用原来的几何。Gerrard 的入口是 ``scripts/hloc_gerrard.py``。参考模型是 ``dense/sparse``，图像是 ``dense/images``。文件名排序后，下标能被 8 整除的是查询，其余是地图。

.. code-block:: text

   地图图像
     SuperPoint → LightGlue（图像对来自参考模型的共视）
     → 写入 database → guided_geometric_verifier → point_triangulator
   查询图像
     NetVLAD 取 10 张地图图 → LightGlue
     → 写成 CALIBRATED 两视图几何 → mapper（已有 frame 固定）

几何后端是 ``third_party/Hierarchical-Localization/hloc/colmap_backend.py``。二进制取 ``COLMAP_BIN``，否则用 ``build/src/colmap/exe/colmap``。不导入 pycolmap。


SuperPoint
----------

实现在 ``third_party/Hierarchical-Localization/third_party/SuperGluePretrainedNetwork/models/superpoint.py``，hloc 的包装是 ``hloc/extractors/superpoint.py``。Gerrard 用的配置名是 ``superpoint_max``：灰度图、长边收到 1600、最多 4096 个点、NMS 半径 3。

网络是一个共享编码器和两个头。编码器是 VGG 风格的卷积，三次池化之后特征图边长是原图的 1/8。

检测头把每个格子预测成 65 类：8×8 个像素位置，再加一个「这里没有点」的 dustbin。对这 65 类做 softmax，丢掉 dustbin，再把 8×8 拼回全分辨率，得到一张分数图。``simple_nms`` 在半径内只留局部最大。分数高于 ``keypoint_threshold``（默认 0.005）的像素成为候选，去掉贴边的点，再按分数取前 ``max_keypoints`` 个。

描述子头在同一张 1/8 特征图上输出 256 维密集描述子，并做 L2 归一化。关键点上的描述子不是再跑一遍网络，而是用 ``grid_sample`` 在这张密集图上双线性插值，然后再归一化一次。hloc 可以换上修正过采样坐标的 ``sample_descriptors_fix_sampling``，``superpoint_max`` 没有打开这个开关。

写进 COLMAP 时，``write_keypoints()`` 把角点坐标加 0.5，对齐本仓库「像素中心在半整数」的约定。


LightGlue
---------

匹配配置是 ``superpoint+lightglue``，包装在 ``hloc/matchers/lightglue.py``。它不在 hloc 仓库里实现网络，而是调用 LightGlue 包，并声明特征类型是 ``superpoint``。默认 ``depth_confidence`` 是 0.95，``width_confidence`` 是 0.99。描述子在送入前做一次转置，凑齐 LightGlue 要的布局。

和 SIFT 的 ratio test 不同，LightGlue 把两张图的关键点当成两个集合，用多层 Transformer 做自注意力和交叉注意力。每一层同时看点的描述子和它在图像上的位置。层数不必跑满：``depth_confidence`` 足够高就提前停止；``width_confidence`` 会丢掉不太可能匹配上的点，后面的层只处理剩下的点。最后一层给出部分指派和每个点的可匹配分数，而不是「最近邻除以次近邻」。

地图阶段的图像对不靠这个网络来选。``covisible_pairs()`` 读参考模型里已经三角化的点，两张地图图只要共享三维点就配成一对。查询阶段才用检索来选对，见下一节。


NetVLAD 选查询的地图图
----------------------

全局描述子在 ``hloc/extractors/netvlad.py``。骨干是 VGG16 的卷积部分，去掉最后的 ReLU 和池化。``NetVLADLayer`` 把每个空间位置的 512 维特征软分配到 ``K = 64`` 个聚类中心：1×1 卷积加 softmax 得到权重，再对「特征减中心」做加权求和。每个簇内部先 L2 归一化，拼成 512×64 维后再整体归一化。默认权重带白化，线性层把它收到 4096 维。权重来自 Pitts30K 的 MATLAB 导出，中心在读入时取了负号，和官方实现的存储符号一致。图像长边收到 1024。

``pairs_from_retrieval.main()`` 把查询和地图的全局描述子做内积。两边都已 L2 归一化，所以这个内积就是余弦相似度。同名图像被遮成负无穷，避免自己配自己。每张查询取相似度最高的 10 张地图图，写成 ``pairs-loc.txt``，再只对这 10 对跑 LightGlue。


几何怎样接回 COLMAP
-------------------

建图在 ``hloc/triangulation.py`` 的 ``triangulation.main()``：

1. ``export_subset_model()`` 把参考模型转到文本，只留地图图像，清空 ``points3D``，再转回二进制。图像 id 保持不变，后面的数据库行才能对上。
2. ``create_database()`` 后，``fill_database_from_model()`` 按同一套 id 写入相机、rig、frame 和图像。
3. ``write_keypoints()`` 和 ``write_matches()`` 把 SuperPoint、LightGlue 的 h5 写进 ``keypoints`` 和 ``matches``。``pair_id`` 用较小的图像 id 在前。
4. ``guided_geometric_verifier`` 用参考模型里已经知道的位姿做引导验证，不重新估计两视图几何。内点写入 ``two_view_geometries``。
5. ``point_triangulator --clear_points 1`` 按这些内点重新三角化。内参精化关闭，焦距留在去畸变后的针孔值上。

查询在 ``hloc/localize_sfm.py`` 的 ``main()``：

1. ``add_query_images()`` 把查询追加到同一个 rig，此时还没有位姿。
2. 查询的 SuperPoint 写入关键点。
3. ``write_query_geometries()`` 把查询和地图的 LightGlue 匹配直接写成 ``config`` 为 ``CALIBRATED`` 的两视图几何。查询没有位姿，不能走引导验证器，所以这里把 LightGlue 的内点当成已经过标定几何的匹配交给 mapper。
4. ``mapper`` 带 ``Mapper.fix_existing_frames 1``。已有 frame 的位姿不动，只注册新图。绝对位姿最大误差 12 像素，最少内点 15，``multiple_models`` 关闭。注册用的就是增量重建里的 P3P 加局部 BA，见 :doc:`incremental`。
5. 局部化模型转成文本，写出 ``poses.txt``。

``QueryLocalizer.localize()`` 仍是旧的 pycolmap 绝对位姿接口，这条 ``main()`` 不调用它。


一个高斯存什么
--------------

训练入口是 ``third_party/gaussian-splatting/train.py`` 的 ``training()``。``Scene`` 经 ``readColmapSceneInfo()`` 读取 ``sparse/0`` 的外参、内参和 ``points3D``，像素来自 ``images/``。相机位姿和内参在训练里固定，被优化的是高斯本身。

``GaussianModel.create_from_pcd()`` 为每个稀疏点建一个高斯，参数是：

- ``_xyz``：点的坐标。
- 球谐系数。最大阶数 ``sh_degree`` 默认是 3，每个颜色通道有 ``(3+1)^2 = 16`` 个系数。0 阶（DC）由点的颜色经 ``RGB2SH`` 填入，其余初始化为 0。DC 和其余系数分成 ``_features_dc``、``_features_rest`` 两块，学习率差 20 倍。
- ``_scaling``：各向同性的初始尺度，取最近邻距离的平方根再取对数。最近邻由 ``distCUDA2`` 计算。存的是对数，前向时再指数化，所以尺度保持为正。
- ``_rotation``：单位四元数 ``(1, 0, 0, 0)``。
- ``_opacity``：存的是 sigmoid 的逆，使初始不透明度等于 0.1。
- 每张训练图还有一个 3×4 的曝光仿射 ``_exposure``，单独一个 Adam。

位置学习率从 ``1.6e-4`` 指数降到 ``1.6e-6``，再乘上场景尺度 ``spatial_lr_scale``。其余学习率见 ``arguments/__init__.py`` 里的 ``OptimizationParams``。


光栅器
------

前向在 ``submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu``。Python 侧是 ``gaussian_renderer.render()``。

每个高斯先做视锥剔除。协方差由尺度和四元数组成世界坐标下的 3×3 矩阵，再用透视投影的雅可比投到屏幕，这是 EWA 溅射。屏幕上的 2×2 协方差求逆，得到圆锥曲线系数。包围半径取最大特征值的 3 倍标准差，用来决定这个高斯盖住哪些 16×16 的图块。颜色在 CUDA 里由球谐沿视线方向算出；若 ``compute_cov3D_python`` 打开，协方差也可以在 Python 里预先算好再传进去。

图块里的高斯按深度排序，从前到后合成。像素上的不透明度是

.. code-block:: text

   alpha = min(0.99, opacity * exp(-0.5 * 二次型))

透过率 ``T`` 从 1 开始。颜色累加 ``color * alpha * T``，逆深度同样用 ``alpha * T`` 加权，然后 ``T`` 乘上 ``(1 - alpha)``。``alpha`` 低于 1/255 的高斯跳过。反向把像素损失链式传回屏幕位置、圆锥曲线、不透明度、颜色和逆深度，再传回三维均值、尺度、旋转和球谐。


训练循环
--------

默认 30000 步。每步从训练相机里无放回地抽一张，抽空再重新洗牌。渲染这张图，和原图比：

.. code-block:: text

   loss = 0.8 * L1 + 0.2 * (1 - SSIM)

``lambda_dssim`` 默认 0.2。若这张相机带有可靠的逆深度图，再加一项随步数衰减的逆深度 L1。Gerrard 的照片没有这张深度图，损失就是上面的颜色项。

球谐不是一开始就用满 3 阶。``active_sh_degree`` 从 0 开始，每 1000 步 ``oneupSHdegree()`` 加一阶，先学一个与方向无关的颜色，再慢慢允许随视线变色。

加密和删除只发生在第 500 步到第 15000 步之间，每 100 步一次。判据是屏幕空间位置梯度的平均模长：``add_densification_stats()`` 把可见高斯的 ``||dL/d(屏幕 xy)||`` 累加，除以被看到的次数。梯度里的 NaN 当成 0。

- 梯度不低于 ``densify_grad_threshold``（默认 ``2e-4``），并且最大尺度不超过场景范围的 ``percent_dense``（默认 0.01）时，``densify_and_clone()`` 复制这个小高斯。它代表一块欠重建的表面，复制一份让优化把它拉开。
- 同样的梯度阈值，但尺度已经大于该范围时，``densify_and_split()`` 把大高斯拆成 ``N = 2`` 个。新的位置在原高斯里按尺度采样，新尺度除以 ``0.8 * N``，原来的那个删掉。
- 不透明度低于 0.005 的删掉。第 3000 步之后，屏幕半径大于 20 像素的，以及世界尺度大于场景范围 0.1 倍的，也删掉。

每 3000 步 ``reset_opacity()`` 把不透明度压到不超过 0.01，再写回 logit。浮在空中、靠高不透明度撑住的高斯会变淡，下一轮若梯度仍大，加密会在正确的位置重新长出来。

位置用普通 Adam，``eps`` 是 ``1e-15``。若装了稀疏 Adam 并且 ``optimizer_type`` 设成 ``sparse_adam``，一步只更新屏幕半径大于 0 的高斯。曝光参数每步单独 ``step``。


高斯定位
--------

``localize.py`` 载入最后一次迭代。测试相机和训练相机的划分在 ``--eval`` 时也是每 8 张留一张，和稀疏定位的 holdout 相同。每张查询做四步。

检索不用 NetVLAD。训练图和查询都缩到高 48、宽 64，按像素 L2 找最近的训练相机，把它的 COLMAP 位姿当作初值。

``render_pose()`` 用这个位姿建 ``MiniCam``：近平面 0.01，远平面 100，投影和视图矩阵相乘后交给同一个 ``render()``。输出是 RGB 和逆深度。这一步只有前向。

``lift_and_pnp()`` 在查询图和渲染图上各提最多 4000 个 SIFT。``BFMatcher`` 用 L2，保留距离小于次近邻 0.75 倍的匹配。渲染图上的关键点去逆深度图里取值。逆深度小于 ``1e-3``，或换成深度后不在 0.3 到 80 之间的点丢掉。剩下的点用检索位姿的焦距和主点反投影到相机坐标，再变到世界坐标，成为 PnP 的三维点。查询图上的像素是二维观测。``solvePnPRansac`` 迭代最多 200 次，重投影阈值是图像宽度的 1% 和 3 像素里的较大者。内点少于 6 视为失败。

``refine()`` 做 60 步 Adam，学习率是旋转 0.002、平移 0.008。旋转增量 ``phi`` 用 ``so3_exp`` 映成矩阵，平移增量是 ``rho``。实现上不把相机位姿放进计算图，而是把全体高斯刚体变到一个单位位姿的相机前：

.. code-block:: text

   xyz  ←  R_delta * (R_init * xyz0 + t_init) + rho
   quat ←  quat(R_delta) * quat(R_init) * quat0

渲染分辨率的宽是 ``REFINE_WIDTH``，查询图双线性缩到同一大小。球谐在这 60 步里降到 0 阶，避免高阶颜色把位姿误差吃掉。损失与训练相同，0.8 的 L1 加 0.2 的 SSIM。梯度因此走光栅器反向，但只更新 ``phi`` 和 ``rho``。``finally`` 里把 ``_xyz``、``_rotation`` 和球谐阶数恢复，下一张查询仍从同一个模型开始。

恢复后的查询位姿是 ``R_cw = R_delta @ R_init``，``t_cw = rho + R_delta @ t_init``。和真值的旋转误差用相对旋转的迹，平移误差用相机中心，中心是 ``-R^T t``。
