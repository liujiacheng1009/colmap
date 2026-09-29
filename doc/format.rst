.. _output-format:

模型格式
========

================
二进制文件格式
================

请注意，所有二进制数据均以小端（little endian）字节序存储。所有 x86
处理器均为小端，因此在大多数平台上读取 COLMAP 二进制数据时无需特殊处理。
解析这些数据最便捷的方式是使用 ``src/colmap/scene/reconstruction_io.h``
下的 C++ reconstruction API，或使用 pycolmap 提供的 Python API。


==============
索引与标识符
==============

任何以 ``*_idx`` 结尾的变量名应视为有序、连续的从零开始的索引。一般而言，
任何以 ``*_id`` 结尾的变量名应视为无序、非连续的标识符。

例如，相机（``CAMERA_ID``）、图像（``IMAGE_ID``）和三维点（``POINT3D_ID``）
的唯一标识符是无序的，且很可能不连续。这也意味着最大的 ``POINT3D_ID``
并不一定等于三维点的数量，因为在重建过程中的过滤等操作会导致部分
``POINT3D_ID`` 缺失。


========
稀疏重建
========

默认情况下，COLMAP 使用二进制文件格式（机器可读、速度快）存储稀疏模型。
此外，COLMAP 也提供将稀疏模型存储为文本文件（人类可读、速度慢）的选项。
在这两种情况下，信息都会拆分到多个文件中，分别包含 ``rigs``、``cameras``、
``frames``、``images`` 和 ``points`` 的信息。任何包含这些文件的目录都构成一个
稀疏模型。二进制文件的扩展名为 ``.bin``，文本文件的扩展名为 ``.txt``\。
请注意，当从同时包含二进制和文本文件的目录加载模型时，COLMAP 会优先使用
二进制格式。

请注意，较旧版本的 COLMAP 不支持 rig，因此 ``rigs`` 和 ``frames`` 文件可能
缺失。COLMAP 中的重建 I/O 例程完全向后兼容：可以读取没有这些文件的模型，
并自动初始化平凡的 rig 和 frame。此外，较新输出重建中的 ``cameras`` 和
``images`` 文件也与旧输出完全兼容。

要在 GUI 中导出当前选中的模型，请选择 ``File > Export
model``。要导出当前数据集中所有已重建的模型，请选择
``File > Export all``\。所选文件夹随后会包含模型文件，并且为方便起见，
还会包含当前项目配置，以便将模型导入 COLMAP。要导入已导出的模型（例如用于
可视化或继续重建），请选择 ``File > Import model``，并选择包含
``rigs``、``cameras``、``frames``、``images`` 和 ``points3D`` 文件的文件夹。

要在 GUI 中在二进制格式与文本格式之间转换，可以使用 ``File > Import model``
加载模型，然后使用 ``File > Export model``\（二进制）或 ``File > Export model as text``
（文本）以所需输出格式导出模型。此外，还可以使用 ``File > Export as...``
将稀疏模型导出为其他格式，例如 VisualSfM 的 NVM、Bundler 文件、PLY、VRML 等。
要从 CLI 在各种格式之间转换，请使用 ``model_converter``
可执行程序。

稀疏重建可以方便地通过 Python 中的 pycolmap 读取。


--------
文本格式
--------

COLMAP 会为每个重建模型导出以下文本文件：
``rigs.txt``、``cameras.txt``、``frames.txt``、``images.txt`` 和 ``points3D.txt``\。
注释以前导 "#" 字符开头，会被忽略。开头的注释行会简要描述文本文件的格式，
本页有更详细的说明。


rigs.txt
-----------

该文件包含已配置的 rig 和传感器，例如::

    # Rig calib list with one line of data per calib:
    #   RIG_ID, NUM_SENSORS, REF_SENSOR_TYPE, REF_SENSOR_ID, SENSORS[] as (SENSOR_TYPE, SENSOR_ID, HAS_POSE, [QW, QX, QY, QZ, TX, TY, TZ])
    # Number of rigs: 1
    1 2 CAMERA 1 CAMERA 2 1 -0.9999701516465348 -0.0011120266840749639 -0.0075347911527510894 0.0012985125893421306 -0.19316906391350164 0.00085222218993398979 0.0070758955539026785
    2 1 CAMERA 3

这里，数据集包含两个 rig：第一个 rig 有两台相机，第二个
有 1 台相机。


cameras.txt
-----------

该文件包含数据集中所有已重建相机的内参，每台相机一行，例如::

    # Camera list with one line of data per camera:
    #   CAMERA_ID, MODEL, WIDTH, HEIGHT, PARAMS[]
    # Number of cameras: 3
    1 SIMPLE_PINHOLE 3072 2304 2559.81 1536 1152
    2 PINHOLE 3072 2304 2560.56 2560.56 1536 1152
    3 SIMPLE_RADIAL 3072 2304 2559.69 1536 1152 -0.0218531

这里，数据集包含 3 台基于不同畸变模型的相机，
传感器尺寸相同（宽度：3072，高度：2304）。参数长度是可变的，
取决于相机模型。对于第一台相机，有 3 个参数，单一焦距为 2559.81 像素，
主点位于像素位置 ``(1536, 1152)``\。一台相机的内参可由多张图像共享，
这些图像通过唯一标识符 ``CAMERA_ID`` 引用相机。


frames.txt
----------

该文件包含 frame，其中 frame 定义了某个
rig 在某一时刻的具体实例，此时所有或一部分传感器同时曝光，例如::

    # Frame list with one line of data per frame:
    #   FRAME_ID, RIG_ID, RIG_FROM_WORLD[QW, QX, QY, QZ, TX, TY, TZ], NUM_DATA_IDS, DATA_IDS[] as (SENSOR_TYPE, SENSOR_ID, DATA_ID)
    # Number of frames: 151
    1 1 0.99801363919752195 0.040985139360073107 0.041890917712361225 -0.023111584553400576 -5.2666546897987896 -0.17120007823690631 0.12300519697527648 2 CAMERA 1 1 CAMERA 2 2
    2 2 0.99816472047267968 0.037605501383281774 0.043101511724657163 -0.019881568259519072 -5.1956060695789192 -0.20794508616745555 0.14967533910764824 1 CAMERA 3 3

这里，数据集包含两个 frame，其中 frame 1 是 rig 1 的一个实例，
frame 2 是 rig 2 的一个实例。


images.txt
----------

该文件包含数据集中所有已重建图像的位姿和关键点，
每张图像两行，例如::

    # Image list with two lines of data per image:
    #   IMAGE_ID, QW, QX, QY, QZ, TX, TY, TZ, CAMERA_ID, NAME
    #   POINTS2D[] as (X, Y, POINT3D_ID)
    # Number of images: 2, mean observations per image: 2
    1 0.851773 0.0165051 0.503764 -0.142941 -0.737434 1.02973 3.74354 1 P1180141.JPG
    2362.39 248.498 58396 1784.7 268.254 59027 1784.7 268.254 -1
    2 0.851773 0.0165051 0.503764 -0.142941 -0.737434 1.02973 3.74354 1 P1180142.JPG
    1190.83 663.957 23056 1258.77 640.354 59070

这里，前两行定义第一张图像的信息，以此类推。
图像的重建位姿指定为从世界坐标系到该图像相机坐标系的投影，
使用四元数 ``(QW, QX, QY, QZ)`` 和平移向量 ``(TX, TY, TZ)``\。四元数采用
Hamilton 约定，例如 Eigen 库也使用该约定。投影中心/相机中心的坐标由
``-R^t * T`` 给出，其中 ``R^t`` 是由四元数构成的 3x3 旋转矩阵的逆/转置，
``T`` 是平移向量。图像的局部相机坐标系定义为：从图像看去，X 轴指向右，
Y 轴指向下，Z 轴指向前。

上述示例中的两张图像使用相同的相机模型并共享内参
（``CAMERA_ID = 1``）。图像名称相对于项目所选的基础图像文件夹。
第一张图像有 3 个关键点，第二张图像有 2 个关键点，关键点位置以像素坐标指定。
两张图像都观测到 2 个三维点；请注意，第一张图像的最后一个关键点没有观测到
重建中的三维点，因为其三维点标识符为 -1。


points3D.txt
------------

该文件包含数据集中所有已重建三维点的信息，
每个点一行，例如::

    # 3D point list with one line of data per point:
    #   POINT3D_ID, X, Y, Z, R, G, B, ERROR, TRACK[] as (IMAGE_ID, POINT2D_IDX)
    # Number of points: 3, mean track length: 3.3334
    63390 1.67241 0.292931 0.609726 115 121 122 1.33927 16 6542 15 7345 6 6714 14 7227
    63376 2.01848 0.108877 -0.0260841 102 209 250 1.73449 16 6519 15 7322 14 7212 8 3991
    63371 1.71102 0.28566 0.53475 245 251 249 0.612829 118 4140 117 4473

这里有三个已重建的三维点，其中 ``POINT2D_IDX`` 定义了
``images.txt`` 文件中关键点的从零开始的索引。误差以重投影误差的像素为单位给出，
并且仅在全局光束法平差之后更新。


========
稠密重建
========

COLMAP 使用如下工作区文件夹结构::

    +── images
    │   +── image1.jpg
    │   +── image2.jpg
    │   +── ...
    +── sparse
    │   +── cameras.txt
    │   +── images.txt
    │   +── points3D.txt
    +── stereo
    │   +── consistency_graphs
    │   │   +── image1.jpg.photometric.bin
    │   │   +── image2.jpg.photometric.bin
    │   │   +── ...
    │   +── depth_maps
    │   │   +── image1.jpg.photometric.bin
    │   │   +── image2.jpg.photometric.bin
    │   │   +── ...
    │   +── normal_maps
    │   │   +── image1.jpg.photometric.bin
    │   │   +── image2.jpg.photometric.bin
    │   │   +── ...
    │   +── patch-match.cfg
    │   +── fusion.cfg
    +── fused.ply
    +── meshed-poisson.ply
    +── meshed-delaunay.ply
    +── textured
    │   +── mesh.ply
    │   +── texture.png
    +── run-colmap-geometric.sh
    +── run-colmap-photometric.sh

这里，``images`` 文件夹包含去畸变后的图像，``sparse`` 文件夹包含带有
去畸变相机的稀疏重建，``stereo`` 文件夹包含立体重建结果，``fused.ply``、
``meshed-poisson.ply`` 和 ``meshed-delaunay.ply`` 是融合与网格化过程的结果，
``textured`` 文件夹包含由 ``mesh_texturer`` 生成的带纹理网格（带有每面 UV
坐标的 ``mesh.ply`` 以及纹理图集 ``texture.png``），而 ``run-colmap-geometric.sh``
和 ``run-colmap-photometric.sh`` 包含执行稠密重建的示例命令行用法。


--------------
深度图与法线图
--------------

深度图以混合的文本和二进制文件存储。文本头以格式 ``width&height&channels&``
定义图像尺寸，随后是按行主序（row-major）排列的 ``float32`` 二进制数据。
对于深度图，``channels=1``；对于法线图，``channels=3``\。深度图和法线图可以
方便地用 Python 通过 pycolmap 读取。


----------
一致性图
----------

一致性图为图像中所有像素定义了该像素与哪些源图像一致。该图以混合的文本和
二进制文件存储，其中文本部分与深度图和法线图等价，二进制部分是连续的
``int32`` 值列表，格式为
``<row><col><N><image_idx1>...<image_idxN>``\。这里，``(row, col)`` 定义像素在
图像中的位置，随后是 ``N`` 个图像索引的列表。这些索引相对于
``images.txt`` 文件中的顺序指定。
