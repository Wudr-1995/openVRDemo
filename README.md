# 用于推流的Demo介绍

## 程序框架
- 这一demo是在openVR的官方sample基础上添加修改而来。主要基于openVR和openGL，openGL用于实现渲染，openVR则用于与VR眼镜通讯，将渲染的结果显示在眼镜上。
- 程序中主要修改的是openGL渲染实现的部分，重点参考了这一openGL[教程网站](https://learnopengl.com/ "With a Title")
- 程序分为两部demo.cc和demo.h，其中demo.h声明了两个类，CGRenderModel和CMainApplication。其中CMainApplication是主要修改的部分，用于返回两个眼睛的坐标->将输入图像作为纹理读入->在顶点信息确定的范围内渲染->并将渲染结果推流到VR眼镜上

## openGL的简要介绍
openGL是一个用于显示图形图像的API，渲染流程是这样的

准备好vertex信息->将vertex信息绑定在确定ID的buffer上->通过编译好的shader程序渲染得到由vertex确定的三角形组成的网格->并将准备好的纹理在网格中显示

## CMainApplication
在demo的实现中，也是按照openGL的流程进行的，与vr有关的主要是最后通过openVR的接口函数将准备好的纹理推流到设备上去。

### openGL vertex渲染部分
与我们目标相关的vertex的部分在这里
c++ `
void CMainApplication::SetupScene() {
	if ( !m_pHMD )
		return;

	float vertics[] = {
		-0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 0.0f, 1.0f,
		 0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 1.0f, 1.0f,
		 0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 1.0f, 0.0f,
		 0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 1.0f, 0.0f,
		-0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 0.0f, 0.0f,
		-0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 0.0f, 1.0f
	};

	m_uiVertcount = 6;

	glGenVertexArrays( 1, &m_unSceneVAO );
	glBindVertexArray( m_unSceneVAO );

	glGenBuffers( 1, &m_glSceneVertBuffer );
	glBindBuffer( GL_ARRAY_BUFFER, m_glSceneVertBuffer );
	glBufferData( GL_ARRAY_BUFFER, sizeof(float) * 30, vertics, GL_STATIC_DRAW);

	uintptr_t offset = 0;

	glEnableVertexAttribArray( 0 );
	glVertexAttribPointer( 0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (const void *)offset);

	offset += sizeof(float) * 3;
	glEnableVertexAttribArray( 1 );
	glVertexAttribPointer( 1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (const void *)offset);
}
`
