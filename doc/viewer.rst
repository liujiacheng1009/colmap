:html_theme.sidebar_secondary.remove: true
:og:description: 在浏览器里查看 COLMAP 稀疏重建。交互在本地完成，文件不会上传。

.. meta::
   :description: 在浏览器里用 Three.js 查看 COLMAP 二进制稀疏重建。交互在本地完成，文件不会上传。

三维查看器
==========

在浏览器中直接打开 COLMAP 的二进制稀疏重建。处理在本地进行，重建文件和图像不会上传。

三维查看器沿用原生 :ref:`图形界面 <gui>` 的显示约定和模型操作，包括导航、选择点和图像、
查看观测，以及调整点和相机的大小。它是只读查看工具，功能少于原生界面：
不运行重建、不编辑模型，也不显示稠密点云和网格。

.. raw:: html

   <link rel="stylesheet" href="_static/viewer/viewer.css">
   <div id="colmap-viewer-root">
     <p>当前文档构建里没有交互式查看器。</p>
     <noscript>此查看器需要 JavaScript。</noscript>
   </div>
   <script type="module" src="_static/viewer/viewer.js"></script>


支持的输入
----------

拖入包含 ``cameras.bin``、``images.bin`` 和 ``points3D.bin`` 的稀疏模型文件夹，
或拖入包含一个或多个稀疏模型的工作目录。当前带 ``rigs.bin``、``frames.bin`` 的重建，
以及旧版二进制重建都可以。找到多个模型时，在工具栏里选一个。

要查看图像观测和重投影，工作目录里还要有原始图像树。模型加载后也可以再拖入图像文件夹。
图像路径按 ``images.bin`` 里记录的相对名称匹配。


操作
----

- **旋转：** 按住左键拖动。
- **平移：** 按住右键拖动。
- **缩放：** 滚轮。
- **点大小：** <CTRL> 加滚轮（Mac 为 <CMD>），或用工具栏。
- **相机大小：** <ALT> 加滚轮，或用工具栏。
- **选择：** 双击点或相机。
- **清除选择：** 双击背景，或用工具栏。

查看器需要支持 WebGL2 的当前桌面浏览器。浏览器不支持拖入文件夹时，可以用选择文件夹作为替代。


复用查看器
----------

查看器也是一个 ES 模块组件。在 ``doc`` 目录执行 ``npm run build``，
把完整的 ``_static/viewer`` 目录复制出去，使解析 worker 与模块放在一起，并同时引入生成的资源：

.. code-block:: html

   <link rel="stylesheet" href="/viewer/viewer.css">
   <div id="my-viewer"></div>
   <input id="model-folder" type="file" webkitdirectory multiple>
   <script type="module">
     import {mountColmapViewer} from "/viewer/component.js";

     const viewer = mountColmapViewer(document.querySelector("#my-viewer"));
     document.querySelector("#model-folder").addEventListener("change", async event => {
       const entries = [...event.target.files].map(file => ({
         path: file.webkitRelativePath || file.name,
         file,
       }));
       await viewer.load(entries);
     });

     // viewer.load() also accepts an already parsed Reconstruction object.
     // Call viewer.clear() or viewer.dispose() when appropriate.
   </script>

每个 ``LocalFile`` 的形状是 ``{path, file}``\。``path`` 是相对路径，``file`` 是浏览器的 ``File`` 对象。
同一页可以放多个组件实例。TypeScript 源码还导出挂载设置和生命周期类型。

把组件嵌到其他网站时，请在页面上标明来自 `COLMAP 项目 <https://colmap.github.io/>`_，
并在网站的法律声明或第三方声明中附上完整的 :doc:`COLMAP 新 BSD 许可文本 <license>`\。
同时保留打包进来的 `Three.js 许可说明 <_static/viewer-licenses.txt>`_。
