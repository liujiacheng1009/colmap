.. _rig-support:

Rig
===

COLMAP 在重建过程中原生支持对传感器 rig 建模。rig 中的传感器被假定彼此之间
具有固定的相对位姿，其中一个参考传感器定义了 rig 的原点。frame 定义了
rig 在某一时刻的具体实例，此时所有或一部分传感器同时曝光。例如，在立体相机
rig 中，一台相机会被定义为参考传感器，并具有单位（identity）的
``sensor_from_rig`` 位姿，而第二台相机则相对于参考相机进行位姿表示。每个
frame 通常由两张图像组成，作为两台相机在同一时刻的测量结果。

工作流程
--------

默认情况下，在运行标准重建流水线时，每台相机都会被建模为单独的 rig，因此
每个 frame 只包含一张图像。要对 rig 建模，推荐的工作流程是按如下文件夹结构
按 rig 和相机组织图像（确保对应于同一 frame 的图像在所有文件夹中具有相同的
文件名）::

    rig1/
        camera1/
            image0001.jpg
            image0002.jpg
            ...
        camera2/
            image0001.jpg # same frame as camera1/image0001.jpg
            image0002.jpg # same frame as camera1/image0002.jpg
            ...
        ...
    rig2/
        camera1/
            ...
        ...
    ...

下一步，我们首先使用如下命令提取特征::

    colmap feature_extractor \
        --image_path $DATASET_PATH/images \
        --database_path $DATASET_PATH/database.db \
        --ImageReader.single_camera_per_folder 1

默认情况下，得到的数据库现在为每台相机包含一个单独的 rig，并为每张图像
包含一个单独的 frame。因此，我们必须按所需的 rig 配置调整数据库中的关系。
这通过如下命令完成::

    colmap rig_configurator \
        --database_path $DATASET_PATH/database.db \
        --rig_config_path $DATASET_PATH/rig_config.json

其中，如果事先已知 rig 中传感器的相对位姿，``rig_config.json`` 可以如下所示::

    [
      {
        "cameras": [
          {
            "image_prefix": "rig1/camera1/",
            "ref_sensor": true
          },
          {
            "image_prefix": "rig1/camera2/",
            "cam_from_rig_rotation": [
                0.7071067811865475,
                0.0,
                0.7071067811865476,
                0.0
            ],
            "cam_from_rig_translation": [
                0,
                0,
                0
            ]
          }
        ]
      },
      {
        "cameras": [
          {
            "image_prefix": "rig2/camera1/",
            "ref_sensor": true
          },
          ...
        ]
      },
      ...
    ]

请注意，这会修改数据库中的 rig 和 frame 配置，该数据库包含我们随后作为下游
处理步骤输入的完整 rig 规格说明。

如果已知已标定的相机参数，每台相机还可以可选地指定 ``camera_model_name``
和 ``camera_params`` 字段。

对于更细粒度的 rig 和 frame 配置，最便捷的选项是使用 pycolmap 手动配置
数据库，可以通过 ``apply_rig_config`` 函数，或者为获得最大灵活性而单独向
重建添加所需的 rig 和 frame 对象。

接下来，我们运行标准的特征匹配。请注意，在顺序特征匹配之前配置 rig
很重要，因为连续 frame 中的图像会自动相互匹配。

最后，我们可以使用标准的 ``mapper`` 命令重建场景，并可选择通过
``--Mapper.ba_refine_sensor_from_rig 0`` 保持 rig 中的相对位姿固定。

未知的 rig 传感器位姿
---------------------

如果事先不知道 rig 中传感器的相对位姿，而只知道某一组传感器是刚性安装并
在同一时刻曝光，可以尝试以下两步重建方法。开始之前，请确保按上文所述组织
图像，并使用 ``--ImageReader.single_camera_per_folder 1`` 选项执行特征提取。

接下来，在没有 rig 约束的情况下重建场景，将每台相机建模为各自的 rig
（这是 COLMAP 在未进一步配置时的默认行为）。请注意，这可以是来自完整输入
图像集子集的部分重建。唯一的要求是：每台相机必须至少有一张已注册图像，
且该图像与参考相机的一张已注册图像处于同一 frame 中。如果重建成功，并且
已注册图像之间的相对位姿看起来大致正确，则可以继续下一步。

``rig_configurator`` 也可以在没有 ``cam_from_rig_*`` 变换的情况下工作。
通过提供场景的现有（部分）重建，它可以从所有已注册图像计算平均的相对
rig 传感器位姿::

    colmap rig_configurator \
        --database_path $DATASET_PATH/database.db \
        --input_path $DATASET_PATH/sparse-model-without-rigs-and-frames \
        --rig_config_path $DATASET_PATH/rig_config.json \
        [ --output_path $DATASET_PATH/sparse-model-with-rigs-and-frames ]

所提供的 ``rig_config.json`` 只需省略相应的
``cam_from_rig_rotation`` 和 ``cam_from_rig_translation`` 字段。

现在，我们可以对（可选）输出的已配置 rig 和 frame 的重建运行
rig 光束法平差::

    colmap bundle_adjuster \
        --input_path $DATASET_PATH/sparse-model-with-rigs-and-frames \
        --output_path $DATASET_PATH/bundled-sparse-model-with-rigs-and-frames

或者，也可以在有 rig 约束的情况下从头开始重建过程，这可能会得到更准确的
重建结果::

    colmap mapper
        --image_path $DATASET_PATH/images \
        --database_path $DATASET_PATH/database.db \
        --output_path $DATASET_PATH/sparse-model-with-rigs-and-frames


示例
----

以下示例展示了如何使用 COLMAP 的 rig 支持端到端地重建 ETH3D
rig 数据集之一::

    wget https://www.eth3d.net/data/terrains_rig_undistorted.7z
    7zz x terrains_rig_undistorted.7z

    colmap feature_extractor \
        --database_path terrains/database.db \
        --image_path terrains/images \
        --ImageReader.single_camera_per_folder 1

ETH3D 数据集方便地附带了一个真值 COLMAP 重建，我们用它来配置传感器
rig 位姿以及相机模型::

    colmap rig_configurator \
        --database_path terrains/database.db \
        --rig_config_path terrains/rig_config.json \
        --input_path terrains/rig_calibration_undistorted

配合如下 ``rig_config.json``::

    [
        {
            "cameras": [
                {
                    "image_prefix": "images_rig_cam4_undistorted/",
                    "ref_sensor": true
                },
                {
                    "image_prefix": "images_rig_cam5_undistorted/"
                },
                {
                    "image_prefix": "images_rig_cam6_undistorted/"
                },
                {
                    "image_prefix": "images_rig_cam7_undistorted/"
                }
            ]
        }
    ]

请注意，我们没有指定传感器位姿，因为我们使用了现有重建（在本例中是真值，
但也可以是没有 rig 约束的重建，如前一节所述）来自动推断平均的
rig 外参和相机参数。

接下来，我们对 frame 进行顺序匹配，因为它们是作为视频采集的::

    colmap sequential_matcher --database_path terrains/database.db

根据所提供的 sensor_from_rig 位姿的精度，你可以选择启用选项
`--FeatureMatching.rig_verification 1`；或者，如果你知道同一 frame 内的
传感器没有视觉重叠，可以启用选项
`--FeatureMatching.skip_image_pairs_in_same_frame 1`。

最后，我们使用 mapper 重建场景，同时保持真值传感器 rig 位姿和相机参数
固定::

    mkdir -p terrains/sparse
    colmap mapper \
        --database_path terrains/database.db \
        --Mapper.ba_refine_sensor_from_rig 0 \
        --Mapper.ba_refine_focal_length 0 \
        --Mapper.ba_refine_extra_params 0 \
        --image_path terrains/images \
        --output_path terrains/sparse


从 360° 球面图像重建
--------------------

COLMAP 可以通过渲染虚拟针孔图像（类似于立方体贴图）并将其作为相机
rig 处理，来处理 360° 全景图集合。由于 rig 外参和相机内参已知，重建过程
更加稳健。我们提供了一个示例 Python 脚本来重建 360° 集合::

    python python/examples/panorama_sfm.py \
        --input_image_path image_directory \
        --output_path output_directory

请确保使用与你所用 COLMAP 版本对应的脚本版本，因为 HEAD 上的脚本不保证
兼容。

该示例是围绕 ``pycolmap.panorama.reconstruct`` 的命令行封装。透视渲染需要
可选的 ``panorama`` 依赖，可通过 ``pip install 'pycolmap[panorama]'`` 安装。
