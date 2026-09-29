.. _tutorial:

教程
====

本教程通过演示 COLMAP 中各个处理步骤，介绍基于图像的三维重建。如果您希望了解更通用、更偏数学的基于图像三维重建导论，也可参阅 `CVPR 2017 Tutorial on Large-scale 3D
Modeling from Crowdsourced Data <https://demuc.de/tutorials/cvpr2017/>`_ 以及
[schoenberger_thesis]_。

基于图像的三维重建传统上首先使用 Structure-from-Motion 恢复场景的稀疏表示以及输入图像的相机位姿。该输出随后作为 Multi-View Stereo 的输入，以恢复场景的稠密表示。

.. contents:: 目录
    :local:
    :depth: 1


.. _quick-start:

快速入门
--------

首先，按 :ref:`此处 <gui>` 所述启动 COLMAP 的图形用户界面。COLMAP 提供自动重建工具，只需指定输入图像文件夹，即可在工作区文件夹中生成稀疏与稠密重建结果。在 GUI 中点击 ``Reconstruction > Automatic Reconstruction``，并设置相关选项。输出将写入工作区文件夹。例如，若图像位于 ``path/to/project/images``，可将 ``path/to/project`` 选为工作区文件夹；运行自动重建工具后，文件夹结构大致如下::

    +── images
    │   +── image1.jpg
    │   +── image2.jpg
    │   +── ...
    +── sparse
    │   +── 0
    │   │   +── rigs.bin
    │   │   +── cameras.bin
    │   │   +── frames.bin
    │   │   +── images.bin
    │   │   +── points3D.bin
    │   +── ...
    +── dense
    │   +── 0
    │   │   +── images
    │   │   +── sparse
    │   │   +── stereo
    │   │   +── fused.ply
    │   │   +── meshed-poisson.ply
    │   │   +── meshed-delaunay.ply
    │   │   +── meshed-advancing-front.ply
    │   +── ...
    +── database.db

其中，``path/to/project/sparse`` 包含所有重建组件的稀疏模型，而 ``path/to/project/dense`` 包含对应的稠密模型。稠密点云 ``fused.ply`` 可通过 ``File > Import from ...`` 导入 COLMAP，而稠密网格需使用 Meshlab 等外部查看器可视化。

同样的自动重建也可在不使用 GUI 的情况下从命令行运行，使用 ``automatic_reconstructor`` 命令，其生成的工作区布局与上文相同::

    colmap automatic_reconstructor \\
        --workspace_path path/to/project \\
        --image_path path/to/project/images

后续各节给出一般性建议，并更详细地描述重建流程；若您需要对重建过程/参数有更多控制，或对 COLMAP 的底层技术感兴趣，可继续阅读。


前言
----

对于一般用户，COLMAP 只需少量步骤即可完成标准重建。对更有经验的用户，程序暴露了许多不同参数，其中只有一部分对初学者直观。通常无需修改任何参数即可使用。默认值是在重建稳健性/质量与速度之间的折中。可通过选择 ``Extras > Set
options for ... data`` 为不同重建场景设置“最优”选项。若不确定应选择何种设置，请坚持使用默认值。源代码中包含关于所有参数的更多文档。

COLMAP 是研究型软件，在极少数情况下，若某些约束未满足，程序可能非优雅退出。此时，程序会向 stderr 打印回溯信息。要查看该回溯或更多调试信息，建议从命令行运行可执行文件（包括 GUI）；在命令行中也可定义不同级别的日志详细程度。


运动恢复结构（Structure-from-Motion）
-------------------------------------

.. figure:: images/incremental-sfm.webp
    :alt: Incremental Structure-from-Motion pipeline
    :figclass: align-center

    COLMAP 的增量 Structure-from-Motion 流水线。

Structure-from-Motion（SfM）是从一系列图像中的投影恢复三维结构的过程。输入是同一物体从不同视点拍摄的一组重叠图像。输出是该物体的三维重建，以及所有图像的重建内参与外参。通常，Structure-from-Motion 系统将该过程分为三个阶段：

1) 特征检测与提取
2) 特征匹配与几何验证
3) 结构与运动重建

COLMAP 将这些阶段体现在不同模块中，可根据应用进行组合。关于 Structure-from-Motion 的一般介绍以及 COLMAP 中的算法，可参阅 [schoenberger16sfm]_ 和
[schoenberger16mvs]_。

若您能够控制拍摄过程，请遵循以下准则以获得最佳重建结果：

- 拍摄具有**良好纹理**\的图像。避免完全无纹理的图像（例如白墙或空桌面）。若场景本身纹理不足，可放置海报等额外背景物体。

- 在**相似光照**\条件下拍摄图像。避免高动态范围场景（例如逆光带阴影，或透过门/窗拍摄）。避免光滑表面上的镜面反射。

- 拍摄具有**高视觉重叠**\的图像。确保每个物体至少出现在 3 张图像中——图像越多越好。

- 从**不同视点**\拍摄图像。不要仅在同一位置旋转相机拍摄，例如每次拍摄后走动几步。同时，也尽量保证有足够来自相对相近视点的图像。注意，更多图像未必更好，且可能导致重建过程变慢。若使用视频作为输入，请考虑对帧率进行降采样。


多视图立体（Multi-View Stereo）
-------------------------------

Multi-View Stereo（MVS）以 SfM 的输出为输入，为图像中每个像素计算深度和/或法线信息。将多张图像的深度图与法线图在三维中融合，即可得到场景的稠密点云。利用融合点云的深度与法线信息，（screened）Poisson 表面重建 [kazhdan2013]_ 或 advancing front 表面重建 [cohen-steiner2004]_ 等算法可进一步恢复场景的三维表面几何。所得网格可选用 Quadric Error Metric（QEM）简化 [garland1997]_，在保留形状与外观的同时降低复杂度。此外，网格还可使用多视图纹理映射 [waechter2014]_ 进行纹理化：为每个面片分配最佳视角的相机图像，并生成带 UV 坐标的纹理图集。关于 Multi-View Stereo 的一般介绍以及 COLMAP 中的算法，可参阅 [schoenberger16mvs]_。


术语
----

术语 **camera** 指使用相同变焦倍数和镜头的物理相机。在 COLMAP 中，相机定义了内参投影模型。单个相机可拍摄多张具有相同分辨率、内参与畸变特性的图像。术语 **image** 与位图文件相关联，例如磁盘上的 JPEG 或 PNG 文件。COLMAP 在每张图像中检测 **keypoints**，其外观由数值 **descriptors** 描述。基于纯外观的关键点/描述子对应关系由 **matches** 定义，而 **inlier matches** 是经过几何验证、用于重建过程的匹配。

术语 **rig** 描述由一个或多个相机组成的固定装配，其相对位姿随时间保持恒定，例如立体或多相机装置，或单台运动相机（仅含一台相机的平凡 rig）。术语 **frame** 表示某一时刻的单次快照，即该 rig 中所有相机同时捕获的一组图像。因此，一个 frame 将共享同一 rig 位姿的图像归为一组，COLMAP 对每个 frame 优化一个位姿，而非对每张图像优化一个位姿。Rigs 与 frames 在重建输出中分别存储为 ``rigs.bin`` 和 ``frames.bin``\（参见 :ref:`Rig Support <rig-support>` 与 :ref:`Output Format <output-format>`）。


数据结构
--------

COLMAP 假定所有输入图像位于一个输入目录中，其中可包含嵌套子目录。它会递归考虑该目录中存储的所有图像，并通过 OpenImageIO 支持多种图像格式。其他文件会被自动忽略。若对性能要求较高，应将非图像文件分开存放。图像由其相对文件路径唯一标识。为便于后续处理（如图像去畸变或稠密重建），应保持相对文件夹结构。COLMAP 不会修改输入图像或目录，所有提取的数据都存储在单个自包含的 SQLite 数据库文件中（参见 :doc:`database`）。

第一步是通过运行预构建二进制文件（Windows：``COLMAP.bat``，Mac：``COLMAP.app``）或从 CMake 构建文件夹执行 ``./src/colmap/exe/colmap gui`` 来启动 COLMAP 的图形用户界面。接下来，选择 ``File > New project`` 创建新项目。在该对话框中，必须选择数据库存储位置以及包含输入图像的文件夹。为方便起见，可通过选择 ``File > Save project`` 将整个项目设置保存到配置文件。项目配置除其他参数设置外，还存储数据库与图像文件夹的绝对路径信息。若决定移动数据库或图像文件夹，必须通过创建新项目相应地更改路径。或者，也可在任意文本编辑器中直接修改生成的 ``.ini`` 配置文件。要重新打开现有项目，只需选择 ``File > Open project`` 打开配置文件，所有参数设置应被恢复。请注意，所有 COLMAP 可执行文件都可从命令行启动，方法是将各项设置指定为命令行参数，或提供项目配置文件的路径（参见 :ref:`Interface <interface>`）。

示例文件夹结构可能如下::

    /path/to/project/...
    +── images
    │   +── image1.jpg
    │   +── image2.jpg
    │   +── ...
    │   +── imageN.jpg
    +── database.db
    +── project.ini

在此示例中，应将 ``/path/to/project/images`` 选为图像文件夹路径，将 ``/path/to/project/database.db`` 选为数据库文件路径，并将项目配置保存到 ``/path/to/project/project.ini``\。


特征检测与提取
--------------

第一步中，特征检测/提取在图像中寻找稀疏特征点，并用数值描述子描述其外观。COLMAP 在单一步骤中导入图像并执行特征检测/提取，因此每张图像只需从磁盘加载一次。

接下来，选择 ``Processing > Extract features``\。在该对话框中，必须首先决定要使用的内参相机模型。您可以自动从嵌入的 EXIF 信息中提取焦距信息，或手动指定内参（例如实验室标定得到的参数）。若图像仅有部分 EXIF 信息，COLMAP 会尝试在大型相机型号数据库中自动查找缺失的相机规格。若所有图像由同一物理相机以相同变焦倍数拍摄，建议在所有图像之间共享内参。请注意，若在所有图像之间共享同一相机模型，但并非所有图像具有相同尺寸或 EXIF 焦距，程序将非优雅退出。若有若干组图像分别共享相同的内参相机参数，也可在稍后轻松修改相机模型（参见 :ref:`Database Management
<database-management>`）。若不确定此步骤应如何选择，请直接使用默认参数。

您可以从图像中检测并提取新特征，或从文本文件导入已有特征。默认情况下，COLMAP 在 GPU 或 CPU 上提取 SIFT [lowe04]_ 特征。当 COLMAP 以 CUDA 支持构建时（推荐），GPU 特征提取无需连接显示器即可运行，适合无头服务器。基于 OpenGL 的回退方案（在 CUDA 不可用时使用）则需要连接显示器，因此在此类系统上，建议在服务器上使用 CPU 版本。一般而言，GPU 版本更优，因为它具有定制的特征检测模式，对高对比度图像通常能产生更高质量的特征。COLMAP 还支持 ALIKED 和 LoMa 特征提取——使用 ONNX 模型的学习型特征提取器，可通过 ``--FeatureExtraction.type`` 选项选择（详情参见 :ref:`Feature Extraction and Matching <features>`）。若导入已有特征，每张图像旁必须有一个文本文件（例如 ``/path/to/image1.jpg`` 与 ``/path/to/image1.jpg.txt``），格式如下::

    NUM_FEATURES 128
    X Y SCALE ORIENTATION D_1 D_2 D_3 ... D_128
    ...
    X Y SCALE ORIENTATION D_1 D_2 D_3 ... D_128

其中 ``X, Y, SCALE, ORIENTATION`` 为浮点数，``D_1...D_128`` 的取值范围为 ``0...255``\。文件应有 ``NUM_FEATURES`` 行，每行对应一个特征。例如，若某图像有 4 个特征，则文本文件大致如下::

    4 128
    1.2 2.3 0.1 0.3 1 2 3 4 ... 21
    2.2 3.3 1.1 0.3 3 2 3 2 ... 32
    0.2 1.3 1.1 0.3 3 2 3 2 ... 2
    1.2 2.3 1.1 0.3 3 2 3 2 ... 3

请注意，按约定图像左上角坐标为 ``(0, 0)``，最左上角像素的中心坐标为 ``(0.5, 0.5)``\。若必须为大型图像集合导入特征，使用您喜欢的脚本语言直接访问数据库会高效得多（参见 :ref:`Database Format <database-format>`）。

设置完所有选项后，选择 ``Extract``，等待提取完成或取消。若在提取过程中取消，下次为同一项目开始提取图像时，COLMAP 会自动从中断处继续。这也允许您向现有项目/重建添加图像。在此情况下，若使用共享内参，请务必验证相机参数。

所有提取的数据将存储在数据库文件中，可在数据库管理工具中查看/管理（参见 :ref:`Database Management
<database-management>`），或由专家使用 SQLite 直接修改（参见 :ref:`Database Format <database-format>`）。


特征匹配与几何验证
------------------

第二步中，特征匹配与几何验证寻找不同图像中特征点之间的对应关系。

请选择 ``Processing > Feature matching``，并选择一种提供的匹配模式；这些模式面向不同的输入场景：

- **Exhaustive Matching**：若数据集中的图像数量相对较少（最多数百张），此匹配模式应足够快，并能获得最佳重建结果。此处每张图像与其他每张图像进行匹配，而块大小决定同时从磁盘加载到内存中的图像数量。

- **Sequential Matching**：若图像按顺序获取（例如由摄像机拍摄），此模式很有用。在此情况下，连续帧具有视觉重叠，无需穷尽匹配所有图像对。相反，连续捕获的图像彼此匹配。此匹配模式内置基于词汇树的回环检测，每第 N 张图像（``--SequentialMatching.loop_detection_period``）会与其视觉上最相似的图像匹配（``--SequentialMatching.loop_detection_num_images``）。检索到的图像可限制为在序列中与查询图像距离足够远的图像（``--SequentialMatching.loop_detection_min_index_distance``），从而使附近图像不会占用回环检测预算。值为零则禁用此限制。请注意，图像文件名必须按顺序排列（例如 ``image0001.jpg``、``image0002.jpg`` 等）。数据库中的顺序无关紧要，因为图像会根据其文件名显式排序。请注意，回环检测需要预训练的词汇树。默认树会自动下载并缓存。更多树可从 https://demuc.de/colmap/ 下载。若数据库中已适当配置 rigs 与 frames，顺序匹配会自动将连续 frame 中的所有图像相互匹配。

- **Vocabulary Tree Matching**：在此匹配模式 [schoenberger16vote]_ 中，每张图像使用带空间重排序的词汇树与其视觉近邻匹配。这是大型图像集合（数千张）的推荐匹配模式。这需要预训练的词汇树，可从 https://demuc.de/colmap/ 下载。

- **Spatial Matching**：此匹配模式将每张图像与其空间近邻匹配。空间位置可在数据库管理中手动设置。默认情况下，COLMAP 也会从 EXIF 提取 GPS 信息，并将其用于空间近邻搜索。若有准确的先验位置信息，这是推荐的匹配模式。

- **Transitive Matching**：此匹配模式利用已有特征匹配的传递关系，以生成更完整的匹配图。若图像 A 与图像 B 匹配，且 B 与 C 匹配，则该匹配器会尝试直接将 A 与 C 匹配。

- **Custom Matching**：此模式允许您指定用于匹配的单个图像对，或导入单个特征匹配。要指定图像对，需提供一个文本文件，每行一对图像::

    image1.jpg image2.jpg
    image1.jpg image3.jpg
    ...

  其中 ``image1.jpg`` 是图像文件夹中的相对路径。导入单个特征匹配有两种选项：未经几何验证的原始特征匹配，或已经过几何验证的特征匹配。两种情况下，期望的格式均为::

    image1.jpg image2.jpg
    0 1
    1 2
    3 4
    <empty-line>
    image1.jpg image3.jpg
    0 1
    1 2
    3 4
    4 5
    <empty-line>
    ...

  其中 ``image1.jpg`` 是图像文件夹中的相对路径，数字对是相应图像中从零开始的特征索引。若必须为大型图像集合导入大量匹配，使用您选择的脚本语言直接访问数据库会更高效。

设置完所有选项后，选择 ``Match``，等待匹配完成或中途取消。请注意，此步骤可能根据图像数量、每张图像的特征数量以及所选匹配模式而花费大量时间。穷尽匹配的预期时间从数十张图像的几分钟，到数百张图像的几小时，再到数千张图像的数天或数周不等。穷尽匹配随图像数量呈二次方扩展，对大型集合很快变得不切实际；对于数千张或更多图像，请改用词汇树或顺序匹配，它们会快得多。若取消匹配过程，或在匹配后导入新图像，COLMAP 仅匹配此前尚未匹配的图像对。跳过已匹配图像对的开销较低。这也使得可以匹配在初始匹配之后导入的额外图像，并可为同一数据集组合不同的匹配模式。

所有提取的数据将存储在数据库文件中，可在数据库管理工具中查看/管理（参见 :ref:`Database Management
<database-management>`），或由专家使用 SQLite 直接修改（参见 :ref:`Database Format <database-format>`）。

请注意，SIFT 特征匹配可使用 GPU 加速，匹配过程中计算机的显示性能可能显著下降。若系统有多个支持 CUDA 的 GPU，可使用 ``--FeatureMatching.gpu_index`` 选项选择特定 GPU。也可通过设置 ``--FeatureMatching.use_gpu 0`` 在 CPU 上执行特征匹配，但对大型数据集这会显著更慢。


稀疏重建
--------

在前两步生成场景图后，可通过选择 ``Reconstruction > Start`` 开始增量重建过程。COLMAP 首先将所有提取的数据从数据库加载到内存，并从初始图像对播种重建。然后，通过注册新图像并三角化新点，逐步扩展场景。结果在此重建过程中“实时”可视化。有关可用控件的更多详情，请参阅 :ref:`Graphical User Interface <gui>` 一节。若并非所有图像都注册到同一模型中，COLMAP 会尝试重建多个模型。不同模型可从工具栏的下拉菜单中选择。若不同模型有共同的已注册图像，可使用 ``model_merger`` 可执行文件将它们合并为单个重建（详情参见 :ref:`FAQ <faq-merge-models>`）。

理想情况下，重建运行良好且所有图像均被注册。若非如此，建议：

- 执行额外匹配。为获得最佳结果，可使用穷尽匹配、启用 guided matching、增加词汇树匹配中的近邻数量，或增加顺序匹配中的重叠等。

- 若 COLMAP 初始化失败，可手动选择初始图像对。选择 ``Reconstruction > Reconstruction options > Init``，并从数据库管理工具中设置具有足够匹配且来自不同视点的图像。


导入与导出
----------

COLMAP 提供多种导出选项以供进一步处理。为获得完整灵活性，建议通过选择 ``File > Export model`` 导出当前查看的模型，或选择 ``File > Export all models`` 导出所有重建模型，以 COLMAP 的数据格式导出重建。模型会使用单独的文本文件导出到所选文件夹，分别对应重建的相机、图像与点。以 COLMAP 的数据格式导出时，可重新导入重建以供后续可视化、图像去畸变，或从中断处继续现有重建（例如在导入并匹配新图像之后）。要导入模型，选择 ``File > Import model`` 并选择导出文件夹路径。或者，可通过选择 ``File > Export model as...`` 将模型导出为 Bundler、VisualSfM [#f1]_、PLY 或 VRML 等多种其他格式。COLMAP 可通过选择 ``File > Import from ...`` 可视化带 RGB 信息的纯 PLY 点云文件。有关导出模型格式的更多信息，可参见 :ref:`此处 <output-format>`\。


.. _dense-reconstruction:

稠密重建
--------

在重建出场景的稀疏表示以及输入图像的相机位姿后，MVS 现在可以恢复更稠密的场景几何。COLMAP 具有集成的稠密重建流水线，可为所有已注册图像生成深度图与法线图，将深度图与法线图融合为带法线信息的稠密点云，最后使用 Poisson [kazhdan2013]_ 或 Delaunay 重建从融合点云估计稠密表面。可选地，可使用 ``mesh_simplifier`` 命令简化所得网格，在保留整体形状的同时减少面片数量。也可使用 ``mesh_texturer`` 命令对网格进行纹理化，从去畸变图像生成纹理图集与每个面片的 UV 坐标。

首先，将稀疏三维模型导入 COLMAP（或在完成先前稀疏重建步骤后选择重建模型）。然后，选择 ``Reconstruction > Multi-view stereo``，并选择一个空的或已有的工作区文件夹，用于所有稠密重建结果的输出。第一步是对图像进行 ``undistort``，第二步使用 ``stereo`` 计算深度图与法线图，第三步将深度图与法线图 ``fuse`` 为点云，随后可选地进行点云 ``meshing`` 步骤。这些步骤也可分别从命令行作为 ``image_undistorter``、``patch_match_stereo``、``stereo_fusion`` 以及 ``poisson_mesher`` / ``delaunay_mesher`` 命令使用。在立体重建过程中，由于计算负载很重，显示可能会冻结；若 GPU 内存不足，重建过程可能非优雅崩溃。请参阅 FAQ（:ref:`freeze <faq-dense-timeout>` 与 :ref:`memory <faq-dense-memory>`）了解如何避免这些问题。请注意，点云的重建法线无法在 COLMAP 中直接可视化，但可在 Meshlab 等外部工具中通过启用 ``Render > Show Normal/Curvature`` 查看。类似地，重建的稠密表面网格模型必须使用外部软件可视化。

除内部稠密重建功能外，COLMAP 还可导出到其他若干稠密重建库，例如 CMVS/PMVS
[furukawa10]_ 或 CMP-MVS [jancosek11]_。请选择 ``Extras > Undistort images`` 并选择适当格式。输出文件夹包含重建结果与去畸变图像。此外，这些文件夹包含用于执行稠密重建的示例 shell 脚本。要运行 PMVS2，请执行以下命令::

    ./path/to/pmvs2 /path/to/undistortion/folder/pmvs/ option-all

其中 ``/path/to/undistortion/folder`` 是去畸变对话框中选择的文件夹。请务必不要忘记上述命令行参数中 ``/path/to/undistortion/folder/pmvs/`` 末尾的斜杠。

对于大型数据集，您可能希望先运行 CMVS 将场景聚类为更易管理的部分，然后再运行 COLMAP 或 PMVS2。请参阅去畸变输出文件夹中的示例 shell 脚本，了解如何将 CMVS 与 COLMAP 或 PMVS2 结合使用。此外，还有若干支持 COLMAP 输出的外部库：

- `CMVS/PMVS <http://www.di.ens.fr/pmvs/>`_ [furukawa10]_
- `CMP-MVS <http://ptak.felk.cvut.cz/sfmservice/websfm.pl>`_ [jancosek11]_
- `Line3D++ <https://github.com/manhofer/Line3Dpp>`_ [hofer16]_.


.. _database-management:

数据库管理
----------

您可以在数据库管理工具中查看和管理导入的相机、图像与特征匹配。选择 ``Processing > Manage database``\。在打开的对话框中，可以看到导入的图像与相机列表。可通过点击 ``Show image`` 和 ``Overlapping images`` 查看每张图像的特征与匹配。可通过双击特定单元格修改数据库表中的各个条目。请注意，对数据库的任何更改仅在点击 ``Save`` 后生效。

要在任意图像组之间共享内参相机参数，请选择一张或多张图像，选择 ``Set camera`` 并设置 ``camera_id``，该值对应相机表中唯一的 ``camera_id`` 列。您也可以添加具有特定参数的新相机。通过将 ``prior_focal_length`` 标志设为 0 或 1，可提示重建算法是否应信任焦距值。若有先验实验室标定，应将此值设为 1。在没有关于焦距的先验知识时，建议将此值设为 ``1.25 *
max(width_in_px, height_in_px)``。

数据库管理工具功能有限；要对数据有完全控制，必须直接修改 SQLite 数据库（参见 :ref:`Database Format <database-format>`）。通过直接访问数据库，可仅使用 COLMAP 进行特征提取与匹配，或导入自己的特征与匹配，并仅使用 COLMAP 的增量重建算法。


.. _interface:

图形与命令行界面
----------------

COLMAP 的大多数功能可从图形界面与命令行界面访问，二者都嵌入在同一可执行文件中。您可以直接将选项作为命令行参数提供，或使用 ``--project_path path/to/project.ini`` 参数提供包含选项的 ``.ini`` 项目配置文件。要启动 GUI 应用程序，请执行 ``colmap gui``，或直接指定项目配置为 ``colmap gui --project_path path/to/project.ini``，以避免在 GUI 中繁琐地选择。要列出命令行可用的不同命令，请执行 ``colmap help``\。例如，要从命令行运行特征提取，必须执行 ``colmap feature_extractor``\。:ref:`graphical user
interface <gui>` 与 :ref:`command-line interface <cli>` 两节提供关于可用命令的更多详情。


.. rubric:: Footnotes

.. [#f1] VisualSfM [wu13]_ 的投影模型将畸变应用于测量值，而 COLMAP 将畸变应用于投影，因此导出的 NVM 文件与 VisualSfM 并非完全兼容。
