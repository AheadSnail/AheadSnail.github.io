---
layout: pager
title: 阅读 GpuImage 有感
date: 2020-12-16 11:03:16
tags: [Android,GpuImage,音视频]
description:  阅读 GpuImage 有感
---

### 概述

> 阅读 GpuImage 有感

<!--more-->

### 简介

GPUImage 毫无疑问是音视频项目里面必读工程了，它的侧重点在于渲染方面。有些公司的招聘要求上可能都会写明熟悉GPUImage ，重要性可见一斑。通过阅读 GPUImage 的源码，能够让你掌握 OpenGL 的渲染以及渲染链的搭建，同时工程里面很多特效 Shader 代码，通过阅读和实践这些 Shader 代码，能够让你掌握初步的 Shader 编写能力。比如常见的滤镜效果，在 GPUImage 就有现成的代码例子，掌握常见滤镜效果的代码编写。本篇文章简单记录下个人的理解,项目的地址为[GpuImage Github地址](https://github.com/cats-oss/android-gpuimage)

### 准备
首先将源码下载下来，需要将 library模块中 build.gradle文件中的 
```java
apply from: 'https://gist.githubusercontent.com/wasabeef/cf14805bee509baf7461974582f17d26/raw/bintray-v1.gradle'
apply from: 'https://gist.githubusercontent.com/wasabeef/cf14805bee509baf7461974582f17d26/raw/install-v1.gradle'
```
注释掉，目前这文件找不到，由于主要是影响打包，去掉也没有任何问题，具体的详细的源码分析可以参考下面的文章 [OpenGL 之 GPUImage 源码分析](https://mp.weixin.qq.com/s/Xc0r6PsxrT-dJ_W4-K7V0Q) 本文主要记录一下个人对此的理解

### 疑点
```java
在GPUImageFilterGroup 中处理多个效果的时候，在onDraw 中有这样的写法
public class GPUImageFilterGroup extends GPUImageFilter {
    public GPUImageFilterGroup(List<GPUImageFilter> filters) {
        ...
        //创建纹理缓冲
        glTextureBuffer = ByteBuffer.allocateDirect(TEXTURE_NO_ROTATION.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        glTextureBuffer.put(TEXTURE_NO_ROTATION).position(0);

        //获取到 执行 镜像处理的纹理 坐标 ,创建 执行 水平镜像的缓冲,这里首先获取到 TEXTURE_NO_ROTATION 对应的纹理，然后执行 镜像的变换操作
        //就是改变y轴坐标
        float[] flipTexture = TextureRotationUtil.getRotation(Rotation.NORMAL, false, true);
        glTextureFlipBuffer = ByteBuffer.allocateDirect(flipTexture.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        glTextureFlipBuffer.put(flipTexture).position(0);
    }
    ...
    public void onDraw(final int textureId, final FloatBuffer cubeBuffer, final FloatBuffer textureBuffer) {
        //先执行还未执行的方法
        runPendingOnDrawTasks();

        //如果还未初始化，直接返回
        if (!isInitialized() || frameBuffers == null || frameBufferTextures == null) {
            return;
        }

        if (mergedFilters != null) {
            int size = mergedFilters.size();
            //相机原始图像转换的纹理ID
            int previousTexture = textureId;
            //一个一个执行绘制操作
            for (int i = 0; i < size; i++) {
                GPUImageFilter filter = mergedFilters.get(i);
                //绑定对应的帧缓冲
                boolean isNotLast = i < size - 1;
                //如果不是最后一个滤镜，绘制到FrameBuffer上，如果是最后一个，就绘制到了屏幕上
                if (isNotLast) {
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, frameBuffers[i]);
                    GLES20.glClearColor(0, 0, 0, 0);
                }
                //滤镜绘制代码
                if (i == 0) {
                    //第一个滤镜绘制使用相机的原始图像纹理Id和参数传递过来的顶点以及纹理坐标，这个处理之后会得到正确的图片
                    filter.onDraw(previousTexture, cubeBuffer, textureBuffer);
                } else if (i == size - 1) {
                    //如果是最后一个,并且 size大小可以被2整除，则纹理坐标设置为 glTextureFlipBuffer ，这是做了水平镜像处理的
                    filter.onDraw(previousTexture, glCubeBuffer, (size % 2 == 0) ? glTextureFlipBuffer : glTextureBuffer);
                } else {
                    //中间的滤镜绘制在之前纹理基础上继续绘制，使用 mGLTextureBuffer纹理坐标
                    filter.onDraw(previousTexture, glCubeBuffer, glTextureBuffer);
                }
                if (isNotLast) {
                    //如果是最后一个绑定到屏幕上
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
                    previousTexture = frameBufferTextures[i];
                }
            }
        }
    }
    ...
}
其中  filter.onDraw(previousTexture, glCubeBuffer, (size % 2 == 0) ? glTextureFlipBuffer : glTextureBuffer); 这里根据size是否能被2整除来设置对应的纹理坐标
其中 glTextureBuffer 的纹理坐标为 TEXTURE_NO_ROTATION

//纹理坐标，这是以左上角为原点的坐标，并不是传统的以 左下角为原点的坐标
public static final float TEXTURE_NO_ROTATION[] = {
    0.0f, 1.0f,
    1.0f, 1.0f,
    0.0f, 0.0f,
    1.0f, 0.0f,
};

glTextureFlipBuffer 刚开始值为 TEXTURE_NO_ROTATION 后面执行了 flipVertical 变换(就是将纹理Y的坐标将1变成0,0变成1)，最终得到的纹理坐标为
public static final float TEXTURE_COORD_NO_ROTATION[] = {
    0.0f,0.0f, //图像的左下角
    1.0f,0.0f, //图像的右下角
    0.0f,1.0f, //图像的左下角
    1.0f,1.0f  //图像的右上角
};

而我们知道OpenGL中纹理坐标为 0到1之间，而且正常的纹理坐标的原点在左下角，类似下面的这张图
```
![结果显示](/uploads/GPUImage/纹理坐标.png)
```C++
而我们的计算机坐标原点是在左上角的，所以我们就需要做变换，也要以左上角为原点得到正确的纹理坐标，要不然以 默认的纹理坐标TEXTURE_COORD_NO_ROTATION 来显示的话，
会跟正确的图像倒置过来的效果为了更好的理解，写个简单的demo来验证下

//通过定义STB_IMAGE_IMPLEMENTATION，预处理器会修改头文件，让其只包含相关的函数定义源码，等于是将这个头文件变为一个 .cpp 文件了。现在只需要在你的程序中包含stb_image.h并编译就可以了。
#define STB_IMAGE_IMPLEMENTATION
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <stb_image.h>
#include <iostream>
#include <Shader.h>

void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow* window);

// settings
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;

int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE); // uncomment this statement to fix compilation on OS X
#endif

    GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
    if (window == NULL)
    {
        std::cout << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);
    glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        std::cout << "Failed to initialize GLAD" << std::endl;
        return -1;
    }

    Shader ourShader("4.1.texture.vs", "4.1.texture.fs");

    // ------------------------------------------------------------------  传统的以纹理坐标得到的坐标系，是以左下角为下标远点的
    float vertices[] = {
        // positions           // texture coords
        -1.0f, -1.0f, 0.0f,    0.0f, 0.0f, //图像的左下角
        1.0f, -1.0f, 0.0f,    1.0f, 0.0f, //图像的右下角
        -1.0f,  1.0f, 0.0f,    0.0f, 1.0f, //图像的左下角
        1.0f,  1.0f, 0.0f,    1.0f, 1.0f  //图像的右上角
    };
	
    unsigned int VBO, VAO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);

    glBindVertexArray(VAO);

    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

    // position attribute
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);
    // color attribute
    glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)(3 * sizeof(float)));
    glEnableVertexAttribArray(1);

    unsigned int texture2;
    int width, height, nrChannels;
	
    glGenTextures(1, &texture2);
    glBindTexture(GL_TEXTURE_2D, texture2);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);	// set texture wrapping to GL_REPEAT (default wrapping method)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    // load image, create texture and generate mipmaps
    unsigned char* data = stbi_load("awesomeface.png", &width, &height, &nrChannels, 0);
    if (data)
    {
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
        glGenerateMipmap(GL_TEXTURE_2D);
    }
    else
    {
        std::cout << "Failed to load texture" << std::endl;
    }
    stbi_image_free(data);

    // 别忘记在激活着色器前先设置uniform！
    ourShader.use(); // don't forget to activate/use the shader before setting uniforms!
    glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0);
	
   while (!glfwWindowShouldClose(window))
   {
        processInput(window);
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, texture2);

        ourShader.use();
        glBindVertexArray(VAO);
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }
    glDeleteVertexArrays(1, &VAO);
    glDeleteBuffers(1, &VBO);

    glfwTerminate();
    return 0;
}
void processInput(GLFWwindow* window)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}

void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
    glViewport(0, 0, width, height);
}

其中 4.1.texture.vs 顶点着色器的内容为  

#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;
out vec2 TexCoord;

void main()
{
    gl_Position = vec4(aPos, 1.0);
    TexCoord = vec2(aTexCoord.x, aTexCoord.y);
}

4.1.texture.fs 片段着色器的内容为

#version 330 core
out vec4 FragColor;
in vec2 TexCoord;

uniform sampler2D texture1;

void main()
{
    FragColor = texture(texture1, TexCoord);
}

我们刚开始使用的纹理坐标为 ,也就是纹理默认的左下角为原点的取值
float vertices[] = {
    // positions          // texture coords
    -1.0f, -1.0f, 0.0f,   0.0f, 0.0f, //图像的左下角
    1.0f, -1.0f, 0.0f,    1.0f, 0.0f, //图像的右下角
    -1.0f,  1.0f, 0.0f,   0.0f, 1.0f, //图像的左下角
    1.0f,  1.0f, 0.0f,    1.0f, 1.0f  //图像的右上角
};
```
awesomeface.png 原图显示的样子
![结果显示](/uploads/GPUImage/默认图片.png)
接下来运行，我们看下结果
![结果显示](/uploads/GPUImage/结果1.png)
可以看出是跟原图是倒置过来的,接下来试一下 以左上角为原点的纹理坐标

```C++
//由于 计算机的坐标系跟 纹理的坐标系是相反的，所以我们可以用 按照左上角为下标原点,这俩者的效果就是翻转过来的样子，接下来我们使用以左上角为原点得到的纹理坐标
float vertices1[] = {
    // positions           // texture coords
    -1.0f, -1.0f, 0.0f,    0.0f, 1.0f,
    1.0f, -1.0f, 0.0f,    1.0f, 1.0f,
    -1.0f,  1.0f, 0.0f,    0.0f, 0.0f,
    1.0f,  1.0f, 0.0f,    1.0f, 0.0f
};
```
![结果显示](/uploads/GPUImage/结果2.png)
可以看出我们的原图正确的显示过来了，

```C++
//水平镜像的效果  , 就是将 x 坐标的值由1变为0,0变为1 ，就跟镜子一样，因为改变x轴的坐标，这个变换是我们以上一个坐标(vertices1)变换过来的
float vertices2[] = {
    // positions           // texture coords
    -1.0f, -1.0f, 0.0f,    1.0f, 1.0f,
    1.0f, -1.0f, 0.0f,    0.0f, 1.0f,
    -1.0f,  1.0f, 0.0f,    1.0f, 0.0f,
    1.0f,  1.0f, 0.0f,    0.0f, 0.0f
};
```
![结果显示](/uploads/GPUImage/结果3.png)
可以看出我们的图片跟上一个图片做了镜像的处理，所以执行水平的变换，改变的是x坐标，能得到镜像的效果，
```C++
//垂直镜像的效果 ，就是倒置过来了，因为改变了y轴的坐标,这变换也是在上一个坐标((vertices1) 变换过来的
float vertices3[] = {
    // positions           // texture coords
    -1.0f, -1.0f, 0.0f,    0.0f, 0.0f,
    1.0f, -1.0f, 0.0f,    1.0f, 0.0f,
    -1.0f,  1.0f, 0.0f,    0.0f, 1.0f,
    1.0f,  1.0f, 0.0f,    1.0f, 1.0f
};	
```
![结果显示](/uploads/GPUImage/结果四.png)
可以看出我们的图片倒置过来了，通过改变y的坐标实现的效果

```C++
继续回到项目中 GPUImage中，在GPUImage中默认的纹理坐标是以左上角为远点的，下面是默认的纹理坐标
public static final float TEXTURE_NO_ROTATION[] = {
    0.0f, 1.0f,
    1.0f, 1.0f,
    0.0f, 0.0f,
    1.0f, 0.0f,
};
所以就有点类似原本就帮我们做了一次倒置的效果，而这也是原本就要的，所以为了方便，就直接在这个基础上做修改了，继续回到
filter.onDraw(previousTexture, glCubeBuffer, (size % 2 == 0) ? glTextureFlipBuffer : glTextureBuffer); 这里根据size是否能被2整除来设置对应的纹理坐标

//纹理坐标，这是以左上角为原点的坐标，并不是传统的以 左下角为原点的坐标
public static final float TEXTURE_NO_ROTATION[] = {
    0.0f, 1.0f,
    1.0f, 1.0f,
    0.0f, 0.0f,
    1.0f, 0.0f,
};

glTextureFlipBuffer 刚开始值为 TEXTURE_NO_ROTATION 后面执行了 flipVertical 变换(就是将纹理Y的坐标将1变成0,0变成1)，最终得到的纹理坐标为
public static final float TEXTURE_COORD_NO_ROTATION[] = {
    0.0f,0.0f, //图像的左下角
    1.0f,0.0f, //图像的右下角
    0.0f,1.0f, //图像的左下角
    1.0f,1.0f  //图像的右上角
};

所以使用 TEXTURE_NO_ROTATION 的纹理坐标实质上是将图像进行了上下翻转，两次调用TEXTURE_NO_ROTATION纹理坐标时，又将图像复原了，所以对于奇数次的话，
就继续使用 glTextureBuffer这个默认就是再倒过来，如果是偶数的话，因为前面一次已经将图片倒置过来了，所以我们可以不用再倒置了，
我们就可以使用 TEXTURE_COORD_NO_ROTATION，这是默认的纹理坐标，所以就能保持上次的效果了
```

### 性能问题
在使用GPUImage的时候，发现开启摄像机过了一会手机就会发烫，下面分析下性能问题
```C++
首先在 Camera2Loader中在获取到预览数据的时候，这里首先要执行一次数据的变换 image.generateNV21Data(),这里加上打印的时间
setOnImageAvailableListener({ reader ->
    //如果为空，从这个 setOnImageAvailableListener lambda中局部返回
    val image = reader?.acquireNextImage() ?: return@setOnImageAvailableListener
    //不为空，回调通知回去
    val start: Long = System.currentTimeMillis()
    Log.d("Camera2Loader", "generateNV21Data before$start")
    val result = image.generateNV21Data()
    Log.d("Camera2Loader", "generateNV21Data after${System.currentTimeMillis() - start}")
    onPreviewFrame?.invoke(result, image.width, image.height)
    image.close()
}, null)

还有一次是在将预览的数据YUV转成 RGB的时候
public void onPreviewFrame(final byte[] data, final int width, final int height) {
    //创建预览界面内存大小
    if (glRgbBuffer == null) {
        glRgbBuffer = IntBuffer.allocate(width * height);
    }
    //为空的时候才能添加
    if (runOnDraw.isEmpty()) {
        //通过任务的方式来进行
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                long startTime  = System.currentTimeMillis();
                //Log.d("GPUImageRenderer","YUVtoRBGA before" + System.currentTimeMillis());
                //首先将yuv转成 rgba格式,输出的格式到 glRgbBuffer
                GPUImageNativeLibrary.YUVtoRBGA(data, width, height, glRgbBuffer.array());
                Log.d("GPUImageRenderer","YUVtoRBGA after" + (System.currentTimeMillis() - startTime));

                //获得预览画面的纹理,将相机的纹理画面保存到 glTextureId 变量中
                glTextureId = OpenGlUtils.loadTexture(glRgbBuffer, width, height, glTextureId);

                //判断宽度是否有改变，有的话，要重新调整缩放的比率
                if (imageWidth != width) {
                    imageWidth = width;
                    imageHeight = height;
                    adjustImageScaling();
                }
            }
        });
    }
}
这里也加上打印的时间，下面看看允许的结果
```
![结果显示](/uploads/GPUImage/转成NV21消耗.png)
![结果显示](/uploads/GPUImage/YUV转成RGB的消耗.png)
可以看出俩者的时间大致在10-11毫秒,而我们的相机正常是30帧的话，每一帧的时间只有33毫秒，假设在数据的转换上就耗费了11毫秒，那么剩下的时间根本就不多了，这还是只是转换，绘制根本还没执行而且由于这个数据的转换是在CPU上进行的，转换操作都是计算，所以导致CPU繁忙，所以会发烫，改进的话，我们在C++一层创建一个  GL_TEXTURE_EXTERNAL_OES Android特有的纹理对象，这个可以直接将YUV的数据转成RGB的纹理接受过来，而且是在GPU上

```C++
在 GPUImageFilterGroup 中，在处理多个滤镜效果的时候，内部的做法是通过创建多个帧缓冲对象 FBO，每个特效对应一个帧缓冲对象，
    public void onOutputSizeChanged(final int width, final int height) {
        super.onOutputSizeChanged(width, height);
        if (frameBuffers != null) {
            destroyFramebuffers();
        }

        //将界面大小传递下去
        int size = filters.size();
        for (int i = 0; i < size; i++) {
            filters.get(i).onOutputSizeChanged(width, height);
        }

        if (mergedFilters != null && mergedFilters.size() > 0) {
            //创建跟 mergedFilters 对应的纹理缓冲对象
            size = mergedFilters.size();
            frameBuffers = new int[size - 1];
            frameBufferTextures = new int[size - 1];

            //注意这里生成了 多个帧缓冲对象
            for (int i = 0; i < size - 1; i++) {
                //创建帧缓冲对象，帧缓冲对象通过挂载纹理可以将内容输出到对应的纹理上
                GLES20.glGenFramebuffers(1, frameBuffers, i);
                //创建挂载的纹理
                GLES20.glGenTextures(1, frameBufferTextures, i);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, frameBufferTextures[i]);
                //设置纹理的环绕方式和过滤方式
                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, width, height, 0, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, null);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);

                //绑定纹理到 当前生成的帧缓冲
                GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, frameBuffers[i]);
                GLES20.glFramebufferTexture2D(GLES20.GL_FRAMEBUFFER, GLES20.GL_COLOR_ATTACHMENT0, GLES20.GL_TEXTURE_2D, frameBufferTextures[i], 0);

                //解绑操作
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
                GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
            }
        }
    }
    在多个特效之间做处理的时候，频繁的绑定帧缓冲和解绑
    public void onDraw(final int textureId, final FloatBuffer cubeBuffer, final FloatBuffer textureBuffer) {
        //先执行还未执行的方法
        runPendingOnDrawTasks();

        //如果还未初始化，直接返回
        if (!isInitialized() || frameBuffers == null || frameBufferTextures == null) {
            return;
        }

        if (mergedFilters != null) {
            int size = mergedFilters.size();
            //相机原始图像转换的纹理ID
            int previousTexture = textureId;
            //一个一个执行绘制操作
            for (int i = 0; i < size; i++) {
                GPUImageFilter filter = mergedFilters.get(i);
                //绑定对应的帧缓冲
                boolean isNotLast = i < size - 1;
                //如果不是最后一个滤镜，绘制到FrameBuffer上，如果是最后一个，就绘制到了屏幕上
                if (isNotLast) {
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, frameBuffers[i]);
                    GLES20.glClearColor(0, 0, 0, 0);
                }
                //滤镜绘制代码
                if (i == 0) {
                    //第一个滤镜绘制使用相机的原始图像纹理Id和参数传递过来的顶点以及纹理坐标，这个处理之后会得到正确的图片
                    filter.onDraw(previousTexture, cubeBuffer, textureBuffer);
                } else if (i == size - 1) {
                    //如果是最后一个,并且 size大小可以被2整除，则纹理坐标设置为 glTextureFlipBuffer ，这是做了水平镜像处理的
                    filter.onDraw(previousTexture, glCubeBuffer, (size % 2 == 0) ? glTextureFlipBuffer : glTextureBuffer);
                } else {
                    //中间的滤镜绘制在之前纹理基础上继续绘制，使用 mGLTextureBuffer纹理坐标
                    filter.onDraw(previousTexture, glCubeBuffer, glTextureBuffer);
                }
                if (isNotLast) {
                    //如果是最后一个绑定到屏幕上
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
                    previousTexture = frameBufferTextures[i];
                }
            }
        }
    }
    ...
}
频繁的绑定帧缓冲和解绑效率会有影响，官方建议是通过创建一个帧缓冲对象，通过频繁的 attach纹理，detach纹理来做到多个效果之间的切换处理
```

### 体验问题
1.GPUImage在切换前置摄像头的时候，会倒置过来，所以还要做垂直的变换操作
2.在切换的过程中会看到原摄像机的内容，就是还没有执行正常处理，比如旋转的角度，倒置处理之后的画面，这是由于 GPUImageRenderer 中 glTextureId 保存的是摄像机原本的画面内容并不是处理完之后，比如旋转，倒置后的结果
```C++
	public void onPreviewFrame(final byte[] data, final int width, final int height) {
        //创建预览界面内存大小
        if (glRgbBuffer == null) {
            glRgbBuffer = IntBuffer.allocate(width * height);
        }
        //为空的时候才能添加
        if (runOnDraw.isEmpty()) {
            //通过任务的方式来进行
            runOnDraw(new Runnable() {
                @Override
                public void run() {
                    long startTime  = System.currentTimeMillis();
                    //Log.d("GPUImageRenderer","YUVtoRBGA before" + System.currentTimeMillis());
                    //首先将yuv转成 rgba格式,输出的格式到 glRgbBuffer
                    GPUImageNativeLibrary.YUVtoRBGA(data, width, height, glRgbBuffer.array());
                    Log.d("GPUImageRenderer","YUVtoRBGA after" + (System.currentTimeMillis() - startTime));

                    //获得预览画面的纹理,将相机的纹理画面保存到 glTextureId 变量中
                    glTextureId = OpenGlUtils.loadTexture(glRgbBuffer, width, height, glTextureId);

                    //判断宽度是否有改变，有的话，要重新调整缩放的比率
                    if (imageWidth != width) {
                        imageWidth = width;
                        imageHeight = height;
                        adjustImageScaling();
                    }
                }
            });
        }
    }
    public void onDrawFrame(final GL10 gl) {
        //首先清除颜色
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);
        //运行运行中的Runnable队列
        //GLSurfaceView中，对于GL环境的操作，出queueEvent是将事件放入队列中，到GL线程中执行外，其他方法基本都是在主线程
        //（也可以是其他线程，非当前GLSurfaceView实例的GL线程）中修改某个状态值，然后取消GL线程的等待，在GL线程中根据状态值作相应的操作，
        // 并在操作后反馈给调用方法的那个线程，当然有的方法也不需要反馈。
        runAll(runOnDraw);
        //执行效果器的绘制,由于前面一步已经将相机的画面已经绘制到了 glTextureId 纹理上面了，接下来我们就可以出来效果了
        filter.onDraw(glTextureId, glCubeBuffer, glTextureBuffer);
        //运行 绘制后 Runnable 队列
        runAll(runOnDrawEnd);
        //调用updateTexImage 来完成数据到 纹理上面
        if (surfaceTexture != null) {
            surfaceTexture.updateTexImage();
        }
    }
```

### 总结
总体而言，这个项目还是非常不错的，非常有参考，学习的意义,虽然有点小瑕疵，但不足以掩盖他的牛逼，尤其是他支持的各种各样的特效处理


### 参考链接
1. [android-gpuimage](https://github.com/cats-oss/android-gpuimage)
2. [android GLSurfaceView渲染模式](https://blog.csdn.net/bzlj2912009596/article/details/78348750)
3. [Android Camera2 简介](https://www.jianshu.com/p/23e8789fbc10)
4. [GLSL 内建函数](https://www.cnblogs.com/kex1n/p/3941765.html)
5. [OpenGL 之 GPUImage 源码分析](https://mp.weixin.qq.com/s/Xc0r6PsxrT-dJ_W4-K7V0Q)
5. [LearnOpenGL](https://learnopengl-cn.github.io/)














