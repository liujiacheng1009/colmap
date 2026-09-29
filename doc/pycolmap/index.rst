.. _pycolmap/index:

PyCOLMAP
========

PyCOLMAP 把 COLMAP 的大部分能力暴露给 Python。

安装
----

Linux、macOS 和 Windows 的预编译 wheel 可以用 pip 安装::

   pip install pycolmap

每次发布都会自动构建并上传到 `PyPI
<https://pypi.org/project/pycolmap/>`_。
要使用 GPU，Linux 上另有针对 CUDA 12 构建的
`pycolmap-cuda12 <https://pypi.org/project/pycolmap-cuda12/>`_。

从源码构建 PyCOLMAP：

1. 先从源码编译并安装 COLMAP。
2. 再构建 PyCOLMAP：

   * Linux 和 macOS::

      python -m pip install .

   * Windows 上用 VCPKG 装好 COLMAP 后，在 PowerShell 里运行::

      python -m pip install . `
          --cmake.define.CMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
          --cmake.define.VCPKG_TARGET_TRIPLET="x64-windows"

代价函数等功能还需要同样方式安装 `PyCeres
<https://github.com/cvg/pyceres>`_，可以从 PyPI 安装，也可以从源码安装。

API
-----

.. toctree::
   :maxdepth: 2

   pycolmap
   cost_functions
