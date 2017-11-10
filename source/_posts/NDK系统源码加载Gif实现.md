---
layout: pager
title: NDK系统源码加载Gif实现
date: 2017-11-10 22:45:03
tags: [Android,NDK系统源码加载Gif实现]
description: NDK系统源码加载Gif实现
---

NDK系统源码加载Gif实现
<!--more-->

**NDK系统源码加载Gif实现**
===

```
加载GIF图片目前的方式有
Java方式
1  Movie类
创建Movie实例，绘制每一帧图片来达到Gif动态效果。
部分Gif图片不能自适应大小，
播放速度比实际播放速度快，
如果要显示的gif过大，
还会出现OOM的问题。
2.通过GifView      
GifView   --》GifHelper   gif文件 --》编码---》解析 --》播放
OOM的问题
3.通过Glide，Glide也是支持Gif的，但是里面的源码也是通过GifHelper，通过在java层来解析gif图的，也会有OOM，还有内心消耗比较大

2.可以通过NDK方式
```

java方法加载gif内存消耗,这个gif比较大，有14M，而且还有一个问题是通过java方法加载的速度跟原来的gif会不一样，会比较块
![结果显示](/uploads/java方法加载gif内存消耗.jpg)

**NDK的方式来实现**
===

```
首先android里面是肯定支持gif图的，要不然系统要怎么显示gif
所以就可以去查找系统的源码实现，系统gif源码的路径在系统的源码\external\giflib里面，把全部的文件拷贝出来，新建一个工程
Gif的基础知识
扩展块 
图形控制扩展(Graphic Control Extension)        固定值0xF9
作用：用来跟踪下一帧的信息和渲染形式        
注释扩展块  固定值0xFE
作用 ：可以用来记录图形、版权、描述等任何的非图形和控制的纯文本数据
图形文本扩展块  固定值0x01
作用：控制绘制的参数，比如左边界偏移量
应用程序扩展   固定值 0xFF
作用：这是提供给应用程序自己使用的，应用程序可以在这里定义自己的标识、
信息。可以做到当前app所生成的gif只能由我这个app打开
可以用来做 gif   加密  
 a   r  g   b   一个像素   4  个字节
 a  24位   r   《16 位     g <位   b 
```

系统gif支持的系统路径
![结果显示](/uploads/系统gif支持的系统路径.png)

GIf知识详解 [Gif基础知识]: http://blog.csdn.net/wzy198852/article/details/17266507
**代码的实现**
===

```java
java代码的书写
public class MainActivity extends AppCompatActivity
{

    Bitmap bitmap;
    GifHandler gifHandler;
    ImageView imageView;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        imageView = (ImageView) findViewById(R.id.image);
    }

    public void ndkLoadGif(View view)
    {
        File file = new File(Environment.getExternalStorageDirectory(), "xsgs163.gif");
        gifHandler = new GifHandler(file.getAbsolutePath());
        Log.i("tuch", "ndkLoadGif: " + file.getAbsolutePath());
        //得到gif   width  height  生成Bitmap
        int width = gifHandler.getWidth();
        int height = gifHandler.getHeight();
        bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        int delay = gifHandler.updateFrame(bitmap);
        handler.sendEmptyMessageDelayed(1, delay);
    }
	
	//通过handler来模拟加载下一帧的图片
    Handler handler = new Handler()
    {
        @Override
        public void handleMessage(Message msg)
        {
			//这里获取的是下一帧图片要显示的延迟时间，我们通过准确的获取到延迟的时间，所以就不会出现跟java一样，不同步的问题
            int delay = gifHandler.updateFrame(bitmap);
            handler.sendEmptyMessageDelayed(1, delay);
            imageView.setImageBitmap(bitmap);
        }
    };

}

//native 方法的书写
public class GifHandler
{
    static
    {
        System.loadLibrary("native-lib");
    }

    private long gifPoint;

    public GifHandler(String path)
    {
        this.gifPoint = load(path);
    }

    public long load(String path)
    {
        gifPoint = loadGif(path);
        return gifPoint;
    }

    //    long  C 指针
    private native long loadGif(String Path);

    private native int getWidth(long gifPoint);

    private native int getHeight(long gifPoint);

    //绘制函数
    private native int updateFrame(Bitmap bitmap, long gifPoint);

    public int getWidth()
    {
        return getWidth(gifPoint);
    }

    public int getHeight()
    {
        return getHeight(gifPoint);

    }

    public int updateFrame(Bitmap bitmap)
    {
        return updateFrame(bitmap, gifPoint);
    }
}

NDK的代码实现
#include <jni.h>
#include <android/bitmap.h>
#include "gif_lib.h"
#include "../../../../../../sdk/ndk-bundle/platforms/android-15/arch-arm/usr/include/malloc.h"
#include "../../../../../../sdk/ndk-bundle/platforms/android-15/arch-arm/usr/include/string.h"
#include <android/log.h>
#define  LOG_TAG    "yuhui"
#define  argb(a, r, g, b) ( ((a) & 0xff) << 24 ) | ( ((b) & 0xff) << 16 ) | ( ((g) & 0xff) << 8 ) | ((r) & 0xff)
#define  LOGE(...)  __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)

typedef struct GifBean {
    //播放当前帧数
    int currentFrame;
    //下一帧的播放时长  数组
    int *dealys;
    //存储gif总的帧数
    int totalFrame;
} GifBean;

//绘制图像
void drawFrame(GifFileType *gifFileType, GifBean *gifBean, AndroidBitmapInfo info, void *pixels) {

    //得到每一帧显示的信息集合
    SavedImage savedImage = gifFileType->SavedImages[gifBean->currentFrame];
    //得到图象标识符(Image Descriptor) ，图形的标识符里面存储了每一帧图片显示的实际宽高，跟显示的偏移值
    GifImageDesc frameInfo = savedImage.ImageDesc;

    //标识当前像素的索引值
    int pixelIndex = 0;
    GifByteType  gifByteType;
    GifColorType gifColorType;
    ColorMapObject * colorMapObject = frameInfo.ColorMap;
    int *px = (int *) pixels;
    px = (int *)((char *)px + info.stride * frameInfo.Top);
    int *line;
    for(int y = frameInfo.Top;y<frameInfo.Top + frameInfo.Height;++y)
    {
        line = px;
        for(int x = frameInfo.Left;x<frameInfo.Left + frameInfo.Width;++x)
        {
            pixelIndex = (y - frameInfo.Top) * frameInfo.Width + (x - frameInfo.Left);
            //RasterBits里面存储的是像素的压缩值，也就是本来是四个字节的，被压缩为1跟字节
            gifByteType=savedImage.RasterBits[pixelIndex];
            gifColorType=colorMapObject->Colors[gifByteType];
            //还原
            line[x]=argb(255,gifColorType.Red,gifColorType.Green,gifColorType.Blue);
        }
        //一行的像素
        px = (int *)((char*)px + info.stride);
    }
}

extern "C"
{

//加载gif图片，初始化操作
JNIEXPORT jlong JNICALL
Java_com_example_administrator_myinit_GifHandler_loadGif(JNIEnv *env, jobject instance,
                                                         jstring Path_) {
    const char *path = env->GetStringUTFChars(Path_, 0);
    int error = 0;
    GifFileType * gifFileType = DGifOpenFileName(path,&error);
    if(gifFileType == NULL)
    {
        LOGE("打开gif失败\n");
        return  0;
    }
    //初始化操作
    DGifSlurp(gifFileType);

    GifBean * gifBean = (GifBean *)malloc(sizeof(GifBean));
    if(gifBean == NULL)
    {
        LOGE("memory fail\n");
        return 0;
    }
    memset(gifBean,0,sizeof(GifBean));
    //设置Tag
    gifFileType->UserData = gifBean;
    //分配一个数组的内存空间，空间的大小为gif图总的帧数
    gifBean->dealys = (int *)malloc(gifFileType->ImageCount * sizeof(int));
    LOGE("total frame == %d\n",gifFileType->ImageCount);
    //出错的地方
    memset(gifBean->dealys,0,sizeof(int) * gifFileType->ImageCount);

    //播放时长赋值
    ExtensionBlock *extensionBlock = NULL;
    for(int i = 0;i<gifFileType->ImageCount;i++)
    {
        //SaveImage 为Gif 每一帧的存储信息
        SavedImage savedImage = gifFileType->SavedImages[i];
        //savedImage.ExtensionBlockCount 为扩展快的大小
        for(int j = 0;j< savedImage.ExtensionBlockCount;j++)
        {
            //获取到图形控制扩展(Graphic Control Extension)，图形控制块中的第5和第6个字节存储了延迟时间，也就是下一帧要显示的时间
            if(GRAPHICS_EXT_FUNC_CODE == savedImage.ExtensionBlocks[j].Function)
            {
                extensionBlock = &savedImage.ExtensionBlocks[j];
                break;
            }
        }
        //获取到了图形控制块
        if(extensionBlock != NULL)
        {
            //图形控制块中的，第5个字节跟第6个字节存储了延迟的时间
            int fra_delay = 10 * (extensionBlock->Bytes[2] << 8 | extensionBlock->Bytes[1]);
            //给数组赋值每一帧显示的时长
            LOGE("时间  %d   ",fra_delay);
            gifBean->dealys[i] = fra_delay;
        }
    }
    //设置总的帧数
    gifBean->totalFrame = gifFileType->ImageCount;

    env->ReleaseStringUTFChars(Path_, path);
    //返回地址，对于
    return (long)gifFileType;
}

JNIEXPORT jint JNICALL
Java_com_example_administrator_myinit_GifHandler_getWidth__J(JNIEnv *env, jobject instance,
                                                             jlong gifPoint) {

    GifFileType  * gifByteType = (GifFileType *) gifPoint;
    return gifByteType->SWidth;
}

JNIEXPORT jint JNICALL
Java_com_example_administrator_myinit_GifHandler_getHeight__J(JNIEnv *env, jobject instance,
                                                              jlong gifPoint) {

    GifFileType * gifFileType = (GifFileType *) gifPoint;
    return gifFileType->SHeight;
}

JNIEXPORT jint JNICALL
Java_com_example_administrator_myinit_GifHandler_updateFrame__Landroid_graphics_Bitmap_2J(
        JNIEnv *env, jobject instance, jobject bitmap, jlong gifPoint) {


    GifFileType * gifFileType = (GifFileType *) gifPoint;
    //因为我们在初始话的时候，给UserData赋值了这个值，所以这边就可以取出来，就相当于是Tag的存在
    GifBean * gifBean = (GifBean *) gifFileType->UserData;
    //获取bigmap图片的信息
    AndroidBitmapInfo bitmapInfo;
    AndroidBitmap_getInfo(env,bitmap,&bitmapInfo);

    void *pixels;
    //锁住bitmap
    AndroidBitmap_lockPixels(env,bitmap,&pixels);

    //绘制bitmap，给bitmap填充内容,填充像素
    drawFrame(gifFileType,gifBean,bitmapInfo,pixels);

    //标识当前的帧加一
    gifBean->currentFrame+=1;
    //如果当了最后的一帧，就让当前的帧赋值为0，重新的循环
    if(gifBean->currentFrame >= gifBean->totalFrame)
    {
        gifBean->currentFrame=0;
        LOGE("播放到了最后一针\n");
    }
    //释放锁
    AndroidBitmap_unlockPixels(env,bitmap);
    //返回延迟的时间
    return gifBean->dealys[gifBean->currentFrame];
}

}
```

**NDk结果显示**
===
NDk的方式来加载GIF，系统内存的消耗,而且不会导致加载速度不一样的问题，
![结果显示](/uploads/NDK方式加载GIF内存消耗.png)
我们可以通过进入adb shell命令得到应用程序的进程号，然后通过dumsys meminfo 加上对应的进程号，可以得到内存的情况
![结果显示](/uploads/NDK方式加载GifNative内存的情况.png)


