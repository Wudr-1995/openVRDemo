# 用于推流的Demo介绍

## 程序框架
- 这一demo是在openvr的官方sample基础上添加修改而来。主要基于openvr和opengl，opengl用于实现渲染，openvr则用于与VR眼镜通讯，将渲染的结果显示在眼镜上。
- 程序中主要修改的是opengl渲染实现的部分，重点参考了这一opengl[教程网站](https://learnopengl.com/ "With a Title")
- 程序分为两部demo.cc和demo.h，其中demo.h声明了两个类，CGRenderModel和CMainApplication。其中CMainApplication是主要修改的部分，用于返回两个眼睛的坐标->将输入图像作为纹理读入->在顶点信息确定的范围内渲染->并将渲染结果推流到VR眼镜上
