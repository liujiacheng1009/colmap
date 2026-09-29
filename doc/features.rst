.. _features:

特征提取与匹配
==============

COLMAP 支持多种特征提取和匹配算法。本页说明如何在命令行或图形界面里切换。


特征提取器类型
--------------

可用的特征提取器：

- ``SIFT``：尺度不变特征变换（默认）。最常用、测试最多。输出 128 维 ``uint8`` 描述子。

- ``ALIKED``：更轻量的学习特征。输出浮点描述子。编译时需要打开 ONNX
  （``-DONNX_ENABLED=ON``）。

- ``LOMA``：LoMa: Local Feature Matching Revisited（ECCV 2026）中的学习特征，
  使用 DeDoDe 结构。两种描述子：``LOMA_B``\（256 维，冻结的 DINOv2 特征加上训练过的卷积特征）
  和 ``LOMA_B128``\（128 维，更轻）。编译时需要打开 ONNX（``-DONNX_ENABLED=ON``）。

命令行选择提取器::

    $ colmap feature_extractor \
        --database_path $DATASET_PATH/database.db \
        --image_path $DATASET_PATH/images \
        --FeatureExtraction.type ALIKED_N16ROT \
        --AlikedExtraction.max_num_features 2048

SIFT 是默认值，可以省略类型，也可以写明::

    $ colmap feature_extractor \
        --database_path $DATASET_PATH/database.db \
        --image_path $DATASET_PATH/images \
        --FeatureExtraction.type SIFT \
        --SiftExtraction.max_num_features 8192

图形界面里打开 ``Processing > Feature extraction``，选好对应标签
（SIFT、ALIKED、LoMa 等）再点 Extract。


特征匹配器类型
--------------

可用的特征匹配器：

- ``SIFT_BRUTEFORCE``：针对 SIFT 的暴力匹配（默认）。使用 L2 距离和 ratio test。

- ``ALIKED_BRUTEFORCE``：针对 ALIKED 的暴力匹配。使用余弦相似度。编译时需要打开 ONNX。

- ``SIFT_LIGHTGLUE``：用 LightGlue 匹配 SIFT 描述子。视点或光照变化大时，匹配数和内点比例通常高于暴力匹配。编译时需要打开 ONNX。

- ``ALIKED_LIGHTGLUE``：用 LightGlue 匹配 ALIKED 描述子。编译时需要打开 ONNX。

- ``LOMA_BRUTEFORCE``：对两种 LoMa 描述子做暴力匹配。使用余弦相似度。编译时需要打开 ONNX。

- ``LOMA_B``：给 256 维 ``LOMA_B`` 描述子用的专用网络匹配器，规模与 LightGlue 相当。编译时需要打开 ONNX。

- ``LOMA_R``：规模与 ``LOMA_B`` 相同，训练时加了旋转增强，对有旋转的图像对更稳。同样匹配 256 维 ``LOMA_B``\。编译时需要打开 ONNX。

- ``LOMA_L``、``LOMA_G``：给 256 维 ``LOMA_B`` 用的更大匹配器，模型更大、匹配质量更高。编译时需要打开 ONNX。

- ``LOMA_B128``：给轻量 128 维 ``LOMA_B128`` 用的专用匹配器。编译时需要打开 ONNX。

命令行选择匹配器::

    $ colmap exhaustive_matcher \
        --database_path $DATASET_PATH/database.db \
        --FeatureMatching.type ALIKED_BRUTEFORCE \
        --AlikedMatching.min_cossim 0.85

SIFT 匹配（默认）::

    $ colmap exhaustive_matcher \
        --database_path $DATASET_PATH/database.db \
        --FeatureMatching.type SIFT_BRUTEFORCE \
        --SiftMatching.max_ratio 0.8

图形界面里打开 ``Processing > Feature matching``，选一种匹配标签
（Exhaustive、Sequential 等），在共用选项的 Type 下拉框里选匹配器。


提取器与匹配器的对应关系
------------------------

提取器和匹配器类型要配套：

- ``SIFT`` 提取配合 ``SIFT_BRUTEFORCE`` 或 ``SIFT_LIGHTGLUE``\。
- ``ALIKED_*`` 提取配合 ``ALIKED_BRUTEFORCE`` 或 ``ALIKED_LIGHTGLUE``\。
- ``LOMA_B`` 提取配合 ``LOMA_BRUTEFORCE``、``LOMA_B``、``LOMA_R``、``LOMA_L`` 或 ``LOMA_G``\。
- ``LOMA_B128`` 提取配合 ``LOMA_BRUTEFORCE`` 或 ``LOMA_B128``\。

类型不配套（例如 SIFT 特征配 ALIKED 匹配器，或 ``LOMA_B128`` 特征配 ``LOMA_B`` 匹配器）会在运行时报错。
同一个数据库里不要混用不同提取器（例如同时有 SIFT 和 ALIKED）。


ALIKED 模型变体
----------------

ALIKED 需要 ONNX 模型文件。几种变体在速度和精度上不同：

- ``aliked-n16rot``：更快，并针对一定的视点变化做过训练。128 维描述子。
- ``aliked-n32``：更贵，没有专门针对视点变化训练。128 维描述子。

用 ``--AlikedExtraction.*_model_path`` 指定模型路径。路径如果是 URL，COLMAP 会自动下载并缓存。
其他 ALIKED 模型见发布页 https://github.com/colmap/colmap/releases


LoMa 模型变体
-------------

LoMa 和 ALIKED 一样，是检测、描述、匹配的一套流程。描述子有两种：

- ``LOMA_B``：256 维，冻结的 DINOv2 特征加上训练过的卷积特征。
- ``LOMA_B128``：更轻的 128 维描述子。

每种描述子有自己的匹配器，速度和精度不同：

- ``LOMA_B`` 可以用 ``LOMA_B``\（最小，与 LightGlue 同规模）、``LOMA_R``\（同规模，训练时加了旋转）、``LOMA_L`` 或 ``LOMA_G``\（依次更大、更慢、更准），也可以用 ``LOMA_BRUTEFORCE``\。
- ``LOMA_B128`` 只能用 ``LOMA_B128`` 或 ``LOMA_BRUTEFORCE``\。最快，在 LoMa 匹配器里精度最低。

默认情况下，第一次使用时会自动下载并缓存全部 LoMa 权重，不用手动准备。
要换模型，提取用 ``--LomaExtraction.*_model_path``，匹配用 ``--LomaMatching.*_model_path``\。
和 ALIKED 一样，路径如果是 URL 会自动下载并缓存。提取和匹配还可以打开 ``use_bf16``
（``--LomaExtraction.use_bf16`` / ``--LomaMatching.use_bf16``）以加快推理。


重复运行
--------

在已有数据库上再次匹配时，已经有双视图几何的图像对会被跳过。
因此只改 ``--FeatureMatching.type`` 不会重做这些图像对。要强制重匹配，先用 ``database_cleaner`` 清掉已有结果::

    $ colmap database_cleaner \
        --database_path $DATASET_PATH/database.db \
        --type two_view_geometries \
        --type matches

换 ``--FeatureExtraction.type`` 重新提取时，再加上 ``--type features``，把已提取的关键点和描述子也清掉。
