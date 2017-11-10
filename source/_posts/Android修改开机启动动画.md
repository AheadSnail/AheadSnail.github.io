---
layout: pager
title: Android修改开机启动动画
date: 2017-10-29 22:55:31
tags: [Android,开机动画]
description: Android修改开机启动动画
---

Android修改开机启动动画
<!--more-->

**Android修改开机启动动画，这边对应的为android源码6.0**
===

```
手机开机会启动init.rc脚本文件，会加载编译好的init文件，这俩个可执行的文件，可以在android里面找到
init文件是 android源码中的/system/core/init文件里面编译的产物，里面提供了Android.mk文件
函数的入口为init.cpp中的main函数,其中mian函数里面有一句这样的代码 if(execv(path, args) == -1)
就会开启对应的android源码中/frameworkers/base/cmds目录下的所有的可执行的文件，注意这个目录下的文件夹都可以编译成一个可以执行的文件
其中包括了开启虚拟机的文件bootanimation文件夹，里面也提供了Android.mk文件，对应的编译文件bootanimation_main也是含有main函数入口的
```

```java
int main()
{
    setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);

    char value[PROPERTY_VALUE_MAX];
    property_get("debug.sf.nobootanimation", value, "0");
    int noBootAnimation = atoi(value);
    ALOGI_IF(noBootAnimation,  "boot animation disabled");
    if (!noBootAnimation) {
		线程池
        sp<ProcessState> proc(ProcessState::self());
        ProcessState::self()->startThreadPool();

        // create the boot animation object
		BootAnimation对应的BootAnimation.cpp文件，本身是继承Thread的，可以交给线程池使用
        sp<BootAnimation> boot = new BootAnimation();
        IPCThreadState::self()->joinThreadPool();

    }
    return 0;
}
本身是继承Thread的,所以当线程池运行这个线程的时候，就会执行run 方法，这里对应的readyToRun
BootAnimation::BootAnimation() : Thread(false), mZip(NULL)
{
    mSession = new SurfaceComposerClient();
}

#define OEM_BOOTANIMATION_FILE "/oem/media/bootanimation.zip"
#define SYSTEM_BOOTANIMATION_FILE "/system/media/bootanimation.zip"
#define SYSTEM_ENCRYPTED_BOOTANIMATION_FILE "/system/media/bootanimation-encrypted.zip"

status_t BootAnimation::readyToRun() {
	.....
    ZipFileRO* zipFile = NULL;
	所以这里要去加载zip的顺序为 SYSTEM_BOOTANIMATION_FILE   OEM_BOOTANIMATION_FILE   SYSTEM_ENCRYPTED_BOOTANIMATION_FILE
    if ((encryptedAnimation &&
            (access(SYSTEM_ENCRYPTED_BOOTANIMATION_FILE, R_OK) == 0) &&
            ((zipFile = ZipFileRO::open(SYSTEM_ENCRYPTED_BOOTANIMATION_FILE)) != NULL)) ||

            ((access(OEM_BOOTANIMATION_FILE, R_OK) == 0) &&
            ((zipFile = ZipFileRO::open(OEM_BOOTANIMATION_FILE)) != NULL)) ||

            ((access(SYSTEM_BOOTANIMATION_FILE, R_OK) == 0) &&
            ((zipFile = ZipFileRO::open(SYSTEM_BOOTANIMATION_FILE)) != NULL))) {
			
			如果上面的路径有一个存在的时候，就会给mZip变量赋值,要不然就为null
        mZip = zipFile;
    }

    return NO_ERROR;
}

```

之后会执行下面的方法
```java
bool BootAnimation::threadLoop()
{
    bool r;
    // We have no bootanimation file, so we use the stock android logo
    // animation.  
	如果mZip为null，说明不存在上面所说的三个路径,这里就会去加载默认的,如果我们有存在上面的三个路径里面有对应的文件，就不会去加载默认的
    if (mZip == NULL) {
        r = android();
    } else {
        r = movie();
    }

    eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
    eglDestroyContext(mDisplay, mContext);
    eglDestroySurface(mDisplay, mSurface);
    mFlingerSurface.clear();
    mFlingerSurfaceControl.clear();
    eglTerminate(mDisplay);
    IPCThreadState::self()->stopProcess();
    return r;
}

r = android();函数的实现为:
bool BootAnimation::android()
{
	这个图片对应的就为手机开机的时候，显示的灰色的android的图标
    initTexture(&mAndroid[0], mAssets, "images/android-logo-mask.png");
	这个图片对应的就为手机开机的是时候，显示的亮色的android的图标
    initTexture(&mAndroid[1], mAssets, "images/android-logo-shine.png");

    // clear screen
    glShadeModel(GL_FLAT);
    glDisable(GL_DITHER);
    glDisable(GL_SCISSOR_TEST);
    glClearColor(0,0,0,1);
    glClear(GL_COLOR_BUFFER_BIT);
    eglSwapBuffers(mDisplay, mSurface);

    glEnable(GL_TEXTURE_2D);
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);

    const GLint xc = (mWidth  - mAndroid[0].w) / 2;
    const GLint yc = (mHeight - mAndroid[0].h) / 2;
    const Rect updateRect(xc, yc, xc + mAndroid[0].w, yc + mAndroid[0].h);

    glScissor(updateRect.left, mHeight - updateRect.bottom, updateRect.width(),
            updateRect.height());

    // Blend state
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);

    const nsecs_t startTime = systemTime();
    do {
        nsecs_t now = systemTime();
        double time = now - startTime;
        float t = 4.0f * float(time / us2ns(16667)) / mAndroid[1].w;
        GLint offset = (1 - (t - floorf(t))) * mAndroid[1].w;
        GLint x = xc - offset;

        glDisable(GL_SCISSOR_TEST);
        glClear(GL_COLOR_BUFFER_BIT);

        glEnable(GL_SCISSOR_TEST);
        glDisable(GL_BLEND);
        glBindTexture(GL_TEXTURE_2D, mAndroid[1].name);
        glDrawTexiOES(x,                 yc, 0, mAndroid[1].w, mAndroid[1].h);
        glDrawTexiOES(x + mAndroid[1].w, yc, 0, mAndroid[1].w, mAndroid[1].h);

        glEnable(GL_BLEND);
        glBindTexture(GL_TEXTURE_2D, mAndroid[0].name);
        glDrawTexiOES(xc, yc, 0, mAndroid[0].w, mAndroid[0].h);

        EGLBoolean res = eglSwapBuffers(mDisplay, mSurface);
        if (res == EGL_FALSE)
            break;

        // 12fps: don't animate too fast to preserve CPU
        const nsecs_t sleepTime = 83333 - ns2us(systemTime() - now);
        if (sleepTime > 0)
            usleep(sleepTime);

        checkExit();
    } while (!exitPending());

    glDeleteTextures(1, &mAndroid[0].name);
    glDeleteTextures(1, &mAndroid[1].name);
    return false;
}
```
这个图片对应的就为手机开机的时候，显示的灰色的android的图标
initTexture(&mAndroid[0], mAssets, "images/android-logo-mask.png");
![结果显示](/uploads/android启动页暗.png)

这个图片对应的就为手机开机的是时候，显示的亮色的android的图标
initTexture(&mAndroid[1], mAssets, "images/android-logo-shine.png")
![结果显示](/uploads/android启动页亮.png)


**加载自定义的动画**
===

首先准备bootanimation.zip压缩文件，里面的内容为：
![结果显示](/uploads/Bootanimation文件的内容.png)

folder1里面的内容为开机的动画图片，要注意图片的命名要按照这样的来
![结果显示](/uploads/bootanimationFolder1.png)

```
desc文件写了动画显示的属性等
540 960 5
p 0 0 folder1

下面对上述参数进行解释：
854 480 7     ----854 480代表动画的分辨率，854代表动画的宽度，480代表动画的高度；7则代表帧率，也就是一秒钟播放多少幅动画图片；
p 1 2 folder1 ----这里的p为标志符，1代表循环次数，2代表阶段间隔时间，folder1代表对应的动画文件夹名；

循环次数：0 : 表示无限循环。
注意

1.desc.txt 文件要在 Linux 环境下生成，因为有些空格不一样，在window下生成会有问题
2.part 目录中的图片的命名要是连续的，比如pic_001, pic_002, _pic_003 …
3.打包成bootanimation.zip文件的时候，要要用zip格式的存储方式打包。不是压缩文件打包这个要注意

1.首先通过 adb push 命令将文件上传到 sdcard 的根目录下。
2.然后通过 adb shell 进入 设备目录下，提取 root 权限， 把 bootanimation.zip 覆盖到 system/media 目录下。
3.关闭模拟器，重新的开机，观看动画
```

结果显示为
![结果显示](/uploads/替换结果显示.png)



