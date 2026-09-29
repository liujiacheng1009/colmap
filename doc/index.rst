:html_theme.sidebar_secondary.remove: true
:og:description: COLMAP 是免费开源的运动恢复结构（SfM）与多视图立体（MVS）流程，带图形界面和命令行，可从有序或无序图像重建三维。

.. meta::
   :description: COLMAP 是免费开源的运动恢复结构（SfM）与多视图立体（MVS）流程，带图形界面和命令行，可从有序或无序图像重建三维。

.. rst-class:: hero__title

COLMAP
======

.. raw:: html

   <div class="hero">
     <p class="hero__tagline">通用的运动恢复结构与多视图立体</p>
     <p class="hero__desc">
       COLMAP 是一套通用的运动恢复结构（SfM）和多视图立体（MVS）流程，
       带图形界面和命令行。它可以重建有序或无序的图像集合，并且免费开源。
     </p>
     <div class="hero__cta">
       <a class="hero__cta--primary" href="algorithms/index.html">算法实现</a>
       <a class="hero__cta--secondary" href="https://github.com/colmap/colmap">GitHub</a>
     </div>
   </div>

.. figure:: images/sparse.webp
   :class: hero__image
   :figclass: hero-figure
   :alt: 罗马市中心的稀疏重建。

   用 COLMAP 的 SfM 从罗马市中心约 2.1 万张照片重建的稀疏模型。


算法实现
--------

下面按代码里的实际调用顺序组织。每页写清入口、数据结构，以及函数真正做了什么。

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: 总览
      :link: algorithms/index
      :link-type: doc

      从图像到稀疏模型、稠密模型和定位的整条数据流，以及各模块对应的源码目录。

   .. grid-item-card:: 特征提取
      :link: algorithms/features
      :link-type: doc

      SIFT、ALIKED、LoMa 怎么选后端，关键点和描述子怎么写入数据库。

   .. grid-item-card:: 匹配与几何验证
      :link: algorithms/matching
      :link-type: doc

      图像对怎么生成，描述子怎么比，两视图几何和对应图怎么变成轨迹。

   .. grid-item-card:: 增量式重建
      :link: algorithms/incremental
      :link-type: doc

      初始像对、下一张图、绝对位姿、三角化、局部与全局光束法平差。

   .. grid-item-card:: 全局与分层重建
      :link: algorithms/global_sfm
      :link-type: doc

      旋转平均、全局定位、视图图标定，以及分层聚类后再合并。

   .. grid-item-card:: 光束法平差
      :link: algorithms/bundle_adjustment
      :link-type: doc

      优化哪些变量、用什么残差，Ceres 和 Caspar 各自接到哪里。

   .. grid-item-card:: 图像检索
      :link: algorithms/retrieval
      :link-type: doc

      词袋、倒排表、汉明嵌入，以及 vote-and-verify 空间验证。

   .. grid-item-card:: 多视图立体
      :link: algorithms/mvs
      :link-type: doc

      PatchMatch 深度、融合点云、Poisson 和 Delaunay 网格。

   .. grid-item-card:: 定位与高斯
      :link: algorithms/localization
      :link-type: doc

      SuperPoint、LightGlue、NetVLAD 怎么建图和注册，以及高斯怎么初始化、加密和用光度细化位姿。


开始使用
--------

1. 从本仓库源码编译 ``colmap``，或下载
   `预编译包 <https://github.com/colmap/colmap/releases>`_。
2. 准备一组重叠照片。
3. 从 :doc:`algorithms/index` 按数据流读各段算法的实现。


支持
----

提问用 `GitHub Discussions <https://github.com/colmap/colmap/discussions>`_。
缺陷和功能请求用 `GitHub issue <https://github.com/colmap/colmap>`_。


引用
----

研究中使用本项目时，请引用::

    @inproceedings{schoenberger2016sfm,
        author={Sch\"{o}nberger, Johannes Lutz and Frahm, Jan-Michael},
        title={Structure-from-Motion Revisited},
        booktitle={Conference on Computer Vision and Pattern Recognition (CVPR)},
        year={2016},
    }

    @inproceedings{schoenberger2016mvs,
        author={Sch\"{o}nberger, Johannes Lutz and Zheng, Enliang and Pollefeys, Marc and Frahm, Jan-Michael},
        title={Pixelwise View Selection for Unstructured Multi-View Stereo},
        booktitle={European Conference on Computer Vision (ECCV)},
        year={2016},
    }

使用全局 SfM（GLOMAP）时，请引用::

    @inproceedings{pan2024glomap,
        author={Pan, Linfei and Barath, Daniel and Pollefeys, Marc and Sch\"{o}nberger, Johannes Lutz},
        title={{Global Structure-from-Motion Revisited}},
        booktitle={European Conference on Computer Vision (ECCV)},
        year={2024},
    }

使用图像检索或词袋时，请引用::

    @inproceedings{schoenberger2016vote,
        author={Sch\"{o}nberger, Johannes Lutz and Price, True and Sattler, Torsten and Frahm, Jan-Michael and Pollefeys, Marc},
        title={A Vote-and-Verify Strategy for Fast Spatial Verification in Image Retrieval},
        booktitle={Asian Conference on Computer Vision (ACCV)},
        year={2016},
    }


致谢
----

COLMAP 最初由 `Johannes Schönberger <https://demuc.de/>`__ 编写，经费来自他的博士导师 Jan-Michael Frahm 和 Marc Pollefeys。
目前的核心维护者包括
`Johannes Schönberger <https://github.com/ahojnnes>`__、
`Paul-Edouard Sarlin <https://github.com/sarlinpe>`_ 和
`Shaohui Liu <https://github.com/B1ueber2y>`_。

PyCOLMAP 的 Python 绑定最初由
`Mihai Dusmanu <https://github.com/mihaidusmanu>`_、
`Philipp Lindenberger <https://github.com/Phil26AT>`_ 和
`Paul-Edouard Sarlin <https://github.com/sarlinpe>`_ 加入。

项目也来自社区的缺陷修复、改进、新功能和第三方工具。特别感谢 `Torsten Sattler <https://tsattler.github.io>`_。


.. toctree::
   :hidden:
   :caption: 算法实现
   :maxdepth: 1

   algorithms/index
   algorithms/features
   algorithms/matching
   algorithms/retrieval
   algorithms/incremental
   algorithms/global_sfm
   algorithms/bundle_adjustment
   algorithms/mvs
   algorithms/localization
