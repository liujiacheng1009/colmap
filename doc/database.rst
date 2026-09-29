.. _database-format:

数据库格式
==========

COLMAP 把提取出的信息存在一个 SQLite 数据库里。可以用图形界面里的数据库工具、
C++ 数据库 API（``src/colmap/scene/database.h``），或 pycolmap 访问。

数据库包含这些表：

- rigs
- cameras
- frames
- images
- keypoints
- descriptors
- matches
- two_view_geometries

要建一个带所需表结构的空库，可以在图形界面新建工程，或运行 ``colmap database_creator``\。


Rig 与传感器
------------

rig 与传感器（相机等）是 1 对 N。其中一个传感器被选为参考，定义 rig 的原点。
每个传感器只能属于一个 rig。


Rig 与 Frame
------------

rig 与 frame 是 1 对 N。一个 frame 是该 rig 在某一时刻的一次实例，
可以包含全部传感器，也可以只包含其中一部分，它们在同一时刻曝光。


相机与图像
----------

相机与图像是 1 对 N。这对运动恢复结构很重要：同一相机共享内参
（焦距、主点、畸变等），每张图像有自己的外参（朝向和位置）。

相机内参以连续的 ``float64`` 二进制块存储，顺序见 ``src/colmap/sensor/models.h``\。
COLMAP 只用被图像引用的相机，其余相机被忽略。

images 表的 ``name`` 列是图像文件夹内的唯一相对路径。因此数据库文件和图像文件夹可以挪到别处，
只要相对目录结构不变。

手工插入图像和相机时，标识必须为正且非零，即 ``image_id > 0`` 且 ``camera_id > 0``\。


关键点与描述子
--------------

检测到的关键点按行主序存成 ``float32`` 二进制块。前两列是图像中的 X、Y。
COLMAP 约定图像左上角坐标为 ``(0, 0)``，左上角像素中心为 ``(0.5, 0.5)``\。
若有 4 列，特征几何是相似变换，第三列是尺度，第四列是方向（按 SIFT 约定）。
若有 6 列，特征几何是仿射，后 4 列是仿射形状（见 ``src/colmap/feature/types.h``）。

描述子按行主序存成二进制块，每一行对应该关键点的外观。数据类型和维度取决于提取器：

- **SIFT**：``uint8``，128 维（每个特征 128 字节）。
- **ALIKED**：``float32``，128 维（每个特征 512 字节）。
- **LOMA_B**：``float32``，256 维（每个特征 1024 字节）。
- **LOMA_B128**：``float32``，128 维（每个特征 512 字节）。

descriptors 表的 ``cols`` 列是每行描述子的字节数。``uint8`` 时等于维度，
``float32`` 时等于 ``4 * 维度``\。

两张表的 ``rows`` 是该图像检测到的特征数，``rows=0`` 表示没有特征。
做特征匹配和几何验证时，每张图像都要有对应的关键点和描述子记录。
只有带快速空间验证的词袋匹配才需要有意义的局部特征几何，
也就是至少要有 X 和 Y，其余关键点列可以填 0。重建流程的其余部分只用关键点位置。


匹配与双视图几何
----------------

特征匹配的结果在 ``matches`` 表，几何验证的结果在 ``two_view_geometries`` 表。
重建只用 ``two_view_geometries``\。两条记录都表示两张不同图像之间的特征匹配。
``pair_id`` 是上三角匹配矩阵的行主序线性下标，生成方式如下::

    def image_ids_to_pair_id(image_id1, image_id2):
        if image_id1 > image_id2:
            return 2147483647 * image_id2 + image_id1
        else:
            return 2147483647 * image_id1 + image_id2

从 ``pair_id`` 可以唯一还原图像标识::

    def pair_id_to_image_ids(pair_id):
        image_id2 = pair_id % 2147483647
        image_id1 = (pair_id - image_id2) / 2147483647
        return image_id1, image_id2

``pair_id`` 让查询更高效，因为匹配表可能有数亿行。这个编码把数据库中的图像数上限定为
2147483647（有符号 32 位整数的最大值），即 ``image_id`` 必须小于 2147483647。

匹配表里的二进制块是行主序 ``uint32`` 矩阵。左列是 ``image_id1`` 特征的从 0 开始的下标，
右列是 ``image_id2`` 的下标。``cols`` 必须为 2，``rows`` 是匹配条数。

``two_view_geometries`` 表中的 F、E、H 以 3×3、行主序 ``float64`` 存储。
``config`` 的含义见 ``src/colmap/estimators/two_view_geometry.h``\。
