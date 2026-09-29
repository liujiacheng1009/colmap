.. _cli:

命令行
======

命令行界面提供对 COLMAP 全部功能的访问，便于自动化脚本。每项核心功能都以
``colmap`` 可执行文件的一条命令实现。运行 ``colmap -h`` 可列出可用命令（在
Windows 下为 ``COLMAP.bat -h``）。注意，若从 CMake 构建目录运行 COLMAP，可执行文件位于
``./src/colmap/exe/colmap``\。要启动图形用户界面，请运行 ``colmap gui``\。

示例
----

假设你的项目图像按如下结构存放::

    /path/to/project/...
    +── images
    │   +── image1.jpg
    │   +── image2.jpg
    │   +── ...
    │   +── imageN.jpg

自动重建工具的命令如下::

    # The project folder must contain a folder "images" with all the images.
    $ DATASET_PATH=/path/to/project

    $ colmap automatic_reconstructor \
        --workspace_path $DATASET_PATH \
        --image_path $DATASET_PATH/images

注意，任意命令都可通过 ``-h,--help`` 命令行参数列出全部可用选项。若需要对重建过程的各项参数有更多控制，可执行以下命令序列，作为自动重建命令的替代::

    # The project folder must contain a folder "images" with all the images.
    $ DATASET_PATH=/path/to/dataset

    $ colmap feature_extractor \
       --database_path $DATASET_PATH/database.db \
       --image_path $DATASET_PATH/images

    $ colmap exhaustive_matcher \
       --database_path $DATASET_PATH/database.db

    $ mkdir -p $DATASET_PATH/sparse

    $ colmap mapper \
        --database_path $DATASET_PATH/database.db \
        --image_path $DATASET_PATH/images \
        --output_path $DATASET_PATH/sparse

    $ mkdir -p $DATASET_PATH/dense

    $ colmap image_undistorter \
        --image_path $DATASET_PATH/images \
        --input_path $DATASET_PATH/sparse/0 \
        --output_path $DATASET_PATH/dense \
        --output_type COLMAP \
        --max_image_size 2000

    $ colmap patch_match_stereo \
        --workspace_path $DATASET_PATH/dense \
        --workspace_format COLMAP \
        --PatchMatchStereo.geom_consistency true

    $ colmap stereo_fusion \
        --workspace_path $DATASET_PATH/dense \
        --workspace_format COLMAP \
        --input_type geometric \
        --output_path $DATASET_PATH/dense/fused.ply

    $ colmap poisson_mesher \
        --input_path $DATASET_PATH/dense/fused.ply \
        --output_path $DATASET_PATH/dense/meshed-poisson.ply

    $ colmap delaunay_mesher \
        --input_path $DATASET_PATH/dense \
        --output_path $DATASET_PATH/dense/meshed-delaunay.ply

    $ colmap advancing_front_mesher \
        --input_path $DATASET_PATH/dense \
        --output_path $DATASET_PATH/dense/meshed-advancing-front.ply

    # Optionally simplify a dense mesh to reduce its size.
    $ colmap mesh_simplifier \
        --input_path $DATASET_PATH/dense/meshed-poisson.ply \
        --output_path $DATASET_PATH/dense/meshed-poisson-simplified.ply \
        --MeshSimplification.target_face_ratio 0.25

    # Optionally texture a mesh using the undistorted images.
    $ colmap mesh_texturer \
        --workspace_path $DATASET_PATH/dense \
        --input_path $DATASET_PATH/dense/meshed-poisson.ply \
        --output_path $DATASET_PATH/dense/textured

优雅关闭与恢复
--------------

特征提取与匹配命令、``mapper``、
``pose_prior_mapper``、``bundle_adjuster``、``point_triangulator``、
``image_registrator``、图像去畸变相关命令、
``patch_match_stereo`` 以及 ``stereo_fusion`` 会协作处理 ``SIGINT`` 与 ``SIGTERM``\。
``automatic_reconstructor`` 在使用增量式 mapper 时支持优雅关闭。第一次信号会在安全点停止工作，
并写出任何可用的进行中结果。随后进程以 ``SIGINT`` 对应状态码 130、或以 ``SIGTERM`` 对应状态码 143 退出。
第二次信号会立即终止，并可能打断正在进行的输出写入。

特征提取与匹配可通过对同一数据库重新运行相同命令来恢复。PatchMatch 同样可对同一工作区重新运行；
已完成的深度图与法线图会被跳过。被中断的增量式重建会写入其常规编号模型目录，
并可用 ``--input_path`` 继续::

    $ colmap mapper \
        --database_path $DATASET_PATH/database.db \
        --image_path $DATASET_PATH/images \
        --input_path $DATASET_PATH/sparse/0 \
        --output_path $DATASET_PATH/sparse/0

点三角化、图像注册与光束法平差会在退出前写出可用的部分结果。立体融合会将非空的部分结果写入包含
``.partial`` 的同级路径，以免覆盖已完成的结果。被中断的图像去畸变可通过重新运行相同命令重启。
Ceres 优化在迭代之间停止。Caspar 光束法平差后端只能在当前求解器调用结束后停止。全局 mapper
不支持优雅关闭，因为其中间重建目前无法恢复。

等效的 pycolmap 函数会自动处理 ``SIGINT``\（Ctrl-C），执行优雅关闭并抛出 ``KeyboardInterrupt``\。
对于其他终止事件，它们接受可选的 ``cancellation_token``\。显式取消会完成清理，然后抛出
``InterruptedError``\。这使应用程序能将云抢占信号（如 ``SIGTERM``）连接到该 token，而不会让
pycolmap 替换宿主应用的信号处理器。自动 ``SIGINT`` 处理要求从 Python 主线程调用 pycolmap 函数；
从其他线程调用时请使用 cancellation token::

    import signal
    import pycolmap

    token = pycolmap.CancellationToken()
    signal.signal(signal.SIGTERM, lambda *_: token.cancel())
    pycolmap.incremental_mapping(
        database_path,
        image_path,
        output_path,
        cancellation_token=token,
    )

若要使用全局 SfM 流程而非增量式 mapper，请将 ``mapper`` 步骤替换为 ``global_mapper``\。
全局 mapper 依赖良好的焦距先验；若没有可靠的内参（例如来自 EXIF 或实验室标定），应先运行
``view_graph_calibrator``\。该步骤为可选，但建议执行以提高全局 SfM 质量，这也是
`GLOMAP <https://github.com/colmap/glomap>`_ 中一贯的默认做法。注意
``view_graph_calibrator`` 会就地修改数据库中的相机内参与双视图几何，因此建议在数据库副本上操作::

    $ colmap feature_extractor \
       --database_path $DATASET_PATH/database.db \
       --image_path $DATASET_PATH/images

    $ colmap exhaustive_matcher \
       --database_path $DATASET_PATH/database.db

    # Optional but often needed: calibrate intrinsics from the view graph.
    # This modifies the database in-place, so work on a copy.
    $ cp $DATASET_PATH/database.db $DATASET_PATH/database_global.db
    $ colmap view_graph_calibrator \
        --database_path $DATASET_PATH/database_global.db

    $ mkdir -p $DATASET_PATH/sparse

    $ colmap global_mapper \
        --database_path $DATASET_PATH/database_global.db \
        --image_path $DATASET_PATH/images \
        --output_path $DATASET_PATH/sparse

若要在无显示器的计算机上运行 COLMAP（例如集群或云服务），COLMAP 会在系统支持时自动切换使用 CUDA。
若没有可用的 CUDA 设备，可通过设置 ``--FeatureExtraction.use_gpu 0`` 与 ``--FeatureMatching.use_gpu 0``
选项，手动选择基于 CPU 的特征提取与匹配。

帮助
----

可用命令可通过以下命令列出::

    $ colmap help

        Usage:
          colmap [command] [options]

        Documentation:
          https://colmap.github.io/

        Example usage:
          colmap help [ -h, --help ]
          colmap gui
          colmap gui -h [ --help ]
          colmap automatic_reconstructor -h [ --help ]
          colmap automatic_reconstructor --image_path IMAGES --workspace_path WORKSPACE
          colmap feature_extractor --image_path IMAGES --database_path DATABASE
          colmap exhaustive_matcher --database_path DATABASE
          colmap mapper --image_path IMAGES --database_path DATABASE --output_path MODEL
          ...

        Available commands:
          help
          gui
          automatic_reconstructor
          bundle_adjuster
          color_extractor
          database_cleaner
          database_creator
          database_merger
          delaunay_mesher
          exhaustive_matcher
          feature_extractor
          feature_importer
          geometric_verifier
          global_mapper
          guided_geometric_verifier
          hierarchical_mapper
          image_deleter
          image_filterer
          image_rectifier
          image_registrator
          image_undistorter
          image_undistorter_standalone
          mapper
          matches_importer
          mesh_simplifier
          mesh_texturer
          model_aligner
          model_analyzer
          model_clusterer
          model_comparer
          model_converter
          model_cropper
          model_merger
          model_orientation_aligner
          model_splitter
          model_transformer
          patch_match_stereo
          point_filtering
          point_triangulator
          pose_prior_mapper
          poisson_mesher
          project_generator
          rig_configurator
          rotation_averager
          sequential_matcher
          spatial_matcher
          stereo_fusion
          transitive_matcher
          view_graph_calibrator
          vocab_tree_builder
          vocab_tree_matcher
          vocab_tree_retriever

并且每条命令都有 ``-h,--help`` 命令行参数，用于显示用法与可用选项，例如::

    $ colmap feature_extractor -h

        Options can either be specified via command-line or by defining
        them in a .ini project file passed to ``--project_path``.

            -h [ --help ]
            --default_random_seed arg (=0)
            --log_target arg (=stderr_and_file)   {stderr, stdout, file, stderr_and_file}
            --log_path arg
            --log_level arg (=0)
            --log_severity arg (=0)               {INFO = 0, WARNING = 1, ERROR = 2, FATAL = 3}
            --log_color arg (=1)
            --project_path arg
            --database_path arg
            --image_path arg
            --camera_mode arg (=-1)
            --image_list_path arg
            --descriptor_normalization arg (=L1_ROOT)
                                                  {L1_ROOT, L2}
            --ImageReader.mask_path arg
            --ImageReader.camera_model arg (=SIMPLE_RADIAL)
                                                  {SIMPLE_PINHOLE, PINHOLE, SIMPLE_RADIAL, RADIAL, OPENCV, OPENCV_FISHEYE,
                                                  FULL_OPENCV, FOV, SIMPLE_RADIAL_FISHEYE, RADIAL_FISHEYE, THIN_PRISM_FISHEYE,
                                                  RAD_TAN_THIN_PRISM_FISHEYE, SIMPLE_DIVISION, DIVISION, SIMPLE_FISHEYE, FISHEYE,
                                                  EUCM, EQUIRECTANGULAR}
            --ImageReader.single_camera arg (=0)
            --ImageReader.single_camera_per_folder arg (=0)
            --ImageReader.single_camera_per_image arg (=0)
            --ImageReader.existing_camera_id arg (=-1)
            --ImageReader.camera_params arg
            --ImageReader.default_focal_length_factor arg (=1.2)
            --ImageReader.camera_mask_path arg
            --FeatureExtraction.type arg (=SIFT)  {SIFT, ALIKED_N16ROT, ALIKED_N32, LOMA_B, LOMA_B128}
            --FeatureExtraction.max_image_size arg (=3200)
            --FeatureExtraction.num_threads arg (=-1)
            --FeatureExtraction.use_gpu arg (=1)
            --FeatureExtraction.gpu_index arg (=-1)
            --SiftExtraction.max_num_features arg (=8192)
            --SiftExtraction.first_octave arg (=-1)
            --SiftExtraction.num_octaves arg (=4)
            --SiftExtraction.octave_resolution arg (=3)
            --SiftExtraction.peak_threshold arg (=0.0066666666666666671)
            --SiftExtraction.edge_threshold arg (=10)
            --SiftExtraction.estimate_affine_shape arg (=0)
            --SiftExtraction.max_num_orientations arg (=2)
            --SiftExtraction.upright arg (=0)
            --SiftExtraction.domain_size_pooling arg (=0)
            --SiftExtraction.dsp_min_scale arg (=0.16666666666666666)
            --SiftExtraction.dsp_max_scale arg (=3)
            --SiftExtraction.dsp_num_scales arg (=10)


可用选项既可直接在命令行提供，也可通过传给 ``--project_path`` 的 ``.ini`` 文件提供。


命令
----

以下列表简要说明各命令的功能，均可作为 ``colmap [command]`` 使用：

- ``gui``: 图形用户界面，更多信息见
  :ref:`Graphical User Interface <gui>`\。

- ``automatic_reconstructor``: 为一组输入图像自动重建稀疏与稠密模型。
  关键选项包括 ``--quality``\（LOW、MEDIUM、HIGH、EXTREME）、
  ``--data_type``\（INDIVIDUAL、VIDEO、INTERNET）以针对不同采集场景调优设置、
  ``--feature``\（SIFT、ALIKED、LOMA、LOMA128）以选择特征提取算法、
  ``--mapper``\（INCREMENTAL、HIERARCHICAL、GLOBAL）以选择 SfM 流程，以及
  ``--mesher``\（POISSON、DELAUNAY、ADVANCING_FRONT）以选择表面重建方法。

- ``project_generator``: 按不同质量设置生成项目文件。

- ``feature_extractor``, ``feature_importer``: 为一组图像执行特征提取或导入特征。

- ``exhaustive_matcher``, ``vocab_tree_matcher``, ``sequential_matcher``,
  ``spatial_matcher``, ``transitive_matcher``, ``matches_importer``:
  在特征提取之后执行特征匹配。

- ``geometric_verifier``: 对数据库中已有的特征匹配运行独立几何验证。
  为匹配的图像对估计双视图几何（基础矩阵/本质矩阵、单应矩阵）。

- ``guided_geometric_verifier``: 在已有稀疏重建引导下运行几何验证。
  利用已知的相对相机位姿改进匹配验证结果。

- ``mapper``: 在特征提取与匹配之后，使用 SfM 对数据集进行稀疏三维重建 / 建图。

- ``global_mapper``: 使用全局 SfM 流程进行稀疏三维重建。
  与增量式 ``mapper`` 不同，全局方法通过旋转平均与全局定位同时求解所有相机位姿。
  对大型数据集可能更快，但对离群点可能不够稳健。
  全局 mapper 依赖较为合理的焦距先验才能表现良好。在 ``global_mapper`` 之前运行
  ``view_graph_calibrator``，以从视图图标定相机内参并估计相对位姿，或手动提供相机标定。

- ``pose_prior_mapper``: 使用位姿先验进行稀疏三维重建 / 建图。

- ``hierarchical_mapper``: 在特征提取与匹配之后，使用分层 SfM 对数据集进行稀疏三维重建 / 建图。
  该方法通过将场景划分为重叠子模型并独立重建每个子模型来并行化重建过程。
  最后将重叠子模型合并为单个重建。建议在此步骤之后再运行几轮点三角化与光束法平差。

- ``image_undistorter``: 对图像去畸变，和/或导出以供 MVS 或外部稠密重建软件（如 CMVS/PMVS）使用。

- ``image_rectifier``: 对相机进行立体校正并对图像去畸变，用于立体视差估计。

- ``image_filterer``: 从稀疏重建中过滤图像。

- ``image_deleter``: 从稀疏重建中删除指定图像。

- ``patch_match_stereo``: 在运行 ``image_undistorter`` 初始化工作区之后，使用 MVS 进行稠密三维重建 / 建图。

- ``stereo_fusion``: 将 ``patch_match_stereo`` 结果融合成彩色点云。

- ``poisson_mesher``: 使用 Poisson 表面重建对融合点云进行网格化。

- ``delaunay_mesher``: 通过对 Delaunay 三角剖分与可见性投票进行图割，对重建的稀疏或稠密点云进行网格化。

- ``advancing_front_mesher``: 使用推进前沿表面重建对融合点云进行网格化。支持基于可见性的过滤，
  以及面向大规模场景的分块并行处理。需要 CGAL。

- ``mesh_simplifier``: 使用二次误差度量（QEM）简化三角网格（PLY 格式）。在保留整体形状与外观的同时减少面数。
  关键选项包括 ``--MeshSimplification.target_face_ratio`` 以控制保留面数比例（默认 0.1）、
  ``--MeshSimplification.max_error`` 以设置最大二次误差阈值（0 = 禁用），以及
  ``--MeshSimplification.boundary_weight`` 以控制边界边保留（默认 1000）。
  通过 ``--MeshSimplification.num_threads`` 支持多线程初始化。

- ``mesh_texturer``: 使用已标定的多视图图像为三角网格生成纹理图集与 UV 坐标。

- ``image_registrator``: 将数据库中的新图像注册到已有模型，例如在运行 ``mapper`` 之后，
  对数据库中新添加的图像提取特征并匹配。注意不会执行光束法平差或三角化。

- ``point_triangulator``: 利用数据库中的特征匹配，对已有模型中已注册图像的所有观测进行三角化。

- ``point_filtering``: 通过强制约束（如最小轨迹长度、最大重投影误差等）过滤模型中的稀疏点。

- ``bundle_adjuster``: 对重建场景运行全局光束法平差，例如需要优化内参时，或在运行
  ``image_registrator`` 之后。求解器后端通过 ``--BundleAdjustment.backend``\（``CERES`` 或 ``CASPAR``）选择。
  Caspar [caspar]_ 是实验性的 GPU 加速后端；详情与限制见 :doc:`faq`\。

- ``database_cleaner``: 清理指定或全部数据库表。

- ``database_creator``: 创建带有必要数据库模式信息的空 COLMAP SQLite 数据库。

- ``database_merger``: 将两个数据库合并为新数据库。注意相机不会被合并，且合并过程中唯一的相机与图像标识符可能会改变。

- ``model_analyzer``: 打印重建的统计信息。

- ``model_clusterer``: 将重建拆分为更小的子模型簇。适用于管理与处理大规模重建。

- ``model_aligner``: 将模型对齐 / 地理配准到给定相机中心的坐标系。

- ``model_orientation_aligner``: 在曼哈顿世界假设下对齐模型的坐标轴。

- ``model_comparer``: 比较两个重建的统计信息。

- ``model_converter``: 将 COLMAP 导出格式转换为其他格式，例如 PLY 或 NVM。

- ``model_cropper``: 按 GPS 或模型坐标系中描述的特定包围盒裁剪模型。

- ``model_merger``: 尝试合并两个不相连的重建（若它们有共同已注册图像）。

- ``model_splitter``: 将模型划分为矩形子模型，由包含包围盒坐标的文件、子模型最大范围，或各维细分数量指定。

- ``model_transformer``: 变换模型的坐标框架。

- ``color_extractor``: 为模型的所有三维点提取平均颜色。

- ``rig_configurator``: 在特征提取之后配置 rig 与 frame。

- ``vocab_tree_builder``: 从包含已提取图像特征的数据库创建词汇树。这是离线过程，可运行一次，
  同一词汇树可复用于其他数据集。注意经验法则是：特征数量至少应为视觉词数量的 10–100 倍。
  预训练树可从 https://demuc.de/colmap/ 下载。
  若要构建在精度/召回与速度之间具有不同权衡的自定义树，这很有用。

- ``vocab_tree_retriever``: 执行基于词汇树的图像检索。

- ``rotation_averager``: 在视图图上运行独立旋转平均。从成对相对旋转估计全局相机旋转。

- ``view_graph_calibrator``: 使用视图图标定相机内参。从成对几何关系估计焦距及其他内参。
  若没有良好的先验相机内参，应在 ``global_mapper`` 之前运行，因为全局 mapper
  依赖较为合理的焦距先验才能表现良好。


可视化
------

若要快速可视化稀疏或稠密重建流程的输出，COLMAP 提供以下方式：

- 通过 ``mapper`` 得到的稀疏点云可在 COLMAP GUI 中可视化：选择 ``File > Import Model``
  并选择包含稀疏模型文件（``cameras.txt``、``images.txt``、``points3D.txt`` 等）的文件夹。

- 通过 ``stereo_fusion`` 得到的稠密点云可在 COLMAP GUI 中可视化：导入 ``fused.ply``，选择
  ``File > Import Model from...``，然后选择文件 ``fused.ply``\。

- 通过 ``poisson_mesher`` 或 ``delaunay_mesher`` 得到的稠密网格模型 ``meshed-*.ply``
  目前无法用 COLMAP 可视化，可改用外部查看器（如 Meshlab）。使用 ``mesh_simplifier``
  命令可减小网格尺寸，以便更快可视化或下游处理。使用 ``mesh_texturer`` 命令可生成带纹理图集的纹理网格，
  可在 Meshlab 或其他三维查看器中可视化。
