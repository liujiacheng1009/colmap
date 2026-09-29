相机模型
========

COLMAP 实现了复杂度不同的多种相机模型。事先不知道内参时，应选能描述畸变、又尽量简单的模型：

- ``SIMPLE_PINHOLE``、``PINHOLE``：图像已经去畸变时用。分别有 1 个和 2 个焦距参数。
  即使图像已去畸变，COLMAP 仍可能用更复杂的模型去改进内参。
- ``SIMPLE_RADIAL``、``RADIAL``：内参未知、且每张图标定不同时优先用，例如网上照片。
  它们是 ``OPENCV`` 的简化，只建模径向畸变，分别有 1 个和 2 个畸变参数。
- ``OPENCV``、``FULL_OPENCV``：事先知道标定参数时用。多张图共享内参时，也可以让 COLMAP 估计这些参数。
  若每张图各自一套内参，自动估计多半会失败。
- ``SIMPLE_RADIAL_FISHEYE``、``RADIAL_FISHEYE``、``OPENCV_FISHEYE``、``FOV``、
  ``THIN_PRISM_FISHEYE``、``RAD_TAN_THIN_PRISM_FISHEYE``：鱼眼镜头用这些模型。
  其他模型描述不了鱼眼畸变。``FOV`` 用于 Google Project Tango（不要把 ``omega`` 初始化为 0）。
- ``SIMPLE_FISHEYE``、``FISHEYE``：等距投影的鱼眼，畸变可以忽略或已经校正时用。
  投影为 theta = atan(r)，没有畸变参数。``SIMPLE_FISHEYE`` 只有一个焦距 f，
  ``FISHEYE`` 有 fx、fy。
- ``SIMPLE_DIVISION``、``DIVISION``：事先知道标定参数时用。和 ``SIMPLE_RADIAL``、``RADIAL`` 类似，
  能描述简单径向畸变。畸变较小时，这两种模型在局部一阶等价。
- ``EUCM``：广角鱼眼和折反射系统用。在针孔参数之外，用两个参数表示径向畸变。

在模型查看器里双击图像，或导出模型后打开 ``cameras.txt``，可以查看估计出的内参。

投影
----

透视相机模型把相机坐标系中的三维点变成像素坐标，分三步：透视除法、畸变、内参变换（焦距和主点）。
COLMAP 的像素以角点为原点，左上角像素的中心是 ``(0.5, 0.5)``\（见 :doc:`database`）。

以 ``SIMPLE_RADIAL``\（参数 ``f, cx, cy, k``）为例。相机坐标系看向正 :math:`Z` 轴，
点 :math:`(X, Y, Z)` 的投影如下：

1. 透视除法，投到归一化像平面：

   .. math::

       u = X / Z, \qquad v = Y / Z

2. 径向畸变，:math:`r^2 = u^2 + v^2`：

   .. math::

       u' = u \, (1 + k \, r^2), \qquad v' = v \, (1 + k \, r^2)

3. 乘焦距、加主点，得到像素坐标：

   .. math::

       x = f \, u' + c_x, \qquad y = f \, v' + c_y

逆映射（像素到归一化射线）先减主点、除以焦距，再迭代去掉畸变。

其他透视模型也是这三步，差别只在焦距个数（共用一个 ``f``，或分开的 ``fx``、``fy``）
和畸变函数。例如 ``RADIAL`` 多一个径向项 ``k2``，``OPENCV`` 多切向项 ``p1, p2``\。
鱼眼模型则把透视除法换成等距投影。每个模型的参数列表见其 ``params_info`` 字符串，
定义在相机模型头文件：
https://github.com/colmap/colmap/blob/main/src/colmap/sensor/models.h

配置
----

要得到好的重建，可能需要换几种相机模型试。重建失败，且焦距或畸变系数明显不对，
通常是模型过于复杂。反过来，如果 COLMAP 反复做很多次局部和全局光束法平差，
通常是模型过于简单，畸变没有被充分描述。

也可以让多张图像共享内参，结果会更稳
（见 :ref:`共享相机内参 <faq-share-intrinsics>`），
或在重建过程中固定内参
（见 :ref:`固定相机内参 <faq-fix-intrinsics>`）。
