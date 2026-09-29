特征提取
========

特征提取把每张图变成一组关键点和一条描述子，写进数据库。后面的匹配、检索和重建都只读这些记录，不再回图像上检测。

入口是 ``CreateFeatureExtractorController()``，实现在 ``src/colmap/controllers/feature_extraction.cc``。类型由 ``FeatureExtractionOptions::type`` 决定，工厂在 ``src/colmap/feature/extractor.cc`` 的 ``FeatureExtractor::Create()``。


三条线程
--------

控制器用三级队列，每级容量是 1：

1. ``ImageResizerThread`` 按 ``EffMaxImageSize()`` 做缩略图。默认最大边长：SIFT 为 3200，ALIKED 和 LoMa 为 1600。
2. ``FeatureExtractorThread`` 调用 ``Extract()``。若有重力方向，可先做 ``Rot90``。关键点再缩回相机分辨率，并用相机掩膜和逐图掩膜做 ``MaskFeatures()``。
3. ``FeatureWriterThread`` 写入 SQLite。

写出的内容：

- ``WriteImage``、必要时 ``WritePosePrior`` 和 ``WriteFrame``
- ``WriteKeypoints``：``FeatureKeypoints``，含 x、y 和仿射形状 ``a11`` 到 ``a22``
- ``WriteDescriptors``：描述子矩阵，并带上提取器类型

某张图已经有关键点或描述子时，这一张会被跳过。


SIFT
----

实现在 ``src/colmap/feature/sift.cc`` 的 ``CreateSiftFeatureExtractor()``。选择顺序是：

.. code-block:: text

   需要仿射形状或 domain-size pooling
     → CovariantSiftCPUFeatureExtractor     VLFeat covdet
   否则 use_gpu
     → SiftGPUFeatureExtractor              有 CUDA 走 CUDA，否则走 OpenGL
   否则
     → SiftCPUFeatureExtractor              VLFeat vl_sift

控制器里还有一条覆盖：打开 ``domain_size_pooling`` 或 ``estimate_affine_shape`` 时，SIFT 不用 GPU。

线程：GPU 时每个 ``gpu_index`` 一条提取线程。CPU 时按 ``num_threads`` 起多个提取器，每个内部线程数是 1。

描述子是 128 维 ``uint8``。这是默认提取器，也是数据库里最常见的一种。


ALIKED
------

``src/colmap/feature/aliked.cc`` 里的 ``AlikedFeatureExtractor`` 走 ONNX Runtime。模型是稀疏检测器，输入里带 ``max_keypoints`` 和 ``min_score``。

两种变体：

- ``ALIKED_N16ROT``：更快，训练时考虑了一定的视点变化
- ``ALIKED_N32``：更重，没有专门针对视点变化训练

输入必须是 RGB。描述子是 128 维 ``float32``。权重路径在选项里，给 URL 时会下载并缓存。编译需要 ``ONNX_ENABLED``。


LoMa
----

``src/colmap/feature/loma.cc`` 里的 ``LomaFeatureExtractor`` 用两个 ONNX 模型：检测器 DaD（fp32，两种描述子共用），加上描述子网络。

- ``LOMA_B``：256 维，冻结的 DINOv2 特征加上训练过的卷积特征，对应 DeDoDe-G
- ``LOMA_B128``：128 维，更轻，对应 DeDoDe-B

可以打开 bf16 描述子，也可以用 ``use_fast_resize`` 在滤波缩放和双线性缩放之间切换。同样要求 RGB，并且要 ONNX。CPU 上 LoMa 只用一个提取器，内部再分线程，因为 DeDoDe-G 的权重大约占 1.3 GB。


和后面怎么接
------------

匹配器必须和提取器配套，否则描述子维度和度量对不上。对应关系写在 :doc:`/features`。几何阶段不关心描述子本身，只使用关键点位置；词袋检索的空间验证才会用到尺度和方向。
