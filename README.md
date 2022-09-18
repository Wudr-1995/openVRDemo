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

### openGL vertex渲染
与VR眼镜显示相关的vertex的初始化部分
```c++
void CMainApplication::SetupScene() {
	if ( !m_pHMD )
		return;

	/*
	 * vertex
	 * six vector (one line one vector) in total
	 * the first three values in a vector are the coordinates of vertex
	 * the last two values are the UV coordinates
	 * modify the first three values to modify the render performance of the image in the VR glasses
	 */
	float vertics[] = {
		-0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 0.0f, 1.0f,
		 0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 1.0f, 1.0f,
		 0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 1.0f, 0.0f,
		 0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 1.0f, 0.0f,
		-0.8f * 1080.0f / 1080.0f,  0.8f, -0.0f, 0.0f, 0.0f,
		-0.8f * 1080.0f / 1080.0f, -0.8f, -0.0f, 0.0f, 1.0f
	};

	m_uiVertcount = 6;

	// generate the vertex array, m_unSceneVAO is the ID of VAO
	glGenVertexArrays( 1, &m_unSceneVAO );
	glBindVertexArray( m_unSceneVAO );

	/*
	 * generate the buffer, m_glSceneVertBuffer is the buffer ID
	 * bind the buffer with the defaut buffer GL_ARRAY_BUFFER
	 * and load the vertex data into the buffer
	 */
	glGenBuffers( 1, &m_glSceneVertBuffer );
	glBindBuffer( GL_ARRAY_BUFFER, m_glSceneVertBuffer );
	glBufferData( GL_ARRAY_BUFFER, sizeof(float) * 30, vertics, GL_STATIC_DRAW);

	uintptr_t offset = 0;

	// set the rule of vector reading
	glEnableVertexAttribArray( 0 );
	glVertexAttribPointer( 0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (const void *)offset);

	offset += sizeof(float) * 3;
	glEnableVertexAttribArray( 1 );
	glVertexAttribPointer( 1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (const void *)offset);
}
```

### 纹理初始化
以左眼纹理初始化为例，右眼与之方法相同，命名类似
```c++
bool CMainApplication::SetupLeftTexturemaps()
{
	std::string sExecutableDirectory = Path_StripFilename( Path_GetExecutablePath() );

	// load a random image (this step maybe skip)
	std::string strFullPath = Path_MakeAbsolute( "../cube_texture.png", sExecutableDirectory );

	std::vector<unsigned char> imageRGBA;
	unsigned nImageWidth, nImageHeight;
	unsigned nError = lodepng::decode( imageRGBA, nImageWidth, nImageHeight, strFullPath.c_str() );

	if ( nError != 0 )
		return false;

	// generate the ID of left texture buffer and bind it with the default buffer GL_TEXTURE_2D
	glGenTextures(1, &m_iLeftTex );
	glBindTexture( GL_TEXTURE_2D, m_iLeftTex );

	// input the image to initialize the texture, the input also can be set as NULL
	glTexImage2D( GL_TEXTURE_2D, 0, GL_RGBA, nImageWidth, nImageHeight,
		0, GL_RGBA, GL_UNSIGNED_BYTE, &imageRGBA[0] );

	// some setting of texture features
	glGenerateMipmap(GL_TEXTURE_2D);
	glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE );
	glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE );
	glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR );
	glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR );

	GLfloat fLargest;
	glGetFloatv(GL_MAX_TEXTURE_MAX_ANISOTROPY_EXT, &fLargest);
	glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MAX_ANISOTROPY_EXT, fLargest);

	// release the default buffer, necessary
	glBindTexture( GL_TEXTURE_2D, 0 );

	return ( m_iLeftTex != 0 );
}
```

### 图像输入与产生纹理
以左眼纹理刷新为例，右眼与之方法相同，命名类似
```c++
bool CMainApplication::RefreshLeftTexturemaps(unsigned char* image, int w, int h) {
	/*
	 * image is the first address of a char array, the format of the array is r, g, b, r, g, b, ...
	 * w is the width of the image
	 * h is the high of the image
	 */
	if (!image) {
		dprintf("The input image data is NULL.");
		return false;
	}

	// bind the buffer ID of left texture with the default buffer GL_TEXTURE_2D
	glBindTexture(GL_TEXTURE_2D, m_iLeftTex);

	unsigned nImageWidth, nImageHeight;
	nImageWidth = w;
	nImageHeight = h;

	// load the image as texture
	glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, nImageWidth, nImageHeight, 0, GL_RGB, GL_UNSIGNED_BYTE, image);
	glGenerateMipmap(GL_TEXTURE_2D);
	glBindTexture(GL_TEXTURE_2D, 0);
	return true;
}
```
