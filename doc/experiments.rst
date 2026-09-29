:html_theme.sidebar_secondary.remove: true
:og:description: Gerrard Hall 上增加的 3DGS 自由视角、高斯定位，以及用本仓库 COLMAP 做的稀疏视觉定位。

.. meta::
   :description: Gerrard Hall 上增加的 3DGS 自由视角、高斯定位，以及用本仓库 COLMAP 做的稀疏视觉定位。

增加的功能
==========

这一页记的是在 Gerrard Hall 上加的两条线。稠密重建仍是仓库里的 PatchMatch 和 Poisson。新的部分是：同一套去畸变相机上的 3D Gaussian Splatting，以及稀疏地图上的视觉定位。代码在 ``third_party/`` 里，已经是本仓库的普通文件，克隆 ``feat/3dgs`` 之后不需要 ``git submodule update``\。

照片、训好的高斯和定位产出在 ``data/``，被 git 忽略。换一台机器要自己准备数据和编译。

交互结果开在本机查看器上，文档服务器不托管这些页面：

- 稠密模型：http://127.0.0.1:8899/?scene=gerrard
- 原图和 3DGS 对照：http://127.0.0.1:8899/gerrard_gs.html
- 自由视角：http://127.0.0.1:8899/gerrard_gs_view.html
- 3DGS 定位：http://127.0.0.1:8899/gerrard_loc/
- 稀疏定位：http://127.0.0.1:8899/gerrard_hloc/

查看器是 ``demo/south_building_viewer/`` 里的 Three.js 页面。3DGS 官方的 SIBR 查看器没有放进这棵树，训练和定位也不用它。


3D Gaussian Splatting
---------------------

稠密重建给出表面。3DGS 给出任意相机下的一张图。两边用同一批 ``dense/images`` 和 ``dense/sparse``，不再重跑 SfM。

Gerrard Hall 是 100 张、长边 2000 的 PINHOLE。PatchMatch 约 57 分钟，融合 357 万点，简化后的 Poisson 网格约 34 万面。3DGS 训练 30000 步，约 20 分钟，输入长边收到 1600，结束时约 56.4 万个高斯。训练视角 L1 0.031、PSNR 23.8。

``dense/sparse`` 要放到 3DGS 认的 ``sparse/0``：

.. code-block:: bash

   export COLMAP_ROOT=/home/jesse/workspace/colmap
   export GS_DATA=$COLMAP_ROOT/data/gerrard-hall/gs
   export GS_ROOT=$COLMAP_ROOT/third_party/gaussian-splatting

   mkdir -p "$GS_DATA/sparse/0"
   ln -sfn ../dense/images "$GS_DATA/images"
   for f in cameras.bin images.bin points3D.bin; do
     ln -sfn ../../../dense/sparse/$f "$GS_DATA/sparse/0/$f"
   done

   cd "$GS_ROOT"
   python train.py -s "$GS_DATA" -m "$GS_DATA/output"

``python`` 用 ``~/.conda/envs/pytorch-common/bin/python``\（torch 2.8，CUDA 12.8）。编 ``diff-gaussian-rasterization`` 时把 ``CUDA_HOME`` 指到这个环境。``rasterizer_impl.h``、``forward.h``、``backward.h`` 里已经补了 ``#include <cstdint>``，否则 CUDA 12.8 编不过。

训练视角的对照：

.. code-block:: bash

   python render.py -m "$GS_DATA/output" -s "$GS_DATA" --skip_test

自由视角页面画的是溅射。打开时是清理后的模型：离开稠密表面 20 厘米以上的、贴在表面上但最长轴超过 15 厘米的，以及离房子中心大约 4.5 米以外的，都去掉了。页面上的「原始模型」切回没删过的那一份。

高斯的 ``point_cloud.ply`` 是椭球中心，不是表面。要量尺寸或导进别的软件，用 ``dense/fused.ply`` 或 Poisson 网格。


在高斯里定位
------------

100 张按文件名排序，下标能被 8 整除的 13 张只做查询，其余 87 张训练。训练加 ``--eval``，写到 ``output_eval``，不覆盖全量模型。查集 PSNR 19.8，训练集 23.8。

定位分三段，描述子没有蒸进高斯：

1. 缩略图在 87 张里找最像的一张，用它的位姿当初值。
2. 渲一张颜色和逆深度。查询图和渲染图做 SIFT，用逆深度把点抬回世界坐标，再 PnP。
3. 以 PnP 为起点做 60 步光度收紧（L1 加 SSIM）。光栅器不把梯度传回 view matrix，刚体增量加在高斯上。

.. code-block:: bash

   cd "$GS_ROOT"
   python train.py -s "$GS_DATA" -m "$GS_DATA/output_eval" --eval
   python localize.py -m "$GS_DATA/output_eval" -s "$GS_DATA" --eval

``IMG_2371.JPG`` 检索到 ``IMG_2370.JPG``，初值 5.2°、0.18 米。PnP 722 个内点，1.34°、7.3 厘米。光度收紧后 0.15°、5 毫米。``IMG_2403.JPG`` 匹配不够，PnP 没做，只从检索位姿收到 2.95°、10 厘米。

脚本在 ``third_party/gaussian-splatting/localize.py``\。


稀疏地图上的视觉定位
--------------------

同一批 13 张查询，不用渲染。地图是 87 张的已知位姿，SuperPoint 重新三角化，查询图做 P3P。特征在 hloc 里，几何调用本仓库的 ``build/src/colmap/exe/colmap``，不使用 pip 里的 pycolmap。

入口是 ``scripts/hloc_gerrard.py``\。实现在 ``third_party/Hierarchical-Localization/hloc/colmap_backend.py``\。

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 步骤
     - 子命令
   * - 建库
     - ``database_creator``
   * - 用已知位姿核验重建图匹配
     - ``guided_geometric_verifier``
   * - 在 87 张已知位姿上三角化
     - ``point_triangulator``，``--clear_points 1 --refine_intrinsics 0``
   * - 查询图 P3P
     - ``mapper --input_path``，``--Mapper.fix_existing_frames 1``

查询图没有位姿，引导式核验不会处理它们。查询和 NetVLAD 前 10 张邻居的 LightGlue 匹配写成 ``CALIBRATED`` 双视几何，离群点交给 mapper 里的 RANSAC。重投影阈值 12 像素，至少 15 个内点。SuperPoint 的坐标从像素角点加 0.5，对齐 COLMAP 的像素中心。``feature_importer`` 只认 128 维 SIFT，SuperPoint 不走它。

.. code-block:: bash

   export COLMAP_ROOT=/home/jesse/workspace/colmap
   export GS_DATA=$COLMAP_ROOT/data/gerrard-hall
   export COLMAP_BIN=$COLMAP_ROOT/build/src/colmap/exe/colmap

   ~/.conda/envs/pytorch-common/bin/python "$COLMAP_ROOT/scripts/hloc_gerrard.py" \
     --images "$GS_DATA/dense/images" \
     --reference "$GS_DATA/dense/sparse" \
     --outputs "$GS_DATA/hloc"

局部特征是 ``superpoint_max``\（最多 4096 点，长边 1600），匹配是 ``superpoint+lightglue``，检索是 ``netvlad`` 前 10。建图图像对来自原 SIFT 模型的共视，不用检索去猜。LightGlue 和 NetVLAD 的权重第一次运行时下载，不进 git。SuperPoint 的权重在 ``third_party/Hierarchical-Localization`` 里。

13 张都注册上了。12 张同时落在 0.25 m / 2° 和 5 cm / 5° 里。

.. list-table::
   :header-rows: 1
   :widths: 22 22 28 28

   * - 查询
     - 检索到
     - 检索误差
     - PnP
   * - IMG_2371.JPG
     - IMG_2372.JPG
     - 4.15° / 0.221 m
     - 0.04° / <1 mm，3071 点
   * - IMG_2403.JPG
     - IMG_2402.JPG
     - 3.67° / 0.309 m
     - 0.03° / 1 mm，1697 点
   * - IMG_2387.JPG
     - IMG_2386.JPG
     - 33.74° / 0.030 m
     - 73.66° / 1.728 m，1 点

这个划分是按文件名每隔 8 张抽一张，查询离最近的重建相机大约 12 到 30 厘米。87 张的位姿来自包含查询的那次光束法平差，只是固定住再三角化。所以这个数是和原来的 COLMAP 位姿对齐，不是换一天或换相机的定位。``IMG_2387`` 的最近邻转了 34°，位姿是错的。

``IMG_2371`` 上，3DGS 的 PnP 是 1.34°、7.3 厘米。稀疏这条更紧，点来自 87 张的三角化，不是一张渲染图。``IMG_2403`` 在 3DGS 里匹配不够，稀疏这条解出来了。


克隆之后还要做的事
------------------

``git clone -b feat/3dgs`` 能拿到上面的源码。要跑起来还要：

- 编译 COLMAP。二进制不在 git 里。
- 用带 CUDA 的 PyTorch 编译光栅化器、simple-knn 和 fused-ssim。``build/`` 和 ``.so`` 没有提交。
- 安装 LightGlue。NetVLAD 的模型第一次运行时下载。
- 自己准备 ``data/gerrard-hall``\。照片和模型不在仓库里。
