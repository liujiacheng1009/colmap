常见问题
========

针对不同重建场景与输出质量调整选项
----------------------------------

COLMAP 提供了许多选项，可针对不同重建场景进行调优，并在精度、完整性与效率之间权衡。默认选项面向中等至高质量的无结构输入数据重建。针对不同场景与质量等级有若干预设，可在 GUI 中通过 ``Extras > Set options for ...`` 设置。若要从命令行使用这些预设，可在选择预设后通过 ``File > Save project`` 保存当前选项集。生成的项目文件可用文本编辑器打开以查看各项选项。也可以通过运行 ``colmap project_generator`` 从命令行生成项目文件。


扩展 COLMAP
-----------

若只需分析 COLMAP 生成的稀疏或稠密重建结果，可在 Python 中使用 pycolmap 加载稀疏模型。

若要编写基于 COLMAP 的 C/C++ 可执行程序，有两种可行方式。其一，COLMAP 的头文件与库默认会安装到 ``CMAKE_INSTALL_PREFIX``\。将 COLMAP 作为库进行编译时，打开 ``BUILD_SHARED_LIBS`` 并安装到 ``CMAKE_INSTALL_PREFIX``\。也可以从 ``src/colmap/tools/example.cc`` 代码模板出发，直接在 COLMAP 内实现所需功能并作为新的二进制程序。


在 SIFT、ALIKED 与 LoMa 特征之间选择
------------------------------------

COLMAP 支持三种特征提取算法：SIFT（默认）、ALIKED 与 LoMa（后两者需要 ONNX 支持）。以下是一些选择建议：

- **SIFT** 是经过最广泛验证、最稳健的选择。适用于视角重叠适中到较高、场景纹理充足、光照条件相近的场景。同时支持 GPU 与 CPU 提取。

- **ALIKED** 是一种学习型特征提取器，在某些情况下可产生更具可重复性的特征，尤其适合视角重叠有限、场景纹理较少以及光照变化剧烈的场景。构建时需要 ONNX Runtime（``-DONNX_ENABLED=ON``）。

- **LoMa** 是一种学习型特征提取器，面向与 ALIKED 相同的困难场景，并配合专用匹配器，以推理开销换取匹配质量。构建时同样需要 ONNX Runtime。

SIFT 与 ALIKED 支持暴力匹配以及基于神经网络的 LightGlue 匹配。LightGlue 通常能得到更高的内点比例，尤其在视点或光照变化较大的图像对上，但需要 ONNX 支持。LoMa 描述子可通过暴力匹配或专用 LoMa 匹配器之一进行匹配。有关可用选项的详细信息，请参见 :ref:`Feature Extraction and Matching <features>`\。

请勿在同一数据库中混用不同特征类型（例如 SIFT 与 ALIKED），因为描述子不兼容。


.. _faq-choosing-camera-model:

选择合适的相机模型
------------------

COLMAP 支持多种参数数量不同的相机模型（完整列表见 :doc:`cameras`）。选择合适的模型取决于镜头类型与重建需求：

- **SIMPLE_RADIAL**\（默认）：大多数标准相机的良好起点。对单个焦距、主点以及一个径向畸变参数建模。

- **PINHOLE**：适用于镜头畸变可忽略的图像（例如已去畸变的图像或高质量工业镜头）。

- **OPENCV**：适用于具有中等畸变的广角镜头。对 2 个焦距、主点以及 4 个畸变参数（2 个径向 + 2 个切向）建模。

- **SIMPLE_RADIAL_FISHEYE** 或 **OPENCV_FISHEYE**：适用于视场角明显大于 120 度的鱼眼镜头。

- **FULL_OPENCV**：仅在有大量图像共享内参、且需要建模复杂畸变模式时使用。该模型有 12 个参数，需要大量观测才能可靠收敛。

经验法则是使用能充分描述镜头的最简单模型。参数过多的过于复杂模型可能导致退化或过拟合的标定，尤其在共享内参的图像较少时。若不确定，可从 ``SIMPLE_RADIAL`` 开始，并在模型统计信息中检查重投影误差。


使用来自 OpenCV、Kalibr 或其他工具的标定
----------------------------------------

若已用 OpenCV 或 Kalibr 等外部工具标定相机，可在 COLMAP 中复用这些内参（参见 :ref:`Fix intrinsics
<faq-fix-intrinsics>` 以在重建期间保持其不变）。首先需要统一两种约定。

**像素坐标约定。** COLMAP 将原点置于图像左上角的 *角点*，因此左上角像素中心为 ``(0.5, 0.5)``，居中主点为 ``(width / 2, height / 2)``\。OpenCV 与 Kalibr 将整数坐标放在像素 *中心*，因此其居中主点为 ``((width - 1) / 2, (height - 1) / 2)``\。将主点从 OpenCV/Kalibr 转换到 COLMAP 时，对 ``cx`` 与 ``cy`` 各加 ``0.5``::

    cx_colmap = cx_opencv + 0.5
    cy_colmap = cy_opencv + 0.5

例如，一张 800×600 图像在 OpenCV 中的居中主点 ``(399.5, 299.5)`` 在 COLMAP 中变为 ``(400.0, 300.0)``\。焦距 ``fx``、``fy`` 以及畸变系数不受该偏移影响。

**畸变参数顺序。** COLMAP 的 ``OPENCV`` 模型使用与 OpenCV 相同的 ``k1, k2, p1, p2`` 畸变参数，``FULL_OPENCV`` 额外按该顺序使用 ``k3, k4, k5, k6``\（见 :doc:`cameras`）。因此上述示例在 ``cameras.txt`` 中的相机行为::

    1 OPENCV 800 600 fx fy 400.0 300.0 k1 k2 p1 p2


在增量、全局与分层 SfM 之间选择
-------------------------------

COLMAP 提供三种 SfM 流水线：

- **Incremental mapper**\（``mapper``，默认）：通过每次增量添加一张图像来重建场景。这是最稳健、经过最充分验证的流水线，但对大型图像集可能变慢，此时反复的光束法平差往往是瓶颈。可使用基于 GPU 的 Caspar 后端显著加速（参见 :ref:`Speedup bundle adjustment <speedup-bundle-adjustment>`）。

- **Global mapper**\（``global_mapper``）：通过旋转平均与全局定位同时求解所有相机位姿。对匹配图良好的大型数据集可能更快，但对匹配中的外点可能不够稳健。全局 mapper 依赖良好的焦距先验。若没有可靠的内参，建议在 ``global_mapper`` 之前运行 ``view_graph_calibrator``，从视图图估计内参（可选，但建议用于提高全局 SfM 质量）。注意 ``view_graph_calibrator`` 会就地修改数据库，因此建议在副本上操作。

- **Hierarchical mapper**\（``hierarchical_mapper``）：将场景划分为重叠的子模型并分别独立重建，再进行合并。适用于增量方法过慢的超大规模数据集，但通常比另外两种流水线稳健性更低。

三者也可通过 ``automatic_reconstructor`` 选择，使用 ``--mapper INCREMENTAL``、``--mapper GLOBAL`` 或 ``--mapper HIERARCHICAL``\。


使用位姿先验（GPS）进行重建
---------------------------

若图像的 EXIF 元数据中包含 GPS 信息，COLMAP 会在特征提取期间自动提取并将其作为位姿先验存入数据库。随后可在重建时通过 ``pose_prior_mapper`` 使用这些先验::

    colmap feature_extractor \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images

    colmap exhaustive_matcher \
        --database_path $PROJECT_PATH/database.db

    colmap pose_prior_mapper \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images \
        --output_path $PROJECT_PATH/sparse

``pose_prior_mapper`` 本质上是启用了先验位置约束的增量 mapper。可使用 ``--overwrite_priors_covariance`` 覆盖先验协方差（不确定性）。新的协方差将根据 ``--prior_position_std_x``、``--prior_position_std_y`` 与 ``--prior_position_std_z`` 的值构建（默认：各 1.0 米）。

若要对已重建模型进行地理配准（映射时未使用先验），请参见 :ref:`地理配准 <geo-registration>` 一节。


.. _faq-share-intrinsics:

共享内参
--------

COLMAP 支持任意图像组与相机模型共享内参。若图像引用同一相机（由数据库中的 ``camera_id`` 属性指定），则它们共享相同内参。可在数据库管理工具中添加新相机并设置共享内参。更多信息请参见 :ref:`Database Management <database-management>`\。


设置已知相机内参
----------------

若相机标定先验已知，推荐在特征提取时通过 ``ImageReader`` 选项提供::

    colmap feature_extractor \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images \
        --ImageReader.single_camera 1 \
        --ImageReader.camera_model OPENCV \
        --ImageReader.camera_params "fx,fy,cx,cy,k1,k2,p1,p2"

参数必须按所选相机模型定义的顺序以逗号分隔列表提供（见 :doc:`cameras`）。在 GUI 中，等效设置位于 ``Processing > Feature extraction > Custom parameters``。若所有图像由同一物理相机以相同设置拍摄，从而在数据库中共享一个相机，请使用 ``--ImageReader.single_camera 1``\（参见 :ref:`共享内参 <faq-share-intrinsics>`）。

要修改已有数据库中的内参，请勿手工编辑 SQLite 表（参数以双精度浮点数的二进制 blob 存储），而应使用 pycolmap 的数据库 API::

    import pycolmap
    with pycolmap.Database.open("path/to/database.db") as db:
        camera = db.read_camera(1)
        camera.params = [fx, fy, cx, cy, k1, k2, p1, p2]
        camera.has_prior_focal_length = True
        db.update_camera(camera)

注意：默认情况下，提供的参数仍会在光束法平差中被优化。若要在重建期间保持固定，请参见 :ref:`Fix intrinsics <faq-fix-intrinsics>`\。


.. _faq-fix-intrinsics:

固定内参
--------

默认情况下，COLMAP 会在重建过程中自动尝试优化相机内参（主点除外）。通常，若数据集中图像足够多且在多张图像间共享内参，SfM 估计的相机内参应优于用标定板手动获得的参数。

然而，COLMAP 的自标定例程有时可能收敛到退化参数，尤其是在具有许多畸变参数的更复杂相机模型中。若先验已知标定参数，可在重建期间固定不同的参数组。选择 ``Reconstruction > Reconstruction options > Bundle Adj. > refine_*``，并勾选要优化或保持不变的参数组。即使在重建期间保持参数不变，仍可在最终全局光束法平差中通过设置 ``Reconstruction > Bundle adj.
options > refine_*``，然后运行 ``Reconstruction > Bundle adjustment`` 来优化这些参数。


主点优化
--------

默认情况下，COLMAP 在重建期间保持主点不变，因为主点估计在一般情况下是病态问题。一旦所有图像都已重建，问题通常已足够约束，可以尝试在全局光束法平差中优化主点，尤其是在多张图像间共享内参时。更多信息请参见 :ref:`Fix intrinsics <faq-fix-intrinsics>`\。


增加匹配数量 / 稀疏三维点数量
-----------------------------

要增加匹配数量，应使用判别力更强的 DSP-SIFT 特征而非普通 SIFT，并使用选项 ``--SiftExtraction.estimate_affine_shape=true`` 与 ``--SiftExtraction.domain_size_pooling=true`` 估计仿射特征形状。此外，应启用引导特征匹配：``--FeatureMatching.guided_matching=true``\。

默认情况下，COLMAP 在三角化时忽略双视图特征轨迹，因此三维点数量少于可能达到的数量。在少数情况下，对双视图轨迹进行三角化可通过在光束法平差中提供额外约束来改善稀疏图像集的稳定性。若也要三角化双视图轨迹，请取消勾选选项 ``Reconstruction > Reconstruction options > Triangulation >
ignore_two_view_tracks``。若图像相对场景拍摄距离较远，可尝试减小最小三角化角度。


从已知相机位姿重建稀疏/稠密模型
-------------------------------

若相机位姿已知，并希望重建场景的稀疏或稠密模型，必须先通过在新文件夹下创建 ``cameras.txt``、``points3D.txt`` 与 ``images.txt`` 来手动构建稀疏模型::

    +── path/to/manually/created/sparse/model
    │   +── cameras.txt
    │   +── images.txt
    │   +── points3D.txt

``points3D.txt`` 文件应为空，且 ``images.txt`` 中每隔一行也应为空，因为稀疏特征将按如下所述计算。有关稀疏模型结构的更多信息，可参见 :ref:`this article <output-format>`\。

images.txt 示例::

    1 0.695104 0.718385 -0.024566 0.012285 -0.046895 0.005253 -0.199664 1 image0001.png
    # Make sure every other line is left empty
    2 0.696445 0.717090 -0.023185 0.014441 -0.041213 0.001928 -0.134851 2 image0002.png

    3 0.697457 0.715925 -0.025383 0.018967 -0.054056 0.008579 -0.378221 1 image0003.png

    4 0.698777 0.714625 -0.023996 0.021129 -0.048184 0.004529 -0.313427 2 image0004.png

上面每张图像的 ``image_id``\（第一列）必须与数据库中的相同（下一步）。可在 GUI 中检查该数据库（位于 ``Database management > Processing``），或者先用 colmap 创建一次重建，再导出为文本以查看其生成的 images.txt 文件。

要重建稀疏地图，首先需对已知相机位姿对应的图像重新计算特征，如下所示::

    colmap feature_extractor \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images

若已知相机内参具有较大的畸变系数，此时应手动将参数从 ``cameras.txt`` 复制到数据库，以便匹配器利用这些内参。修改数据库的方式有很多，一种简便方法是使用 pycolmap 的数据库 API。否则，可跳过此步骤并按如下继续::

    colmap exhaustive_matcher \ # or alternatively any other matcher
        --database_path $PROJECT_PATH/database.db

    colmap point_triangulator \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images
        --input_path path/to/manually/created/sparse/model \
        --output_path path/to/triangulated/sparse/model

注意：从已知相机位姿计算稠密模型并不需要稀疏重建步骤。假设已从已知相机位姿计算出稀疏模型，可按如下方式计算稠密模型::

    colmap image_undistorter \
        --image_path $PROJECT_PATH/images \
        --input_path path/to/triangulated/sparse/model \
        --output_path path/to/dense/workspace

    colmap patch_match_stereo \
        --workspace_path path/to/dense/workspace

    colmap stereo_fusion \
        --workspace_path path/to/dense/workspace \
        --output_path path/to/dense/workspace/fused.ply

也可以不经过稀疏模型直接生成稠密模型，如下所示::

    colmap image_undistorter \
        --image_path $PROJECT_PATH/images \
        --input_path path/to/manually/created/sparse/model \
        --output_path path/to/dense/workspace

由于稠密立体阶段会使用稀疏点云自动选择邻域图像，必须手动指定源图像，如 :ref:`here <faq-dense-manual-source>` 所述。稠密立体阶段此时也需要手动指定深度范围。

最后，在这种情况下，若 min_num_pixels 保持默认值（大于 1），融合将无法成功匹配点。因此还需如下设置该参数::

    colmap patch_match_stereo \
        --workspace_path path/to/dense/workspace \
        --PatchMatchStereo.depth_min $MIN_DEPTH \
        --PatchMatchStereo.depth_max $MAX_DEPTH

    colmap stereo_fusion \
        --workspace_path path/to/dense/workspace \
        --StereoFusion.min_num_pixels 1 \
        --output_path path/to/dense/workspace/fused.ply


.. _faq-merge-models:

合并断开的模型
--------------

有时 COLMAP 无法将所有图像重建到同一模型中，从而产生多个子模型。若这些子模型具有共同的已注册图像，可在后处理步骤中将它们合并为单个模型::

    colmap model_merger \
        --input_path1 /path/to/sub-model1 \
        --input_path2 /path/to/sub-model2 \
        --output_path /path/to/merged-model

为提高两个子模型之间的对齐质量，建议在合并后再次运行全局光束法平差::

    colmap bundle_adjuster \
        --input_path /path/to/merged-model \
        --output_path /path/to/refined-merged-model


.. _geo-registration:

地理配准
--------

通过对部分或全部已注册图像的相机中心提供三维位置，可对模型进行地理配准。重建模型与地理配准目标坐标系之间的三维相似变换由这些对应关系确定。

地理配准的三维坐标既可从数据库（tvec_prior 字段）提取，也可来自用户指定的文本文件。对于文本文件，图像相机中心的地理配准三维坐标必须按以下格式指定::

    image_name1.jpg X1 Y1 Z1
    image_name2.jpg X2 Y2 Z2
    image_name3.jpg X3 Y3 Z3
    ...

坐标可以是基于 GPS 的（lat/lon/alt）或基于笛卡尔坐标的（x/y/z）。若为 GPS 坐标，将转换为笛卡尔坐标。转换可将 GPS 转为 ECEF（地心地固）或 ENU（东-北-天）坐标。若使用 ENU 坐标，第一张图像的 GPS 坐标将定义 ENU 坐标系的原点。也可以使用 ECEF 坐标进行对齐，再将对齐后的重建旋转到 ENU 平面。

注意：估计三维相似变换至少需要指定 3 张图像。然后可使用如下命令对模型进行地理配准::

    colmap model_aligner \
        --input_path /path/to/model \
        --output_path /path/to/geo-registered-model \
        --ref_images_path /path/to/text-file (or --database_path /path/to/database.db) \
        --ref_is_gps 1 \
        --alignment_type ecef \
        --alignment_max_error 3.0 (where 3.0 is the error threshold to be used in RANSAC)

将使用 RANSAC 估计器估计三维相似变换，以对数据中潜在的外点保持稳健。需要提供 RANSAC 估计器使用的误差阈值。

曼哈顿世界对齐
--------------

COLMAP 具备在曼哈顿世界假设下对齐重建坐标轴的功能，即 COLMAP 可通过图像中的消失点检测自动确定重力轴与曼哈顿世界的主水平轴。更多细节请参见 ``model_orientation_aligner``\。


掩膜图像区域
------------

COLMAP 支持在特征提取期间以两种不同方式对关键点进行掩膜：

1. 将 ``mask_path`` 指向包含图像掩膜的文件夹。对于给定图像，对应掩膜在该根目录下的子路径必须与图像在 ``image_path`` 下的子路径相同。文件名必须相同，仅额外增加扩展名 ``.png``\。例如，对于图像 ``image_path/abc/012.jpg``，掩膜应为 ``mask_path/abc/012.jpg.png``\。

2. 将 ``camera_mask_path`` 指向单张掩膜图像。该单一掩膜将应用于所有图像。

在这两种情况下，掩膜图像为黑色（灰度像素强度值为 0）的区域都不会提取特征。


图像方向与 EXIF
---------------

COLMAP 在特征提取期间会自动读取图像的 EXIF 方向标签。该方向会转换为传感器坐标系中的重力方向向量，并作为位姿先验的一部分存入数据库。该重力信息在特征提取与匹配期间用于提高对图像旋转的稳健性。这对于方向不变性有限的特征提取器/匹配器（如 ALIKED、LightGlue 与 LoMa）至关重要。



将新图像注册/定位到已有重建中
-----------------------------

若已有图像重建结果，并希望在该重建中注册/定位新图像，可按以下步骤操作::

    colmap feature_extractor \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images \
        --image_list_path /path/to/image-list.txt

    colmap vocab_tree_matcher \
        --database_path $PROJECT_PATH/database.db \
        --VocabTreeMatching.match_list_path /path/to/image-list.txt

    colmap image_registrator \
        --database_path $PROJECT_PATH/database.db \
        --input_path /path/to/existing-model \
        --output_path /path/to/model-with-new-images

    colmap bundle_adjuster \
        --input_path /path/to/model-with-new-images \
        --output_path /path/to/model-with-new-images

注意：这首先会为新图像提取特征，然后将其与数据库中的已有图像匹配，最后将它们注册到模型中。图像列表文本文件包含要提取与匹配的图像列表，每行指定一个图像文件名。光束法平差是可选的。

若需要更精确的、带三角化的图像注册，则应重新开始或继续重建过程，而不仅仅是将图像注册到模型。不要运行 ``image_registrator``，而应运行 ``mapper``，从已有模型继续重建过程::

    colmap mapper \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images \
        --input_path /path/to/existing-model \
        --output_path /path/to/model-with-new-images

或者，也可以从头开始重建::

    colmap mapper \
        --database_path $PROJECT_PATH/database.db \
        --image_path $PROJECT_PATH/images \
        --output_path /path/to/model-with-new-images

注意：在运行 ``mapper`` 或 ``bundle_adjuster`` 之后，必须从头重新运行稠密重建，因为模型的坐标系在这些步骤中可能会改变。


无 GPU/CUDA 时的可用功能
------------------------

若没有支持 CUDA 的 GPU，但有其他 GPU，则可使用 COLMAP 除稠密重建之外的所有功能。不过，可以使用外部稠密重建软件作为替代，如 :ref:`Tutorial <dense-reconstruction>` 所述。若 GPU 计算能力较低，或想在无外接显示器且无 CUDA 支持的机器上运行 COLMAP，可通过指定相应选项在 CPU 上运行所有步骤（例如特征提取步骤使用 ``--FeatureExtraction.use_gpu=false``）。但请注意，这可能导致重建流水线显著变慢。另外请注意，在默认设置下，对大图像进行 CPU 特征提取可能消耗过多 RAM，可能需要使用 ``--FeatureExtraction.max_image_size`` 手动减小最大图像尺寸，和/或设置 ``--SiftExtraction.first_octave 0``，或使用 ``--FeatureExtraction.num_threads`` 手动限制线程数。


特征提取/匹配中的多 GPU 支持
----------------------------

可通过为支持 CUDA 的 GPU 指定多个索引，在多个 GPU 上运行特征提取/匹配，例如 ``--FeatureExtraction.gpu_index=0,1,2,3`` 与 ``--FeatureMatching.gpu_index=0,1,2,3`` 可在 4 个 GPU 上并行运行特征提取/匹配。注意每个 GPU 只能运行一个线程，这通常也能获得最佳性能。默认情况下，COLMAP 为每个支持 CUDA 的 GPU 运行一个特征提取/匹配线程，与在同一 GPU 上运行多个线程相比，这通常能获得最佳性能。


因非法内存访问导致特征匹配失败
------------------------------

若遇到以下错误信息::

    MultiplyDescriptor: an illegal memory access was encountered

或以下信息：

    ERROR: Feature matching failed. This probably caused by insufficient GPU
    memory. Consider reducing the maximum number of features.

在特征匹配期间出现上述情况，说明 GPU 内存不足。请尝试减小选项 ``--FeatureMatching.max_num_matches``，直到错误消失。注意这可能导致特征匹配结果变差，因为为将特征装入 GPU 内存，较低尺度的输入特征会被截断。也可以改用基于 CPU 的特征匹配，但可能非常慢，或者更好的做法是购买内存更大的 GPU。

所需最大 GPU 内存可近似用以下公式估计：对 SIFT 为 ``4 * num_matches * num_matches + 4 * num_matches * 256``\。例如，若设置 ``--FeatureMatching.max_num_matches 10000``，所需最大 GPU 内存约为 400MB，且仅当某张图像实际具有如此多特征时才会分配。


.. _speedup-bundle-adjustment:

加速光束法平差
--------------

以下介绍减少光束法平差运行时间的实用方法。

- **减小问题规模**

  限制对应点数量，使 BA 求解更小的问题：

  - 通过减小 ``--SiftExtraction.max_image_size`` 和/或 ``--SiftExtraction.max_num_features`` 减少特征。
  - 通过减小 ``--SequentialMatching.overlap``、``--SpatialMatching.max_num_neighbors`` 或 ``--VocabTreeMatching.num_images`` 减少匹配对（并尽可能避免使用 ``exhaustive_matcher``）。
  - 通过减小 ``--FeatureMatching.max_num_matches`` 减少匹配。
  - 使用 ``--Mapper.ba_global_ignore_redundant_points3D 1`` 启用实验性路标剪枝以丢弃冗余三维点。

- **利用 GPU 加速**

  通过为 ``mapper`` 设置 ``--Mapper.ba_use_gpu 1``，并为独立的 ``bundle_adjuster`` 设置 ``--BundleAdjustmentCeres.use_gpu 1``，启用基于 GPU 的 Ceres 求解器进行光束法平差。若干参数控制何时以及使用哪种 GPU 求解器：

  - 仅当图像数量超过 ``--BundleAdjustmentCeres.min_num_images_gpu_solver`` 时才会激活 GPU 求解器。
  - 使用 ``--BundleAdjustmentCeres.max_num_images_direct_dense_gpu_solver`` 与 ``--BundleAdjustmentCeres.max_num_images_direct_sparse_gpu_solver`` 在直接稠密、直接稀疏与迭代稀疏 GPU 求解器之间选择

  .. Attention:: 在 Ceres 2.3 正式发布之前，COLMAP 官方启用 CUDA 的二进制发行版
     不附带 ceres[cuda]。要使用 GPU 求解器，必须编译带有 CUDA/cuDSS 支持的 Ceres，
     并将其链接到 COLMAP。

  **注意：** 当 Schur 补矩阵稀疏性降低（即出现更多填充）时，基于 Schur 的稀疏求解器
  （cuDSS）可能出现 GPU 利用率偏低。常见原因包括：

  - 图像共视程度高
  - 共享相机内参。

- **使用 Caspar GPU 光束法平差后端**

  COLMAP 包含 Caspar [caspar]_，一种实验性的 GPU 加速光束法平差后端，对于中等至大规模问题可比 Ceres CUDA 求解器快一到两个数量级，尤其能为增量 mapper 带来显著加速。Caspar 需要 CUDA，默认禁用；必须在构建时通过 ``-DCASPAR_ENABLED=ON`` 配置 COLMAP 来启用。

  Caspar 通过光束法平差的 ``backend`` 选项选择，该选项接受 ``CERES``\（默认）或 ``CASPAR``：

  - 独立的 ``bundle_adjuster``：``--BundleAdjustment.backend CASPAR``\。
  - 增量 ``mapper``：``--Mapper.ba_local_backend CASPAR`` 和/或 ``--Mapper.ba_global_backend CASPAR``\。GPU 设备通过 ``--Mapper.ba_gpu_index`` 选择。

  独立后端的求解器行为与 GPU 设备可通过 ``--BundleAdjustmentCaspar.*`` 选项调优，例如 ``--BundleAdjustmentCaspar.gpu_index``\（默认 ``-1`` 会自动选择最佳 CUDA 设备）。

  .. Attention:: Caspar 为实验性功能，目前仅支持 ``SIMPLE_RADIAL`` 与
     ``PINHOLE`` 相机模型；使用其他相机模型的观测会被跳过。它不支持位姿先验，
     也不支持对非参考 rig 传感器优化 ``sensor_from_rig``，并要求
     ``refine_focal_length`` 与 ``refine_extra_params`` 相等。
     ``global_mapper`` 未提供 Caspar 后端选择器。

- **其他实用建议**

  - 通过调整观测过滤参数，使 BA 获得更多内点、更少外点，或提供准确先验（例如内参、位姿）来改善初始条件。
  - 在可能时固定或限制参数优化（例如在内参已知时保持固定），以减少优化变量数量。
  - 通过减少 LM 迭代次数或放宽收敛容差，以少量精度换取运行时间：``--Mapper.ba_global_max_num_iterations``、``--Mapper.ba_global_function_tolerance``\。
  - 使用 mapper 选项降低昂贵全局 BA 的频率：``--Mapper.ba_global_frames_freq``、``--Mapper.ba_global_points_freq``、``--Mapper.ba_global_frames_ratio`` 与 ``--Mapper.ba_global_points_ratio``\。


在稠密重建中权衡完整性与精度
----------------------------

若稠密点云包含过多外点与噪声，请尝试增大选项 ``--StereoFusion.min_num_pixels`` 的值。

若使用 Poisson 重建得到的稠密表面网格模型没有表面，或存在过多外点表面，应减小选项 ``--PoissonMeshing.trim`` 的值以减小表面积，反之则增大。也可考虑在融合阶段按上文所述减少外点或提高完整性。

若使用 Delaunay 重建得到的稠密表面网格模型过于嘈杂或不完整，应增大 ``--DelaunayMeshing.quality_regularization`` 参数以获得更平滑的表面。若网格分辨率过粗，应将 ``--DelaunayMeshing.max_proj_dist`` 选项减小到更低的值。


改善弱纹理表面的稠密重建结果
----------------------------

对于弱纹理表面的场景，使用高分辨率输入图像（``--PatchMatchStereo.max_image_size``）与较大的 patch 窗口半径（``--PatchMatchStereo.window_radius``）会有帮助。也可降低光度一致性代价的过滤阈值（``--PatchMatchStereo.filter_min_ncc``）。


表面网格重建
------------

COLMAP 支持三种表面重建算法：

- **Poisson surface reconstruction** [kazhdan2013]_ 通常需要几乎无外点的输入点云，在存在外点或输入数据有大孔洞时往往会产生较差的表面。

- 基于 **Delaunay triangulation** 的网格化对外点更稳健，且通常比 Poisson 算法更能扩展到大型数据集，但生成的表面通常不够平滑。它可应用于稀疏与稠密重建结果。

- **Advancing front surface reconstruction** [cohen-steiner2004]_ 从输入点的 Delaunay 三角剖分增量生长表面网格。它支持基于可见性的过滤以移除违反自由空间约束的面，并支持面向大规模场景的分块并行处理。它使用 float32 CGAL 内核以提高内存效率。

作为后处理步骤以提高表面平滑度，可使用拉普拉斯平滑，例如 Meshlab 中的实现。

注意：也可以将 Poisson 与 Delaunay 网格化结合：先运行 Delaunay 网格化以稳健地从稀疏或稠密点云中过滤外点，然后在第二步执行 Poisson 表面重建以获得平滑表面。

网格化之后，可使用 ``mesh_texturer`` 命令生成带纹理图集的纹理网格 [waechter2014]_。该方法根据投影面积与视角将每个网格面分配给最佳视角的相机图像，并将纹理烘焙到带有逐面 UV 坐标的图集中。该命令需要 ``image_undistorter`` 生成的去畸变工作区作为输入。


加速稠密重建
------------

稠密重建可通过多种方式加速：

- 在系统中增加更多 GPU，因为稠密重建可在立体重建步骤中使用多个 GPU。在系统中增加更多 RAM，并将 ``--PatchMatchStereo.cache_size``、``--StereoFusion.cache_size`` 增大到尽可能大的值，以加速稠密融合步骤。

- 不执行几何稠密立体重建 ``--PatchMatchStereo.geom_consistency false``\。此时请确保同时启用 ``--PatchMatchStereo.filter true``\。

- 减小 ``--PatchMatchStereo.max_image_size``、``--StereoFusion.max_image_size`` 的值，以在最大图像分辨率下进行稠密重建。

- 减少每张参考图像考虑的源图像数量，如 :ref:`here <faq-dense-memory>` 所述。

- 将 patch 窗口步长 ``--PatchMatchStereo.window_step`` 增大到 2。

- 减小 patch 窗口半径 ``--PatchMatchStereo.window_radius``\。

- 减少 patch match 迭代次数 ``--PatchMatchStereo.num_iterations``\。

- 减少采样视图数量 ``--PatchMatchStereo.num_samples``\。

- 对于非常大的重建，要加速稠密立体与融合步骤，可使用 CMVS 将场景划分为多个簇并剪枝冗余图像，如 :ref:`here <faq-dense-memory>` 所述。

注意：除升级硬件外，上述改动可能会降低稠密重建结果的质量。若取消立体重建过程并稍后重启，之前的进度不会丢失，已处理的视图将被跳过。


.. _faq-dense-memory:

减少稠密重建期间的内存占用
--------------------------

若在 patch match 立体期间 GPU 内存不足，可通过设置选项 ``--PatchMatchStereo.max_image_size`` 减小最大图像尺寸，或将 ``stereo/patch-match.cfg`` 文件中的源图像数量从例如 ``__auto__, 30`` 减少到 ``__auto__, 10``\。注意启用 ``geom_consistency`` 选项会增加所需的 GPU 内存。

若在立体或融合期间 CPU 内存不足，可减小以 GB 为单位指定的 ``--PatchMatchStereo.cache_size`` 或 ``--StereoFusion.cache_size``，或减小 ``--PatchMatchStereo.max_image_size`` 或 ``--StereoFusion.max_image_size``\。注意过低的值可能导致处理非常缓慢并对硬盘造成沉重负载。

对于数千张图像的大规模重建，应考虑使用例如 CMVS [furukawa10]_ 将稀疏重建拆分为更易管理的图像簇。此外，CMVS 允许剪枝观测同一场景元素的冗余图像。注意：对于这种用例，从命令行执行时，COLMAP 的稠密重建流水线也支持 PMVS/CMVS 文件夹结构。请参考工作区文件夹中的示例 shell 脚本。注意：仅当输出类型设置为 PMVS 时，才会生成 PMVS/CMVS 的示例 shell 脚本。由于 CMVS 会产生高度重叠的簇，建议根据可用系统资源与速度要求，将每簇默认的 100 张图像尽可能提高。要使用 CMVS 更改图像数量，必须相应修改 shell 脚本。例如，``cmvs pmvs/ 500`` 可将每个簇限制为 500 张图像。若只想用 CMVS 剪枝冗余图像而不对场景分簇，可将该数字设为非常大的值。


.. _faq-dense-manual-source:

稠密重建期间手动指定源图像
--------------------------

可将 ``stereo/patch-match.cfg`` 文件中的源图像数量从例如 ``__auto__, 30`` 改为 ``__auto__, 10``\。这会自动选择视觉重叠最多的图像作为源图像。也可以通过指定 ``__all__`` 使用所有其他图像作为源图像。或者，可手动按名称指定图像，例如::

    image1.jpg
    image2.jpg, image3.jpg
    image2.jpg
    image1.jpg, image3.jpg
    image3.jpg
    image1.jpg, image2.jpg

这里，``image2.jpg`` 与 ``image3.jpg`` 用作 ``image1.jpg`` 的源图像，依此类推。


稠密重建中的多 GPU 支持
-----------------------

可通过为支持 CUDA 的 GPU 指定多个索引，在多个 GPU 上运行稠密重建，例如 ``--PatchMatchStereo.gpu_index=0,1,2,3`` 可在 4 个 GPU 上并行运行稠密重建。也可通过重复指定同一 GPU 索引，在同一 GPU 上运行多个稠密重建线程，例如 ``--PatchMatchStereo.gpu_index=0,0,1,1,2,3``\。默认情况下，COLMAP 为每个支持 CUDA 的 GPU 运行一个稠密重建线程。


.. _faq-dense-timeout:

修复稠密重建期间的 GPU 冻结与超时
---------------------------------

立体重建流水线使用 CUDA 在 GPU 上运行，并使 GPU 处于高负载状态。重建期间可能会遇到显示冻结甚至程序崩溃。解决该问题的一种方法是在系统中使用未连接到显示器的第二块 GPU，并通过显式设置 GPU 索引（通常索引 0 对应连接显示器的显卡）。也可以增大系统的 GPU 超时，详见下文。

默认情况下，Windows 操作系统会检测 GPU 的响应问题，并通过重置显卡并中止立体重建过程来恢复可用的桌面。解决方法是将所谓的 "Timeout Detection & Recovery"（TDR）延迟增大到更大的值。请参考 `NVIDIA Nsight documentation <https://goo.gl/UWKVs6>`_ 或 `Microsoft
documentation <http://www.microsoft.com/whdc/device/display/wddm_timeout.mspx>`_ 了解如何在 Windows 下增大延迟时间。可通过以下 Windows 注册表项增大延迟::

    [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\GraphicsDrivers]
    "TdrLevel"=dword:00000001
    "TdrDelay"=dword:00000120

要设置注册表项，请以管理员权限执行以下命令（例如在 ``cmd.exe`` 或 ``powershell.exe`` 中）::

    reg add HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\GraphicsDrivers /v TdrLevel /t REG_DWORD /d 00000001
    reg add HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\GraphicsDrivers /v TdrDelay /t REG_DWORD /d 00000120

然后重启机器以使更改生效。

Linux/Unix 下的 X 窗口系统有类似功能，也会检测 GPU 的响应问题。避免 X 窗口系统超时问题的最简单方法是关闭它，并从命令行运行立体重建。在 Ubuntu 下，可先使用以下命令停止 X::

    sudo service lightdm stop

然后从命令行运行稠密重建代码::

    colmap patch_match_stereo ...

最后，可用以下命令重启桌面环境::

    sudo service lightdm start

若在这些更改后稠密重建仍然崩溃，原因可能是 GPU 内存不足，如本列表中另一项所述。
