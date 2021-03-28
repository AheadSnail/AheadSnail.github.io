---
layout: pager
title: grafika-理解
date: 2021-01-07 11:21:30
tags: [Android,grafika,音视频]
description:  阅读 grafika 源码 ，记录下个人对grafika 的理解 
---

### 简介

> 跟前面俩篇文章一样，grafika 也是音视频项目里面 一个非常有学习，有参考意义的项目,此项目是Google 提供的一个非官方的项目，它的侧重点在于将 OpenGL 与 Android音视频 ApI综合使用，它包含了很多完整的demo，比如如何使用TextureView显示OpenGL内容，使用三种方式进行OpenGL内容的录制，如何进行硬编码操作，纹理的变换，滤镜等,由于这个项目每个demo都很有价值，源码又多这里大致简单的记录下个人觉得不错或者为什么这样写的原因


### 源码分析

- MainActivity  

```java
这个是应用程序的主界面，使用了适配器的方式列出来了当前包含的demo，这里面会首先准备生成俩个视频文件
GenerateTask genTask = new GenerateTask(caller, dialog, tags);//创建一个任务,并执行任务 ，因为 AsyncTask的创建必须在主线程中，所以外部调用这个方法的时候必须在主线程中
genTask.execute();

接着在AsyncTask 的 doInBackground 子线程调用的方法中，执行  contentManager.prepare(this, mTags[i]);其中 Tag的取值为 MOVIE_EIGHT_RECTS，MOVIE_SLIDERS，
这里会根据Tag的类型生成对应的视频文件

/**
* Returns the filename for the tag. 根据 Tag的类型返回对应的文件名
*/
private String getFileName(int tag) {
    switch (tag) {
        case MOVIE_EIGHT_RECTS:
            return "gen-eight-rects.mp4";
        case MOVIE_SLIDERS:
            return "gen-sliders.mp4";
        default:
            throw new RuntimeException("Unknown tag " + tag);
    }
}

private void prepare(ProgressUpdater prog, int tag) {
    GeneratedMovie movie;
    switch (tag) {
        case MOVIE_EIGHT_RECTS:// gen-eight-rects.mp4 视频文件对应的 标识
             movie = new MovieEightRects();
             movie.create(getPath(tag), prog);
             synchronized (mContent) {
                 mContent.add(tag, movie);
             }
         break;
         case MOVIE_SLIDERS: //gen-sliders.mp4 视频文件 对应的 标识
              movie = new MovieSliders();
              movie.create(getPath(tag), prog);
                synchronized (mContent) {
                    mContent.add(tag, movie);
              }
         break;
         default:
            //其他不支持
             throw new RuntimeException("Unknown tag " + tag);
	}
}

内部会创建 MediaCodec，其中Color类型设置为 MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface
而且I帧生成的时间设置为 private static final int IFRAME_INTERVAL = 5;
format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, IFRAME_INTERVAL);因为是本地视频所以没有必要设置太短，要不然视频会很大
接着通过  surface = mEncoder.createInputSurface();获得对应的Surface后续只要将内容绘制到这个Surface上面，就可以通知MediaCodec获得编码后的内容了
对于视频的生成使用了MediaMuxer，源码对于MediaCodec,MediaMuxer都是非常规范的，我们要写代码的话，完全可以参考来写，
比如当编码到最后一帧的时候要发送结束的标识要通过  mEncoder.signalEndOfInputStream(); 这也是因为 我们创建的COLOR_FormatSurface的类型所决定的,
对于 MediaMuxer的初始化我们可以在 MediaCodec.INFO_OUTPUT_FORMAT_CHANGED 中
 
//应该在接收缓冲区之前发生，并且应该只发生一次
if (mMuxerStarted) {
     throw new RuntimeException("format changed twice");
}
//获取到编码器的输出格式
MediaFormat newFormat = mEncoder.getOutputFormat();
Log.d(TAG, "encoder output format changed: " + newFormat);
// now that we have the Magic Goodies, start the muxer  接着我们用来初始化 Muxer，通过addTracker的方法添加Tracker
mTrackIndex = mMuxer.addTrack(newFormat);
//启动 muxer
mMuxer.start();

由于使用了 MediaMuxer所以对于编码后的数据，如果为配置信息的话，要忽略掉
//BUFFER_FLAG_CODEC_CONFIG 标识为 配置信息
if ((mBufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
    // The codec config data was pulled out and fed to the muxer when we got
    // the INFO_OUTPUT_FORMAT_CHANGED status.  Ignore it
    //如果是 INFO_OUTPUT_FORMAT_CHANGED muxer 忽略他
    if (VERBOSE) Log.d(TAG, "ignoring BUFFER_FLAG_CODEC_CONFIG");
    mBufferInfo.size = 0;
}
```

- PlayMovieActivity

```java
这个demo是一个播放视频的demo,用于显示视频画面使用的是 TextureView，然后设置对应的监听 mTextureView.setSurfaceTextureListener(this);
我们只有在回调了onSurfaceTextureAvailable的时候才能获取到对应的Surface，onSurfaceTextureDestroyed 为销毁的函数，这个函数的返回值有必要注意一下

/**
* TextureView 销毁回调,TextureView的生命周期为
*
* 切换至后台的时候会调用onSurfaceTextureDestroyed,从后台切换回来会调用onSurfaceTextureAvailable
* TextureView的ViewGroup remove TextureView的时候会调用onSurfaceTextureDestroyed,相同，TextureView的ViewGroup add TextureView的时候会调用 
* onSurfaceTextureAvailable，这些都是建立在视图可见的基础上，如果视图不可见，add也不会调用 onSurfaceTextureAvailable方法，remove也不会调用onSurfaceTextureDestroyed
*
* 当TextureView设置为Gone的时候，并不会调用onSurfaceTextureDestroyed方法
* @param st
* @return
*/
@Override
public boolean onSurfaceTextureDestroyed(SurfaceTexture st) {
    //标识不可用
    mSurfaceTextureReady = false;
    // assume activity is pausing, so don't need to update controls
    //大致的意思是如果返回ture，SurfaceTexture会自动销毁，如果返回false，SurfaceTexture不会销毁，需要用户手动调用release()进行销毁。
    return true;    // caller should release ST
}
真正播放的操作是在 MoviePlayer
player = new MoviePlayer(new File(getFilesDir(), mMovieFiles[mSelectedMovie]), surface, callback);
播放视频在另一个线程中进行 创建一个播放的任务，用来执行后台的播放操作
mPlayTask = new MoviePlayer.PlayTask(player, this);
还有一个重要的类 用来控制播放速度的对象
SpeedControlCallback callback = new SpeedControlCallback();
//执行播放任务
mPlayTask.execute();

播放任务由于是在子线程中执行，所以有时候我们要等待子线程启动完毕，外部调用才能继续往下执行，使用了while循环解决虚假的唤醒
public void waitForStop() {
     synchronized (mStopLock) {
        //如果当前线程还未停止，一直while循环，将 wait操作放在 while循环中，防止假的唤醒
        while (!mStopped) {
             try {
                    mStopLock.wait();
              } catch (InterruptedException ie) {
                    // discard
            }
        }
    }
}

播放的话使用了MediaExtractor来获取媒体的信息,比如获取到视频对应的TrackerIndex,设置对应的ColorFormat
private static int selectTrack(MediaExtractor extractor) {
    // Select the first video track we find, ignore the rest.
    int numTracks = extractor.getTrackCount();
    for (int i = 0; i < numTracks; i++) {
         MediaFormat format = extractor.getTrackFormat(i);
         String mime = format.getString(MediaFormat.KEY_MIME);
         if (mime.startsWith("video/")) {
             if (VERBOSE) {
                 Log.d(TAG, "Extractor selected track " + i + " (" + mime + "): " + format);
             }
             return i;
       	 }
   }
   return -1;
}
extractor.selectTrack(trackIndex);
//获取指定（index）的通道格式
MediaFormat format = extractor.getTrackFormat(trackIndex);
后续就可以使用这个格式创建对应的MediaCodec
String mime = format.getString(MediaFormat.KEY_MIME);
//创建视频的解码器对象
decoder = MediaCodec.createDecoderByType(mime);
//解码器配置了 输出的 Surface，这样解码之后的内容直接可以输出到 这个 Surface上
decoder.configure(format, mOutputSurface, null, 0);
//解码器开始解码
decoder.start();

这里由于使用了MediaCodec来做解码，对于视频的最后一帧我们需要手动的告知，像之前是调用signalEndOfInputStream来通知的
int chunkSize = extractor.readSampleData(inputBuf, 0);
if (chunkSize < 0) {
    // End of stream -- send empty frame with EOS flag set. 如果读到了文件的结尾，我们要发送一个空的数据包到 解码器中，同时将 flag设置为 BUFFER_FLAG_END_OF_STREAM
    //标识为当前解码器已经到了结尾了后续不能再接受内容了，同时将 inputDone 设置为 true
    decoder.queueInputBuffer(inputBufIndex, 0, 0, 0L, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
    //同时将 inputDone 设置为 true，标识后续不能再添加内容了
    inputDone = true;
    if (VERBOSE) Log.d(TAG, "sent input EOS");
}
同时解码的时候为 BUFFER_FLAG_END_OF_STREAM 就为输出的结束标识
//如果是到了文件的结尾，将 outputDone 设置为 true
if ((mBufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
   if (VERBOSE) Log.d(TAG, "output EOS");
   if (mLoop) {
       doLoop = true;
   } else {
      //用来控制是否已经到了结尾了，控制整个while循环
      outputDone = true;
    }
}
//释放操作，这样内容就会到了 解码器配置的 Surface上，这样内容就播放出来了
decoder.releaseOutputBuffer(decoderStatus, doRender);
                    
其中内部设计到了重新播放,这设计到了MediaCodec的生命周期，具体可以看注释
//如果要执行循环,doLoop 如果为 true，则到了解码的末尾，这里通过 extractor 重新seek到 开始的位置,重新的执行while循环
if (doLoop) {
   //Log.d(TAG, "Reached EOS, looping");
   extractor.seekTo(0, MediaExtractor.SEEK_TO_CLOSEST_SYNC);
   //标识输入没有完成
   inputDone = false;
   //执行状态（Executing）包含三个子状态： 刷新（Flushed）、运行（ Running） 以及流结束（End-of-Stream）。
   // 在调用start()方法后编解码器立即进入刷新子状态（Flushed），此时编解码器会拥有所有的缓存。一旦第一个输入缓存（input buffer）被移出队列，
   // 编解码器就转入运行子状态（Running），编解码器的大部分生命周期会在此状态下度过。当你将一个带有end-of-stream 标记的输入缓存入队列时，
   // 编解码器将转入流结束子状态（End-of-Stream）。在这种状态下，编解码器不再接收新的输入缓存，但它仍然产生输出缓存（output buffers）
   // 直到end-of- stream标记到达输出端
   //。所以 你可以在执行状态（Executing）下的任何时候通过调用flush()方法使编解码器重新返回到刷新子状态（Flushed）。
   decoder.flush();    // reset decoder state  重置 解码器的状态
   //回调 通知重置重新播放
   frameCallback.loopReset();
}

在调用 decoder.releaseOutputBuffer(decoderStatus, doRender); 函数之前还有这样的代码
if (doRender && frameCallback != null) {
    //播放前的回调
    frameCallback.preRender(mBufferInfo.presentationTimeUs);
}
其实这是用来控制播放速度的，因为没解码一帧视频都是有时间戳的，如果不按照对应的时间戳来播放的话，就会造成快进的效果,其实内部就是调用Sleep函数
//解码线程休眠
Thread.sleep(sleepTimeUsec / 1000, (int) (sleepTimeUsec % 1000) * 1000);
这样解码线程就不会过快，就能达到正常的显示速度
```

- ContinuousCaptureActivity


```java
ContinuousCaptureActivity 是一个录制视频的demo，内部使用 SurfaceView 来渲染相机的画面，并且设置回调监听
SurfaceView sv = (SurfaceView) findViewById(R.id.continuousCapture_surfaceView);
SurfaceHolder sh = sv.getHolder();
sh.addCallback(this);
在 surfaceCreated 回调函数中，获取到当前可用的Surface创建 
//创建渲染线程的 EGL 上下文,所以这里是主线程的 EGL 上下文,所以后续在使用这个 mEglCore 的时候，要确保在主线程
mEglCore = new EglCore(null, EglCore.FLAG_RECORDABLE);
//将当前的 Surface 对象 使用 EGLSurface 包装，再跟 EGL 上下文线程关联起来
mDisplaySurface = new WindowSurface(mEglCore, holder.getSurface(), false);
mDisplaySurface.makeCurrent();

// 首先创建一个 Texture2dProgram 对象，内部会根据指定的加载的类型，加载对应的纹理，顶点着色器，片段着色器，链接成一个着色器程序对象,之后使用 FullFrameRect进行包装 
mFullFrameBlit = new FullFrameRect(new Texture2dProgram(Texture2dProgram.ProgramType.TEXTURE_EXT));
//创建对应的纹理 GL_TEXTURE_EXTERNAL_OES
mTextureId = mFullFrameBlit.createTextureObject();
//之后将创建的纹理，使用SurfaceTexture 进行封装
mCameraTexture = new SurfaceTexture(mTextureId);
//并且设置监听事件
mCameraTexture.setOnFrameAvailableListener(this);
//开始相机的预览
startPreview();

所以在当前线程的EGL环境下,也就是前面一步的 mDisplaySurface.makeCurrent(); 是非常有必要的，创建了一个GL_TEXTURE_EXTERNAL_OES 纹理对象，
并且设置到摄像机中，这个纹理对象可以直接相机的YUV数据转成RGB，同时设置了监听回调，接着创建了一个 CircularEncoder编码器
try {
    //创建一个CircularEncoder，内部会开启一个编码线程
    mCircEncoder = new CircularEncoder(VIDEO_WIDTH, VIDEO_HEIGHT, 6000000,
    mCameraPreviewThousandFps / 1000, 7, mHandler);
} catch (IOException ioe) {
    throw new RuntimeException(ioe);
}
//拿到编码器的 Surface创建一个 WindowSurface 将当前的EGL 跟 Surface 关联起来
mEncoderSurface = new WindowSurface(mEglCore, mCircEncoder.getInputSurface(), true);

先来看
mCircEncoder = new CircularEncoder(VIDEO_WIDTH, VIDEO_HEIGHT, 6000000,mCameraPreviewThousandFps / 1000, 7, mHandler);
内部会创建一个编码的MedaiaCodec，并且获取到编码器中的InputSurface，而且内部还会创建一个编码线程
mEncoderThread = new EncoderThread(mEncoder, encBuffer, cb);
mEncoderThread.start();        
//确保编码线程启动完成
mEncoderThread.waitUntilReady();

到这里可以看出，我们主线程创建的EGL上下文是要跟编码的子线程共享的，因为通过共享EGL上下文，才能在线程之间拷贝纹理等操作,而上面确实是这样做,我们再看看预览编码阶段
private void drawFrame() {
   // Latch the next frame from the camera.  首先将 要显示的 SurfaceView 绑定到 当前线程的 EGL 上下文,这里直接在主线程执行绘制操作
   mDisplaySurface.makeCurrent();
   //调用 updateTexImage 这样相机的内容就能到 对应的纹理上面
   mCameraTexture.updateTexImage();  
   //当从OpenGL ES的纹理对象取样时，首先应该调用getTransformMatrix()来转换纹理坐标。
   //每次updateTexImage()被调用时，纹理矩阵都可能发生变化。所以，每次texture image被更新时，getTransformMatrix ()也应该被调用。
   mCameraTexture.getTransformMatrix(mTmpMatrix);
    
   // Fill the SurfaceView with it.  之后将相机的内容绘制到 预览的SurfaceView上面
   SurfaceView sv = (SurfaceView) findViewById(R.id.continuousCapture_surfaceView);
   //获取到预览显示控件的宽高
   int viewWidth = sv.getWidth();
   int viewHeight = sv.getHeight();
   //设置窗口的大小
   GLES20.glViewport(0, 0, viewWidth, viewHeight);
   //之后执行绘制操作
   mFullFrameBlit.drawFrame(mTextureId, mTmpMatrix);
   //绘制录制的进度
   drawExtra(mFrameNum, viewWidth, viewHeight);
    //交换缓冲，这样内容就能绘制到预览的界面上
    mDisplaySurface.swapBuffers();

   // Send it to the video encoder.  之后判断是否需要执行编码操作,mFileSaveInProgress 默认为 false,也就是 绘制操作都是在主线程
   //编码操作在子线程，这样能确保预览的内容一定能到编码的子线程中，当然内部也涉及到了共享EGL，所以纹理可以传递
   if (!mFileSaveInProgress) {
       //设置当前的EGL绑定的为编码线程的 WindowSurface
       mEncoderSurface.makeCurrent();
       //设置窗口的大小，这里为 视频的宽高大小
       GLES20.glViewport(0, 0, VIDEO_WIDTH, VIDEO_HEIGHT);
       //再次执行绘制操作
       mFullFrameBlit.drawFrame(mTextureId, mTmpMatrix);
       //绘制下面的录制进度条
       drawExtra(mFrameNum, VIDEO_WIDTH, VIDEO_HEIGHT);
       //通知编码子线程可以执行编码操作了
       mCircEncoder.frameAvailableSoon();
       //设置时间戳
       mEncoderSurface.setPresentationTime(mCameraTexture.getTimestamp());
       //交换缓冲，这样内容就会到了编码线程里面
       mEncoderSurface.swapBuffers();
    }
    mFrameNum++;
}
首先绘制到编码对应的Surface上，之后通知编码子线程执行编码操作，在获取到编码的内容有这样的操作
//将编码后的内容添加到 CircularEncoderBuffer 中
mEncBuffer.add(encodedData, mBufferInfo.flags, mBufferInfo.presentationTimeUs);
其实这就是循环队列缓冲，这个缓冲只能缓存7秒大小的数据，下面是这个类的成员定义
//使用 ByteBuffer来再次封装 mDataBuffer
private ByteBuffer mDataBufferWrapper;
//用来存储7秒输出的字节缓冲, desiredSpanSec默认为 7
private byte[] mDataBuffer;
//保存当前压缩数据包对应的flags
private int[] mPacketFlags;
//保存pts
private long[] mPacketPtsUsec;
//保存起始数据开始位置
private int[] mPacketStart;
//保存要存储的大小
private int[] mPacketLength;
// Data is added at head and removed from tail.  Head points to an empty node, so if
// head==tail the list is empty. 内部是一个循环队列，这是队列的头部, 相对于循环队列来说，就相当于队列的尾部
private int mMetaHead;
//循环队列，队列的尾部,尾指针用来标识 数据有效开始的位置
private int mMetaTail;

而在构造函数中，会完成对应的赋值操作，desiredSpanSec 是外面传递进来的，demo里面默认是 7
//算出每秒输出的 大小， bitRate 代表每秒输出的位数 / 8 就能得到字节数,这里就得到了 7秒视频输出的字节数
int dataBufferSize = bitRate * desiredSpanSec / 8;
mDataBuffer = new byte[dataBufferSize];
mDataBufferWrapper = ByteBuffer.wrap(mDataBuffer);
int metaBufferCount = frameRate * desiredSpanSec * 2;
mPacketFlags = new int[metaBufferCount];
mPacketPtsUsec = new long[metaBufferCount];
mPacketStart = new int[metaBufferCount];
mPacketLength = new int[metaBufferCount];

这里看下是怎么样添加
public void add(ByteBuffer buf, int flags, long ptsUsec) {
  //当前要保存的 大小
  int size = buf.limit() - buf.position();
   //如果不能添加，就一直移除尾部的元素
   while (!canAdd(size)) {
      removeTail();
   }
   //存储视频的总缓冲长度
   final int dataLen = mDataBuffer.length;
   //存储元数据的缓冲
   final int metaLen = mPacketStart.length;

   //获得当前数据包，要开始的索引位置
   int packetStart = getHeadStart();
   //保存当前压缩数据包对应的flags
   mPacketFlags[mMetaHead] = flags;
   //保存pts
   mPacketPtsUsec[mMetaHead] = ptsUsec;
   //保存起始数据开始位置
   mPacketStart[mMetaHead] = packetStart;
   //保存要存储的大小
   mPacketLength[mMetaHead] = size;

   // Copy the data in.  Take care if it gets split in half. 将数据拷贝进去，这里要注意，这里会将数据拷贝俩份
   if (packetStart + size < dataLen) {
        //如果当前剩余的缓冲够存储，那么就一次性拷贝进去
        // one chunk
        buf.get(mDataBuffer, packetStart, size);
    } else {
        // two chunks  如果当前的存储空间不够了
        int firstSize = dataLen - packetStart;
        if (VERBOSE) { Log.v(TAG, "split, firstsize=" + firstSize + " size=" + size); }
         //先存储当前能存储的大小
         buf.get(mDataBuffer, packetStart, firstSize);
         //接着将剩余的内容，直接填充到缓冲的头部，直接覆盖掉,因为前面一步已经通过 mMetaTail 指针设置了有效的开始位置
         buf.get(mDataBuffer, 0, size - firstSize);
    }
    //Head头指针加一操作
    mMetaHead = (mMetaHead + 1) % metaLen;
    //设置分割线
    if (EXTRA_DEBUG) {
         // The head packet is the next-available spot.  设置分割线
         mPacketFlags[mMetaHead] = 0x77aaccff;
         mPacketPtsUsec[mMetaHead] = -1000000000L;
         mPacketStart[mMetaHead] = -100000;
         mPacketLength[mMetaHead] = Integer.MAX_VALUE;
     }
}
/**
* Removes the tail packet.  移除尾部的指针，这里并没有真正的移除，只是将尾部指针向下移动一个位置
*/
private void removeTail() {
   if (mMetaHead == mMetaTail) {
        throw new RuntimeException("Can't removeTail() in empty buffer");
    }
    final int metaLen = mPacketStart.length;
    mMetaTail = (mMetaTail + 1) % metaLen;
}
可以看到其实就是保存我们编码的信息，存储到一个队列中，而这个队列只能存储7秒的视频内容，如果超过了，则移动mMetaTail 尾指针，这里只是移动这个指针，
而没有实际的操作里面的内存，因为后续这些内存可以直接复用，对于视频编码的时候，有这样的操作
//返回第一帧的有效数据，对于视频的第一帧这里要为关键帧
int index = mEncBuffer.getFirstIndex();
if (index < 0) {
   //如果找不到，
   Log.w(TAG, "Unable to get first index");
   mCallback.fileSaveComplete(1);
   return;
}
也就是从找到的关键帧开始，然后使用 MediaMuxer 来合成视频，而数据就从我们找到的这个关键帧索引开始从里面获取到数据
```

- DoubleDecodeActivity

```java
这是一个界面俩个窗口进行播放视频的demo，内部使用的是俩个TextureView 来进行画面的预览展示，这个demo主要想用来展示 比如横竖屏切换的时候保留SurfaceTexture
后续直接使用的目的，首先在成员函数中使用静态变量来保存对象

//一定要使用 静态变量存储，这样它们才能在重启后 Activity中的变量还有效
private static boolean sVideoRunning = false;
//静态变量 VideoBlob 数组
private static VideoBlob[] sBlob = new VideoBlob[VIDEO_COUNT];
在onCreate的函数中
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_double_decode);

    if (!sVideoRunning) {
        //播放  gen-sliders.mp4 视频文件
        sBlob[0] = new VideoBlob((TextureView) findViewById(R.id.double1_texture_view),ContentManager.MOVIE_SLIDERS, 0);
         //播放 gen-eight-rects.mp4 视频文件
         sBlob[1] = new VideoBlob((TextureView) findViewById(R.id.double2_texture_view),ContentManager.MOVIE_EIGHT_RECTS, 1);
         //标识视频解码器对象创建成功了
         sVideoRunning = true;
     } else {
         // 如果是重新创建的话，我们执行 resetView的操作, 比如横竖屏切换
         sBlob[0].recreateView((TextureView) findViewById(R.id.double1_texture_view));
         sBlob[1].recreateView((TextureView) findViewById(R.id.double2_texture_view));
     }
}
在VideoBlob 中，完成对应TextureView的监听回调
public void onSurfaceTextureAvailable(SurfaceTexture st, int width, int height) {
   Log.d(LTAG, "onSurfaceTextureAvailable size=" + width + "x" + height + ", st=" + st);
         
    //如果是第一次 那么 mSavedSurfaceTexture 为空，此时保存 mSavedSurfaceTexture 变量
    if (mSavedSurfaceTexture == null) {
        mSavedSurfaceTexture = st;
        //得到 mMovieTag 对应的视频文件路径
         File sliders = ContentManager.getInstance().getPath(mMovieTag);
        //之后创建一个 播放电影的线程执行播放, 注意这里使用了  Surface 来保证 SurfaceTexture ，后续再来传递给 解码器的MediaCodec的 解码器对应渲染的Surface上
         mPlayThread = new PlayMovieThread(sliders, new Surface(st), mCallback);
     } else { 
        //不写在这里的原因就是，有些数据，界面发生重新绘制的时候，不会调用这个回调方法 onSurfaceTextureAvailable,所以不写在这里
        //Log.d(LTAG, "using saved st=" + mSavedSurfaceTexture);
        //mTextureView.setSurfaceTexture(mSavedSurfaceTexture);
     }
}
在TextureView可用的时候，创建对应的播放线程 PlayMovieThread，内部会在子线程中完成解析播放的操作，由于传递了Surface进去，
所以使用MediaCodec可以直接将编码后的数据直接显示出来,而在 onSurfaceTextureDestroyed 的时候来决定是否要回收
    
public boolean onSurfaceTextureDestroyed(SurfaceTexture st) {
   Log.d(LTAG, "onSurfaceTextureDestroyed st=" + st);
   
    //这里要注意 SurfaceTexture 中 onSurfaceTextureDestroyed 声明周期回调返回值的意思
    //大致的意思是如果返回ture，SurfaceTexture会自动销毁，如果返回false，SurfaceTexture不会销毁，需要用户手动调用release()进行销毁
    //如果是界面发生重新绘制，比如横竖屏切换，也会回调这个方法 onSurfaceTextureDestroyed,如果 这个返回 true，则代表要销毁 SurfaceTexture
    return (mSavedSurfaceTexture == null);
}
而将这个变量重置为null的函数为
/**
* Stop playback and shut everything down.  停止播放线程，然后释放掉任何内容
*/
public void stopPlayback() {
   Log.d(LTAG, "stopPlayback");
   //停止播放线程
    mPlayThread.requestStop();   
   //将保存的 mSavedSurfaceTexture 重置为 null，这样后续的 onSurfaceTextureDestroyed 的时候，就会因为这个变量为空，而释放掉整个TextureView
    mSavedSurfaceTexture = null;
}
而这个方法的调用在 onPause的时候,也就是Activity界面不可见的时候，而且还要满足isFinishing才能
@Override
protected void onPause() {
   super.onPause();
   //判断是否要执行销毁了
   boolean finishing = isFinishing();
   Log.d(TAG, "isFinishing: " + finishing);
   for (int i = 0; i < VIDEO_COUNT; i++) {
       //如果是要执行销毁了，我们发送停止播放线程的通知
       if (finishing) {
           //发送停止播放线程的通知
           sBlob[i].stopPlayback();
           //同时将静态变量设置为 null
           sBlob[i] = null;
        }
    }
    //设置sVideoRunning 的状态
    sVideoRunning = !finishing;
    Log.d(TAG, "onPause complete sVideoRunning  == "+ sVideoRunning);
}
要满足了isFinishing()才会调用stopPlayback，比如我们点击了返回值，就能触发，而像其他的比如横竖屏切换的时候，虽然会执行onPause但是并不满isFinishing()
所以并不会执行stopPlayback这样，对应的TextureView就不会销毁了，所以下次重新创建的时候，会直接执行onCreate，然后调用recreateView()

public void recreateView(TextureView view) {
   Log.d(LTAG, "recreateView: " + view);
   mTextureView = view;
   //重新设置 TextureView的监听回调
   mTextureView.setSurfaceTextureListener(this);

   // mSavedSurfaceTexture 默认为 空 ,比如 横竖屏切换的时候，下次再进来就可以直接将之前保存的 mSavedSurfaceTexture 直接设置上去
   if (mSavedSurfaceTexture != null) {
       Log.d(LTAG, "using saved st=" + mSavedSurfaceTexture);
       view.setSurfaceTexture(mSavedSurfaceTexture);
    }
}
因为渲染子线程其实一直都在执行，所以下次重新创建的时候，直接调用setSurfaceTexture关联上就能显示出内容了
```

- HardwareScalerActivity

```java
这个demo是通过绘制一个旋转的三角形，一个四周移动的正放形，和一个边框 然后在不同的宽高大小的情况下渲染过程,画面的预览使用了SurfaceView设置监听，
然后在surfaceCreated的时候，初始化变量，创建渲染子线程RenderThread，同时设置了Choreographer.getInstance().postFrameCallback(this);
同时通知渲染子线程准备渲染子线程的环境，加载彩色的三角形，四边形的纹理

//加载四边形颜色的纹理
mCoarseTexture = GeneratedTexture.createTestTexture(GeneratedTexture.Image.COARSE);
//加载三角形颜色的纹理
mFineTexture = GeneratedTexture.createTestTexture(GeneratedTexture.Image.FINE);
//创建纹理
public static int createTestTexture(Image which) {
   ByteBuffer buf;
   switch (which) {
       case COARSE:
            //生成四边形颜色的缓冲内容
            buf = sCoarseImageData;
            break;
        case FINE:
            //生成三角形颜色的缓冲内容
            buf = sFineImageData;
            break;
        default:
            throw new RuntimeException("unknown image");
   }
   //之后在根据前面生成的图片颜色的内容，使用纹理的方式加载得到对应的内容
   return GlUtil.createImageTexture(buf, TEX_SIZE, TEX_SIZE, FORMAT);
}
//这里根据 64 * 64的生成 4 * 4的正方形颜色表格
private static final ByteBuffer sCoarseImageData = generateCoarseData();
//这里生成三角形显示的颜色 缓冲
private static final ByteBuffer sFineImageData = generateFineData();
//这里就看下彩色四边形内容的生成
private static ByteBuffer generateCoarseData() {
   //创建一个缓冲空间,这个缓冲的大小为 64 * 64 的，而且每个元素都为 RGBA四个字节
   byte[] buf = new byte[TEX_SIZE * TEX_SIZE * BYTES_PER_PIXEL];
   //TEX_SIZE / 4 = 16
   final int scale = TEX_SIZE / 4;        // convert 64x64 --> 4x4

   //填充buf,i的变量是每四个字节累加
   for (int i = 0; i < buf.length; i += BYTES_PER_PIXEL) {
        //行 ,这里的取值为 0 - 15
        int texRow = (i / BYTES_PER_PIXEL) / TEX_SIZE;
        //列，这里的取值为 0 - 15
        int texCol = (i / BYTES_PER_PIXEL) % TEX_SIZE;

        //因为上面的行跟列的取值都为 0- 15 所以这里的取值为 0 - 3
        int gridRow = texRow / scale;  // 0-3
        int gridCol = texCol / scale;  // 0-3
        //进一步算出 gridIndex 取值为 0 - 15
        int gridIndex = (gridRow * 4) + gridCol;  // 0-15 也就是将 64 * 64的转成 4 * 4

         //算出索引之后，从颜色值表中得到索引对应的颜色
         int color = GRID[gridIndex];

         // override the pixels in two corners to check coverage  ，检查覆盖范围
         if (i == 0) {
             color = OPAQUE | WHITE;
          } else if (i == buf.length - BYTES_PER_PIXEL) {
             color = OPAQUE | WHITE;
          }

          // extract RGBA; use "int" instead of "byte" to get unsigned values 分离出 RGBA
          int red = color & 0xff;
          int green = (color >> 8) & 0xff;
          int blue = (color >> 16) & 0xff;
          int alpha = (color >> 24) & 0xff;

          // pre-multiply colors and store in buffer 根据RGB 与 alpha相乘，最后将结果保存到 buf 中
          float alphaM = alpha / 255.0f;
          buf[i] = (byte) (red * alphaM);
          buf[i+1] = (byte) (green * alphaM);
          buf[i+2] = (byte) (blue * alphaM);
          buf[i+3] = (byte) alpha;
    }
    //最后将 64 * 64的内容，填充到 ByteBuffer中，返回
    ByteBuffer byteBuf = ByteBuffer.allocateDirect(buf.length);
    byteBuf.put(buf);
    byteBuf.position(0);
    return byteBuf;
}

//根据传递的图像内容缓冲加载得到纹理
public static int createImageTexture(ByteBuffer data, int width, int height, int format) {
    int[] textureHandles = new int[1];
    int textureHandle;

    //创建一个纹理对象
    GLES20.glGenTextures(1, textureHandles, 0);
    //绑定纹理
    textureHandle = textureHandles[0];
    GlUtil.checkGlError("glGenTextures");
    // Bind the texture handle to the 2D texture target.
    GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureHandle);
    //设置纹理的环绕方式
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
    GlUtil.checkGlError("loadImageTexture");
    GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, /*level*/ 0, format, width, height, /*border*/ 0, format, GLES20.GL_UNSIGNED_BYTE, data);
    GlUtil.checkGlError("loadImageTexture");
    return textureHandle;
}

在surfaceChanged的回调中，传递当前预览控件的宽高大小初始化旋转三角形，四边形，还有四个边框的初始值,比如设置三角形，四边形
// 设置三角形的颜色
mTri.setColor(0.1f, 0.9f, 0.1f);
//设置彩色三角形的纹理
mTri.setTexture(mFineTexture);
//设置x，y缩放的比率
mTri.setScale(smallDim / 3.0f, smallDim / 3.0f);
//设置起始点x,y的坐标,这里设置为屏幕的中心点的坐标
mTri.setPosition(width / 2.0f, height / 2.0f);
//设置四边形的颜色
mRect.setColor(0.9f, 0.1f, 0.1f);
//设置四边形彩色的纹理
mRect.setTexture(mCoarseTexture);
//设置缩放的大小
mRect.setScale(smallDim / 5.0f, smallDim / 5.0f);
//设置起始点x,y的坐标，这里设置为 屏幕的中心点的坐标
mRect.setPosition(width / 2.0f, height / 2.0f);

至于是怎么样做到旋转和移动，这就要参考OpenGL的做坐标系统了,这里面会有介绍
https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/

同时为了让物体能按照比例显示在我们的界面中，我们要生成一个正交投影矩阵，上面的链接也有介绍，在这个demo中是通过下面的方式创建的
// Use full window. 设置窗口的大小
GLES20.glViewport(0, 0, width, height);
// Simple orthographic projection, with (0,0) in lower-left corner.
//创建正交投影矩阵,第二个参数指定 偏移量，3,4个参数指定了宽度，4,5指定了高度，后面俩个参数指定了近平面和远平面的距离
//也就是在这范围内容就会保留，其他的会被忽略掉 近平面设置为 -1 ，原平面设置为 1
Matrix.orthoM(mDisplayProjectionMatrix, 0, 0, width, 0, height, -1, 1);

初始值设置完之后，后面就可以按照时间的变化来改变这些值了,因为前面是设置了Choreographer所以会执行对应的doFrame方法,根据doFrame传递过来的时间算出对应的值
private void update(long timeStampNanos) {
    // Compute time from previous frame.  计算当前帧和上一帧时间的差值
    long intervalNanos;
    if (mPrevTimeNanos == 0) {
        //刚开始为0
        intervalNanos = 0;
    } else {
        //计算当前帧的时间跟上一帧的时间差
        intervalNanos = timeStampNanos - mPrevTimeNanos;

        final long ONE_SECOND_NANOS = 1000000000L;
      	...
        //保存当前帧的时间戳
        mPrevTimeNanos = timeStampNanos;

        final float ONE_BILLION_F = 1000000000.0f;
        final float elapsedSeconds = intervalNanos / ONE_BILLION_F;
        
        // or 120 degrees per second. 旋转三角形。我们需要每3秒完整的360度旋转，或者每秒120度旋转。
        final int SECS_PER_SPIN = 3;
        //每三秒要旋转 360 ，所以这里根据时间戳算出对应的旋转角度
        float angleDelta = (360.0f / SECS_PER_SPIN) * elapsedSeconds;
        //设置三角形的旋转角度
        mTri.setRotation(mTri.getRotation() + angleDelta);

        //在屏幕上反弹。矩形是1x1的平方，放大到 nxn。//我们不做花哨的碰撞侦测，所以盒子可以稍微重叠边缘。我们把边缘画在最后，这样不会太显眼。
        float xpos = mRect.getPositionX();
        //起始点的位置
        float ypos = mRect.getPositionY();
        //返回 X 轴的 缩放数值
        float xscale = mRect.getScaleX();
        //返回 Y 轴的 缩放数值
        float yscale = mRect.getScaleY();

        // mRectVelX,mRectVelY 代表的是矩形在视图中x，y轴的移动速度，所以这里根据时间的百分比算出下一个点的坐标，x,y的坐标
        xpos += mRectVelX * elapsedSeconds;
        ypos += mRectVelY * elapsedSeconds;

        //mRectVelX > 0 && xpos + xscale/2 > mInnerRight+1 为到了右边的最大值，此时要执行反弹的操作，这里直接将   mRectVelX = -mRectVelX
        //那么这里就要考虑最左边的情况,这里最左边的情况就是   xpos - xscale/2 < mInnerLeft，那么此时把 mRectVelX 设置为正值，那么又实现了左边反弹的效果
        if ((mRectVelX < 0 && xpos - xscale/2 < mInnerLeft) || (mRectVelX > 0 && xpos + xscale/2 > mInnerRight+1)) {
            mRectVelX = -mRectVelX;
        }
        //这是确保不能超过上下边界，然后实现类似反弹的效果
        if ((mRectVelY < 0 && ypos - yscale/2 < mInnerBottom) || (mRectVelY > 0 && ypos + yscale/2 > mInnerTop+1)) {
            mRectVelY = -mRectVelY;
        }
        //设置起始点的值
        mRect.setPosition(xpos, ypos);
}
修改了这些值之后就可以执行绘制操作了
mTri.draw(mTexProgram, mDisplayProjectionMatrix);
mRect.draw(mTexProgram, mDisplayProjectionMatrix);

public void draw(Texture2dProgram program, float[] projectionMatrix) {
  // Compute model/view/projection matrix. 矩阵乘法Matrix.multiplyMM();
  //矩阵相乘，这里将 模型矩阵 * 投影矩阵 ，所以这个顺序不能改变 注意乘法要从右向左读 gl_Position = projection * view * model * vec4(aPos, 1.0);
  Matrix.multiplyMM(mScratchMatrix, 0, projectionMatrix, 0, getModelViewMatrix(), 0);

  //最终执行绘制操作
  program.draw(mScratchMatrix, mDrawable.getVertexArray(), 0,
                mDrawable.getVertexCount(), mDrawable.getCoordsPerVertex(),
                mDrawable.getVertexStride(), GlUtil.IDENTITY_MATRIX, mDrawable.getTexCoordArray(),
                mTextureId, mDrawable.getTexCoordStride());
}
首先看下变换矩阵 getModelViewMatrix()生成的方式
private void recomputeMatrix() {
   float[] modelView = mModelViewMatrix;
   //设置为单位矩阵,也即是每个索引对应的值都为1
   Matrix.setIdentityM(modelView, 0);
   //设置平移，这里设置的平移为 mPosX,mPosY   表示为  Matrix.translateM(m, offset, x, y, z)
   Matrix.translateM(modelView, 0, mPosX, mPosY, 0.0f);
   //如果含有角度
   if (mAngle != 0.0f) {
       //设置旋转角度  Matrix.rotateM(m, offset, a, x, y, z) 旋转
        Matrix.rotateM(modelView, 0, mAngle, 0.0f, 0.0f, 1.0f);
   }
   //最后再设置缩放配置,Matrix.scaleM(m,offset,x,y,z) m 表示原矩阵，offset表示矩阵数组是用m的哪个index开始，x,y,z 表示缩放因子，等价于 m * (x,y,z)
   Matrix.scaleM(modelView, 0, mScaleX, mScaleY, 1.0f);
   //同时表示
   mMatrixReady = true;
}
将生成的最终的变换坐标传递给 program,其实内部就是绑定设置各种OpenGL的属性和变量，我们这里看下对应的顶点着色器的声明
// Simple vertex shader, used for all programs.  简单的顶点着色器，所有的都适配
//gl_Position = uMVPMatrix * aPosition;\n"  是 将顶点坐标 变换为 平面坐标的过程，uMVPMatrix 矩阵为 模型 观察 视图 交互的结果
private static final String VERTEX_SHADER =
      "uniform mat4 uMVPMatrix;\n" +
      "uniform mat4 uTexMatrix;\n" +
      "attribute vec4 aPosition;\n" +
      "attribute vec4 aTextureCoord;\n" +
      "varying vec2 vTextureCoord;\n" +
      "void main() {\n" +
          "gl_Position = uMVPMatrix * aPosition;\n" +
          "vTextureCoord = (uTexMatrix * aTextureCoord).xy;\n" +
      "}\n";
可以看到这其实就是前面的介绍OpenGL的坐标系统的变换，不懂的可以详细看下，对于边框的设置和绘制这里就不看了，大体是一样的，
只要记住 OpenGL的坐标原点是左下角的就可以理解为什么要做这样的设置
```

- MultiSurfaceActivity

```java
MultiSurfaceActivity 是一个采用了三个SurfaceView来显示画面数据的，下面是对应的SurfaceView的排列方式,大致就是 setZOrderMediaOverlay和setZOrderOnTop调用了
    
mSurfaceView1 = (SurfaceView) findViewById(R.id.multiSurfaceView1);
mSurfaceView1.getHolder().addCallback(this);
//控制是否应将表面视图的内容视为安全内容，以防止其出现在屏幕截图中或在非安全显示器上查看,也就是设置禁止截屏的意思
mSurfaceView1.setSecure(true);
       
// plane should switch us to RGBA8888.  第二个SurfaceView是在第一个的上面， 必须设置为透明，并且颜色设置为  RGBA8888
mSurfaceView2 = (SurfaceView) findViewById(R.id.multiSurfaceView2);
mSurfaceView2.getHolder().addCallback(this);
//设置透明
mSurfaceView2.getHolder().setFormat(PixelFormat.TRANSLUCENT);
//控制表面视图的表面是否放置在窗口中另一个常规表面视图的顶部（但仍位于窗口本身后面）。
mSurfaceView2.setZOrderMediaOverlay(true);

// #3 is above everything, including the UI.  Also translucent.  第三个 SurfaceView 设置在 最上层，也是设置为透明的
mSurfaceView3 = (SurfaceView) findViewById(R.id.multiSurfaceView3);
mSurfaceView3.getHolder().addCallback(this);
mSurfaceView3.getHolder().setFormat(PixelFormat.TRANSLUCENT);
//控制是否将表面视图的表面放置在其窗口的顶部。也就是设置在窗口的顶部
mSurfaceView3.setZOrderOnTop(true);

之后在没一层绘制内容，绘制的方式是通过Canvas的方式来进行的     Canvas canvas = surface.lockCanvas(null);获得画布之后，
就可以利用我们Android的绘制方式简单方便的绘制出来了,当然绘制出来后，别忘了设置  surface.unlockCanvasAndPost(canvas); 将内容渲染出来
```

- PlayMovieSurfaceActivity

```java
其实也是一个播放视频demo 跟第一个demo很像，不过这个demo的预览数据是使用 SurfaceView，而第一个是使用TextureView的方式,下面简单的介绍下不同点，
像文中的介绍说的是SurfaceViw来进行绘制的话，效率会更高,因为内部本身一个Surface，而TextureView的话，是要自己创建一个Surface
    
还有就是在对对应的控件进行缩放，放大的方式不一样，如果是TextureView的话，可以通过下面的方式来进行变换
//获得原本的矩阵
Matrix txform = new Matrix();
mTextureView.getTransform(txform);
//设置缩放
txform.setScale((float) newWidth / viewWidth, (float) newHeight / viewHeight);
//txform.postRotate(10);          // just for fun
txform.postTranslate(xoff, yoff);
//重新更新 矩阵
mTextureView.setTransform(txform);

而如果是SurfaceView的话，要通过在外面嵌套一层来实现
//设置 播放视图的窗口比率变化
AspectFrameLayout layout = (AspectFrameLayout) findViewById(R.id.playMovie_afl);
int width = player.getVideoWidth();
int height = player.getVideoHeight();
layout.setAspectRatio((double) width / height);

//布局是这样的
<com.example.grafika.AspectFrameLayout
        android:id="@+id/playMovie_afl"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_below="@id/play_stop_button"
        android:layout_centerInParent="true" >

        <SurfaceView
            android:id="@+id/playMovie_surface"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="center" />
</com.example.grafika.AspectFrameLayout>
```

- RecordFBOActivity

```java
RecordFBOActivity 是一个展示使用了三种方式来进行录制视频的效率，一种是通过绘制俩次的方式，就是一次绘制到预览控件上面，一次绘制到MediaCodec的InputSurface上面，
第二种是使用 GLSL3 的方式，通过调用 glBlitFramebuffer来拷贝数据，第三种是通过 FBO的方式，下面依次介绍下,由于这个demo里面的内容跟前面绘制彩色三角形很相似，
其实就是在这个demo上面加了录制的功能，所以这些就不介绍了，直接看下是怎么样实现视频的编码的

先看下第一种的绘制方式，就是在预览线程中先将内容绘制到预览的控件上面，接着还是在预览线程中，将内容绘制到MediaCodec的InputSurface中，绘制完成之后，
通过编码子线程执行编码了，因为涉及到了OpenGL 共享EGL上下文，还有这样做可以保证预览的内容跟编码的内容一样
draw();
swapResult = mWindowSurface.swapBuffers();
// Draw for recording, swap.  通知编码线程
mVideoEncoder.frameAvailableSoon();
//当前的EGL 绑定为 编码线程的 EGLSurface
mInputWindowSurface.makeCurrent();            
GLES20.glClearColor(0f, 0f, 0f, 1f);
GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
//设置窗口的大小
GLES20.glViewport(mVideoRect.left, mVideoRect.top, mVideoRect.width(), mVideoRect.height());
//设置裁剪的范围
GLES20.glEnable(GLES20.GL_SCISSOR_TEST);
GLES20.glScissor(mVideoRect.left, mVideoRect.top, mVideoRect.width(), mVideoRect.height());
//执行绘制操作
draw();
//禁用裁剪的范围
GLES20.glDisable(GLES20.GL_SCISSOR_TEST);
//设置对应的时间
mInputWindowSurface.setPresentationTime(timeStampNanos);
//提交到 编码的子线程中
mInputWindowSurface.swapBuffers();
// Restore.  之后，重新恢复过来
GLES20.glViewport(0, 0, mWindowSurface.getWidth(), mWindowSurface.getHeight());
mWindowSurface.makeCurrent();

接着看下第二种方式，大致就是使用了OpenGL3.0版本的glBlitFramebuffer方式来拷贝数据，但是在执行这个方法之前需要改变编码器EGL的 read EGLSurface绑定为
预览的 mWindowSurface
draw();
//通知编码子线程可以编码了
mVideoEncoder.frameAvailableSoon();
//这里让我们编码EGL上下文 对应的readSurface来自 mWindowSurface,但是drawSurface还是渲染子线程关联的EGLSurface对象
mInputWindowSurface.makeCurrentReadFrom(mWindowSurface);
              
//清除像素，我们不准备覆盖所有的像素，再说一遍，我们不需要清除整个屏幕
//glClear函数来自OPENGL,其中它是通过glClear使用红，绿，蓝以及AFA值来清除颜色缓冲区的，并且都被归一化在(0，1)之间的值，其实就是清空当前的所有颜色。
GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
GlUtil.checkGlError("before glBlitFramebuffer");
Log.v(TAG, "glBlitFramebuffer: 0,0," + mWindowSurface.getWidth() + "," +mWindowSurface.getHeight() + "  " + mVideoRect.left +        
   "," +mVideoRect.top + "," + mVideoRect.right + "," + mVideoRect.bottom + "  COLOR_BUFFER GL_NEAREST");

//我们可以直接通过glBitFrameBuffer来拷贝对应的渲染缓冲区，拷贝之前需要通过glBinderFrameBuffer()操作将源帧缓冲区绑定为 GL_READ_FRAMEBUFFER，将
//目的缓冲区绑定为 GL_GRAW_FRAMEBUFFER或 GL_FRAME_BUFFER
GLES30.glBlitFramebuffer(0, 0, mWindowSurface.getWidth(), mWindowSurface.getHeight(),mVideoRect.left, mVideoRect.top,
   mVideoRect.right, mVideoRect.bottom,GLES30.GL_COLOR_BUFFER_BIT, GLES30.GL_NEAREST);

//拷贝完成，检查错误
int err;
if ((err = GLES30.glGetError()) != GLES30.GL_NO_ERROR) {
   Log.w(TAG, "ERROR: glBlitFramebuffer failed: 0x" + Integer.toHexString(err));
}
//设置渲染子线程的 显示的时间，交换缓冲，这样内容就到了对应的编码队列中
mInputWindowSurface.setPresentationTime(timeStampNanos);
mInputWindowSurface.swapBuffers();
// Now swap the display buffer. 现在我们可以交换回来了
mWindowSurface.makeCurrent();
swapResult = mWindowSurface.swapBuffers();

第三种就是通过FBO的方式了,关于帧缓冲的介绍可以下OpenGL的介绍,大致就是通过先绑定帧缓冲，先将内容绘制到帧缓冲挂载的纹理上面，然后操作这个挂载的纹理就好了，
比如绘制到预览画面，绘制到 MediaCodec的 InputSurface中即可,下面是关于帧缓冲的详细介绍
https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/

GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, mFramebuffer);
GlUtil.checkGlError("glBindFramebuffer");
//执行绘制操作,这样绘制的内容就到了 mOffscreenTexture 上面
draw();

// Blit to display.  接着恢复到屏幕
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
GlUtil.checkGlError("glBindFramebuffer");
//我们使用前面得到的纹理，直接绘制,然后显示到屏幕上
mFullScreen.drawFrame(mOffscreenTexture, mIdentityMatrix);
swapResult = mWindowSurface.swapBuffers();

// Blit to encoder.  用来编码
mVideoEncoder.frameAvailableSoon();
//将当前的EGL 上下文绑定为 渲染子线程的 Surface
mInputWindowSurface.makeCurrent();
//glClear函数来自OPENGL,其中它是通过glClear使用红，绿，蓝以及AFA值来清除颜色缓冲区的，并且都被归一化在(0，1)之间的值，其实就是清空当前的所有颜色。
GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);    // again, only really need to
GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);     //  clear pixels outside rect
//设置窗口的大小
GLES20.glViewport(mVideoRect.left, mVideoRect.top, mVideoRect.width(), mVideoRect.height());
//执行绘制操作
mFullScreen.drawFrame(mOffscreenTexture, mIdentityMatrix);
//设置编码子线程显示的时间，交换缓冲，这样内容就到了 编码子线程的 Surface上
mInputWindowSurface.setPresentationTime(timeStampNanos);
mInputWindowSurface.swapBuffers();
// Restore previous values. 执行恢复操作
GLES20.glViewport(0, 0, mWindowSurface.getWidth(), mWindowSurface.getHeight());
mWindowSurface.makeCurrent();
```

- ScreenRecordActivity

```java
ScreenRecordActivity 是一个集成自播放器的demo 实现的屏幕录制的demo，关于播放的部分这里就不细看，主要介绍下 屏幕录制的实现，
可以实现 MediaProjectionManager来实现，其实也有一套的流程的,

比如首先完成 MediaProjectionManager 的初始化
//得到系统的 MediaProjectionManager
mediaProjectionManager = (MediaProjectionManager) getSystemService(android.content.Context.MEDIA_PROJECTION_SERVICE);

点击录制的时候，要先创建对应的Intent
//然后调用 startActivityForResult传入 projectionManager.createScreenCaptureIntent()创建的Intent
Intent permissionIntent = mediaProjectionManager.createScreenCaptureIntent();
startActivityForResult(permissionIntent, REQUEST_CODE_CAPTURE_PERM);

在onActivityResult 的中创建 对应的MediaProjection,之后开始录制
//拥有权限，通过 resultCode和data来获取 MediaProjection
mediaProjection = mediaProjectionManager.getMediaProjection(resultCode, intent);
//开始录制屏幕
startRecording();

接着在建立关联，也就是将MediaCodec的 InputSurface关联到 mediaProjection上
// Start the video input.  开始设置视频的输出，第6个参数是一个Surface，当VirtualDisplay被创建出来后，也就是createVirtualDisplay调用后，
你在真实屏幕上的每一帧都会使人到Surface参数中，也就是说，如果你设置了 SurfaceView，然后传入SurfaceView的Surface那么你在屏幕上的操作都会显示在Surface中
mediaProjection.createVirtualDisplay("Recording Display", screenWidth,
    screenHeight, screenDensity, DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR/* flags */, inputSurface,
    null /* callback */, null /* handler */);

内部还有介绍了 MediaCodec的异步方式，但是API的级别要在 M 以上，也即是23以上，异步的话，可以这样设置
videoEncoder = MediaCodec.createEncoderByType(VIDEO_MIME_TYPE);
videoEncoder.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
inputSurface = videoEncoder.createInputSurface();
//设置异步的回调函数
videoEncoder.setCallback(encoderCallback);
//开始编码
videoEncoder.start();

//这里采用的是 MediaCodec 的异步方式
encoderCallback = new MediaCodec.Callback() {
  @Override
  public void onInputBufferAvailable(@NonNull MediaCodec codec, int index) {
      Log.d(TAG, "Input Buffer Avail");
  }

  /**
  * 编码出来了内容了
  * @param codec
  * @param index
  * @param info
  */
  @Override
  public void onOutputBufferAvailable(@NonNull MediaCodec codec, int index, @NonNull MediaCodec.BufferInfo info) {
     //拿到编码后的内容
     ByteBuffer encodedData = videoEncoder.getOutputBuffer(index);
     if (encodedData == null) {
        throw new RuntimeException("couldn't fetch buffer at index " + index);
     }

     //由于使用了 MediaMuxer 对于 配置信息 这里可以忽略
     if ((info.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
         info.size = 0;
     }
     if (info.size != 0) {
        if (muxerStarted) {
          //获取到编码后的内容，写到  MediaMuxer 中
          encodedData.position(info.offset);
          encodedData.limit(info.offset + info.size);
          muxer.writeSampleData(trackIndex, encodedData, info);
       }
     }
     //最后的释放操作
     videoEncoder.releaseOutputBuffer(index, false);
   }

   @Override
   public void onError(@NonNull MediaCodec codec, @NonNull MediaCodec.CodecException e) {
      Log.e(TAG, "MediaCodec " + codec.getName() + " onError:", e);
   }

   /**
   * 在  onOutputFormatChanged 的时候，来初始化 MediaMuxer
   * @param codec
   * @param format
   */
   @Override
   public void onOutputFormatChanged(@NonNull MediaCodec codec, @NonNull MediaFormat format) {
     Log.d(TAG, "Output Format changed ==  " + Thread.currentThread().getName());
     //如果之前有添加过，直接抛出异常
     if (trackIndex >= 0) {
        throw new RuntimeException("format changed twice");
     }
     //添加tracker
     trackIndex = muxer.addTrack(videoEncoder.getOutputFormat());
     //启动 MediaMuxer
     if (!muxerStarted && trackIndex >= 0) {
        muxer.start();
        //标识 MediaMuxer 启动完成
        muxerStarted = true;
      }
    }
  };
}
当然这里的异步并不是说会在其他的线程上执行，只是不是同步调用
```

- CameraCaptureActivity

```java
CameraCaptureActivity 是一个使用摄像头来录制视频，画面的预览使用 GLSurfaceView 也就是预览画面是在另一个线程，然后视频的编码又是在另一个线程中，
同时还可以支持设置滤镜的效果，比如黑白，边缘等效果
    
这里涉及到了摄像头的操作，由于这里的摄像头打开是在主线程中的
protected void onResume() {
   Log.d(TAG, "onResume -- acquiring camera");
   super.onResume();
   //更新按钮的状态
   updateControls();

   //判断当前是否有相机的权限,如果没有申请权限，如果有直接打开摄像机
   if (PermissionHelper.hasCameraPermission(this)) {
        if (mCamera == null) {
            openCamera(1280, 720);      // updates mCameraPreviewWidth/Height
         }
    } else {
        //申请权限
         PermissionHelper.requestCameraPermission(this, false);
    }
    //通知GLSurfaceView 执行 onResume 生命周期
    //渲染方式，RENDERMODE_WHEN_DIRTY表示被动渲染，只有在调用requestRender或者onResume等方法时才会进行渲染。RENDERMODE_CONTINUOUSLY表示持续渲染
     mGLView.onResume();
     //通过 queueEvent 将下面的操作 转到 mGLView 对应的渲染线程中 执行
     mGLView.queueEvent(new Runnable() {
       @Override public void run() {
           //通知渲染对象，相机的预览大小
           mRenderer.setCameraPreviewSize(mCameraPreviewWidth, mCameraPreviewHeight);
        }
     });
     Log.d(TAG, "onResume complete: " + this);
}
所以后续操作这个摄像头的话，也要在主线程中进行，不能在其他的线程中，比如这里的画面预览是使用 GLSurfaceView内部是封装了EGL上下文的而且是在单独的一个线程中的，
他的绘制方式是通过设置渲染对象来进行的,比如这里的 CameraSurfaceRenderer 就是渲染对象
    
mGLView = (GLSurfaceView) findViewById(R.id.cameraPreview_surfaceView);
mGLView.setEGLContextClientVersion(2);     // select GLES 2.0  设置 EGL 上下文的版本号
mRenderer = new CameraSurfaceRenderer(mCameraHandler, sVideoEncoder, outputFile);//创建 渲染对象
mGLView.setRenderer(mRenderer);//给GLSurfaceView 设置渲染对象

//默认渲染方式为RENDERMODE_CONTINUOUSLY，当设置为RENDERMODE_CONTINUOUSLY时渲染器会不停地渲染场景，
//当设置为RENDERMODE_WHEN_DIRTY时只有在创建和调用requestRender()时才会刷新。
mGLView.setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY); //设置渲染的模式

之后我们可以在对应的渲染对象回调里面执行各种操作，比如在 onSurfaceCreated 回调中创建OES的纹理并且使用SurfaceTexture封装然后通过主线程的Handle
发送消息给主线程通知可以设置相机的预览了

//创建OES 对应的纹理对象
mTextureId = mFullScreen.createTextureObject();
//创建一个SurfaceTexture使用一个外部的纹理，在这个上下文中，我们在这个线程中我们没有设置Loop，GLSurfaceView没有创建，所以这个帧可用的消息要发送给主线程
mSurfaceTexture = new SurfaceTexture(mTextureId);
// Tell the UI thread to enable the camera preview.  告诉 UI线程，相机可以预览了
mCameraHandler.sendMessage(mCameraHandler.obtainMessage(CameraCaptureActivity.CameraHandler.MSG_SET_SURFACE_TEXTURE,mSurfaceTexture));

主线程收到了通知后，设置相机的预览，所以这里要注意的点就是，在哪个线程操作摄像机就要在哪个线程继续操作，其他线程是不允许的.
private void handleSetSurfaceTexture(SurfaceTexture st) {
	//首先给SurfaceTexture 设置 页面的监听回调
    st.setOnFrameAvailableListener(this);
    try {
        //设置预览控件
        mCamera.setPreviewTexture(st);
    } catch (IOException ioe) {
        throw new RuntimeException(ioe);
    }
    //开始预览
    mCamera.startPreview();
}

后续相机捕获到了数据就会回调执行 onFrameAvailable 方法，那么我们就可以在这个方法中调用mGLView的 requestRender方法
//渲染方式，RENDERMODE_WHEN_DIRTY表示被动渲染，只有在调用requestRender或者onResume等方法时才会进行渲染。RENDERMODE_CONTINUOUSLY表示持续渲染
mGLView.requestRender();

这里还要注意的一个点就是我们不能直接调用GLSurfaceView的方法，我们通过 queueEvent 将后续的操作转到GLSurfaceView的线程中才可以,比如当打开相机的时候获取到预览大小的时候
//通过 queueEvent 将下面的操作 转到 mGLView 对应的渲染线程中 执行
mGLView.queueEvent(new Runnable() {
    @Override public void run() {
         //通知渲染对象，相机的预览大小
         mRenderer.setCameraPreviewSize(mCameraPreviewWidth, mCameraPreviewHeight);
      }
});
之后在 GLSurfaceView的 onDrawFrame回调中,执行绘制操作
首先调用  updateTexImage 将相机的内容刷新到纹理上面
// was there before.   调用刷新的方法，这样相机的画面数据就到了纹理上面，注意，则个一定要在渲染线程里面调用
mSurfaceTexture.updateTexImage();

后续就可以界面的绘制了,这样内容就绘制出来了
// Draw the video frame.  获取到 SurfaceTexture 中的矩阵
mSurfaceTexture.getTransformMatrix(mSTMatrix);
//执行绘制操作,这样预览的内容就显示出来了
mFullScreen.drawFrame(mTextureId, mSTMatrix);    

对于视频的编码阶段,我们要共享这个预览线程的EGL上下文，下面是编码线程的创建
// start recording  由于 mRecordingStatus 在 onSurfaceCreated 的时候 被初始化成 RECORDING_OFF，后续录制的时候，也不会改变到，所以这里执行开始录制操作
//要注意，这里使用了共享EGL上下文，这里的编码线程的上下文跟预览线程的上下文是同一个，因为这样才能做到纹理的拷贝共享等操作
mVideoEncoder.startRecording(new TextureMovieEncoder.EncoderConfig(mOutputFile, 640, 480, 100000,EGL14.eglGetCurrentContext()));

创建的过程
public void startRecording(EncoderConfig config) {
    Log.d(TAG, "Encoder: startRecording()");
    synchronized (mReadyFence) {
       //如果当前已经在运行了，直接返回
       if (mRunning) {
             Log.w(TAG, "Encoder thread already running");
             return;
        }
        //否则标识当前 线程正在运行
        mRunning = true;
        //启动编码线程
        new Thread(this, "TextureMovieEncoder").start();
        while (!mReady) {
            try {
                 //等待编码线程启动完成
                 mReadyFence.wait();
             } catch (InterruptedException ie) {
                 // ignore
              }
            }
        }
    //如果执行到了这里，那么编码线程肯定启动完成，那么mHandle肯定已经完成了，初始化操作了,此时发送 MSG_START_RECORDING 消息，通知 编码子线程完成录制的准备操作
    mHandler.sendMessage(mHandler.obtainMessage(MSG_START_RECORDING, config));
}

编码子线程收到了MSG_START_RECORDING 完成编码子线程的初始化操作,比如创建EGL，绑定EGLSurface，绑定EGL上下文等
private void prepareEncoder(EGLContext sharedContext, int width, int height, int bitRate, File outputFile) {
   try {
       //首先根据 要编码视频的宽度，高度，比特率创建对应的 MediaCodec 编码器
       mVideoEncoder = new VideoEncoderCore(width, height, bitRate, outputFile);
    } catch (IOException ioe) {
       throw new RuntimeException(ioe);
   }
   //根据传递进来的EGL 上下文，创建当前线程对应的 EGLCore
   mEglCore = new EglCore(sharedContext, EglCore.FLAG_RECORDABLE);
   //创建当前线程对应的 WindowSurface
   mInputWindowSurface = new WindowSurface(mEglCore, mVideoEncoder.getInputSurface(), true);
   //绑定操作,关联起来
   mInputWindowSurface.makeCurrent();
   //创建 内部创建的也是 GL_TEXTURE_EXTERNAL_OES 纹理
   mFullScreen = new FullFrameRect(new Texture2dProgram(Texture2dProgram.ProgramType.TEXTURE_EXT));
}
后续至于怎么编码的就不用看了，无非就是绘制的内容再绘制到 MediaCodec的InputSurface中
```

- TextureViewGLActivity

```java
TextureViewGLActivity 是一个简单的demo，是使用TextureView来显示绘制的画面，然后在渲染子线程里面通过获取Surface获取到对应Canvas，
然后通过java的方式来绘制的操作，跟TextureViewCanvasActivity 是非常相似的，不同的地方是在销毁的方式上
    
比如 TextureViewCanvasActivity 类中销毁的函数为    
@Override   // will be called on UI thread  销毁，这里直接返回true，代表要销毁,这是主线程的调用
public boolean onSurfaceTextureDestroyed(SurfaceTexture st) {
     Log.d(TAG, "onSurfaceTextureDestroyed");
     //销毁的时候，重置默认值
    synchronized (mLock) {
        mSurfaceTexture = null;
    }
    return true;
}
可以看到这里是每次销毁的时候都会返回 true，而且这个回调方法的调用是在主线程里面的，而 mSurfaceTexture 是子线程的变量，在渲染子线程中有这样的绘制操作
// Create a Surface for the SurfaceTexture.  根据SurfaceTexture 创建 Surface
Surface surface = null;
synchronized (mLock) {
     SurfaceTexture surfaceTexture = mSurfaceTexture;
     if (surfaceTexture == null) {
          Log.d(TAG, "ST null on entry");
          return;
     }
    surface = new Surface(surfaceTexture);
}
while (true) {
    Rect dirty = null;
    if (partial) {      
        dirty = new Rect(0, mHeight * 3 / 8, mWidth, mHeight * 5 / 8);
    }
    //获得Canvas，lockCanvas(null) 就是锁住整个画布，绘画完成后也更新整张画布的内容到屏幕上，这个没有什么问题，而lockCanvas(Rect dirty)就是
    //锁住画布中的某个区域，绘画完成后也只更新这个区域的内容到屏幕，使用后一接口的初衷是只更新必要的画面内容以节省时间，提高程序运行的效率
    //适用于大动态画面的场景
    Canvas canvas = surface.lockCanvas(dirty);//有时候会出现闪退 FATAL EXCEPTION: TextureViewCanvas Renderer
    if (canvas == null) {
        Log.d(TAG, "lockCanvas() failed");
        break;
    }
    ...
}
如果主线程的TextureView执行了销毁操作，那么这边就会有问题，因为对于渲染子线程来说，完全有可能因为 Surface 的不可用导致闪退,大致可以理解为主线程销毁了，
子线程还在使用之前的变量

接着看下 TextureViewGLActivity 
@Override   // will be called on UI thread  这是在主线程的调用
public boolean onSurfaceTextureDestroyed(SurfaceTexture st) {
   Log.d(TAG, "onSurfaceTextureDestroyed");
   synchronized (mLock) {
        mSurfaceTexture = null;
    }
   //销毁的时候，返回 true 代表完成销毁，否则没有,sReleaseInCallback 默认为 true,如果 sReleaseInCallback 为false代表要自己手动的销毁，所以当前的这个方法，不能返回true
   if (sReleaseInCallback) {
       Log.i(TAG, "Allowing TextureView to release SurfaceTexture");
    }
    return sReleaseInCallback;
}

sReleaseInCallback默认是为ture的，现在看下正常模式下的退出  
// Must be static or it'll get reset on every Activity pause/resume.  标识当前是否是使用了 Callback的回调方式，默认为 true
private static volatile boolean sReleaseInCallback = true;

 // Render frames until we're told to stop or the SurfaceTexture is destroyed.  执行动画的绘制
doAnimation(windowSurface);

//释放操作，因为上面的 doAnimation是无限循环
windowSurface.release();
mEglCore.release();

//如果sReleaseInCallback 为 false 代表需要我们手动的执行释放操作，所以要手动的调用 release方法
if (!sReleaseInCallback) {
     Log.i(TAG, "Releasing SurfaceTexture in renderer thread");
     surfaceTexture.release();
}

而在 doAnimation 方法中有这样的判断,可以看出来就算主线程执行了销毁的操作，这里由于每次都会判断 surfaceTexture 是否为空
//无限循环
while (true) {
     // Check to see if the TextureView's SurfaceTexture is still valid.
     synchronized (mLock) {
         SurfaceTexture surfaceTexture = mSurfaceTexture;
         if (surfaceTexture == null) {
               Log.d(TAG, "doAnimation exiting");
               return;
          }
    }
// Still alive, render a frame.
GLES20.glClearColor(clearColor, clearColor, clearColor, 1.0f);
GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
...
// 绘制显示的内容,注意这个方法，如果 这个 SurfaceTexture 已经被销毁了，这个方法的调用将会抛出一个异常
//WARNING: swapBuffers() failed 这就是由系统来释放的时候，出现的警告
eglSurface.swapBuffers();

虽然有实时的判断 mSurfaceTexture 是否为空，但是还会出现问题，在执行 eglSurface.swapBuffers();的时候会出现WARNING: swapBuffers() failed
还是一样主线程在销毁的时候，子线程可能刚好执行到了swap操作等，所以会出现问题
    
我们看下手动释放的情况，手动释放就是点击了toggleRelease_button 更改 sReleaseInCallback的值
/**
* onClick handler for toggleRelease_button.  按钮的点击事件，这里是切换回调的模式
*/
public void clickToggleRelease(View unused) {
    sReleaseInCallback = !sReleaseInCallback;
    updateControls();
}
回到前面的onSurfaceTextureDestroyed返回 就会因为 sReleaseInCallback为false，把这个函数的返回值设置为false，代表系统不帮我们释放，回到前面的
while(tue){
    ...
}
循环中，因为我们在主线程中没有销毁，所以在执行 swap的操作的时候不会出现警告，而且由于推出的时候设置了mSurfaceTexture == null,所以可以让子线程安全的退出
后续在渲染子线程中，再来判断是否需要手动的释放，所以这不会出现问题
 
//如果sReleaseInCallback 为 false 代表需要我们手动的执行释放操作，所以要手动的调用 release方法
if (!sReleaseInCallback) {
     Log.i(TAG, "Releasing SurfaceTexture in renderer thread");
     surfaceTexture.release();
}
```

- TextureFromCameraActivity

```java
TextureFromCameraActivity 是一个使用了前置摄像头来捕获像素数据，然后提供了旋转，放大画面，放到或者缩小画面的内容，还可以响应手指的移动，
手指到哪里画面也会跟着动，相机的预览显示就不看了，看下是怎么样实现 旋转等操作的
 
在surfaceChanged 回调中，通知渲染的子线程初始化变量,比如创建正交投影矩阵
private void finishSurfaceSetup() {
   int width = mWindowSurfaceWidth;
   int height = mWindowSurfaceHeight;
   Log.d(TAG, "finishSurfaceSetup size=" + width + "x" + height + " camera=" + mCameraPreviewWidth + "x" +mCameraPreviewHeight);

   // Use full window.  设置全屏
   GLES20.glViewport(0, 0, width, height);

   // Simple orthographic projection, with (0,0) in lower-left corner.  获得一个正交投影矩阵，可以将显示的物体按照比率进行缩放
   Matrix.orthoM(mDisplayProjectionMatrix, 0, 0, width, 0, height, -1, 1);

   // Default position is center of screen.  设置中心点的坐标
   mPosX = width / 2.0f;
   mPosY = height / 2.0f;

   //更新数值
  updateGeometry();

   // Ready to go, start the camera.
   Log.d(TAG, "starting camera preview");
   try {
       //设置预览
       mCamera.setPreviewTexture(mCameraTexture);
    } catch (IOException ioe) {
       throw new RuntimeException(ioe);
    }
    //开启预览
    mCamera.startPreview();
}

private void updateGeometry() {
  //SurfaceView 预览宽高的大小
  int width = mWindowSurfaceWidth;
  int height = mWindowSurfaceHeight;

   //取宽和高的最小值
   int smallDim = Math.min(width, height);
   // Max scale is a bit larger than the screen, so we can show over-size. 最大尺寸比屏幕大一点，所以我们可以显示超大尺寸。
   //根据缩放的比率，求得 对应的缩放之后的比率值
   float scaled = smallDim * (mSizePercent / 100.0f) * 1.25f;

    //摄像机 预览宽高的比，重新算出 新的宽高比
    float cameraAspect = (float) mCameraPreviewWidth / mCameraPreviewHeight;
    //此方法返回的参数最接近的整数.
    int newWidth = Math.round(scaled * cameraAspect);
    int newHeight = Math.round(scaled);

     //算法放大的比率
     float zoomFactor = 1.0f - (mZoomPercent / 100.0f);
     //算出旋转的角度
     int rotAngle = Math.round(360 * (mRotatePercent / 100.0f));

     /* Log.d(TAG,"updateGeometry scaled == " + scaled + " == zoomFactor == " + zoomFactor + " == Angle ==" + rotAngle
       + " == newWidth == " + newWidth + " == newHeight ==" + newHeight);*/
    
     //将这些缩放的值，平移的位置，旋转角度 设置到 mRect
     mRect.setScale(newWidth, newHeight);
     //设置平移的点为中心点的距离
     mRect.setPosition(mPosX, mPosY);
     mRect.setRotation(rotAngle);
     //设置缩放的比率
     mRectDrawable.setScale(zoomFactor);
     ...
}

之后执行绘制的操作
public void draw(Texture2dProgram program, float[] projectionMatrix) {
   // Compute model/view/projection matrix. 矩阵乘法Matrix.multiplyMM();
   //矩阵相乘，这里将 模型矩阵 * 投影矩阵 ，所以这个顺序不能改变 注意乘法要从右向左读 gl_Position = projection * view * model * vec4(aPos, 1.0);
    Matrix.multiplyMM(mScratchMatrix, 0, projectionMatrix, 0, getModelViewMatrix(), 0);

    //最终执行绘制操作
    program.draw(mScratchMatrix, mDrawable.getVertexArray(), 0,
                mDrawable.getVertexCount(), mDrawable.getCoordsPerVertex(),
                mDrawable.getVertexStride(), GlUtil.IDENTITY_MATRIX, mDrawable.getTexCoordArray(),
                mTextureId, mDrawable.getTexCoordStride());
}
private void recomputeMatrix() {
   float[] modelView = mModelViewMatrix;
   //设置为单位矩阵,也即是每个索引对应的值都为1
   Matrix.setIdentityM(modelView, 0);

   //设置平移，这里设置的平移为 mPosX,mPosY   表示为  Matrix.translateM(m, offset, x, y, z)
   Matrix.translateM(modelView, 0, mPosX, mPosY, 0.0f);
   //如果含有角度
   if (mAngle != 0.0f) {
       //设置旋转角度  Matrix.rotateM(m, offset, a, x, y, z) 旋转
        Matrix.rotateM(modelView, 0, mAngle, 0.0f, 0.0f, 1.0f);
   }
   //最后再设置缩放配置,Matrix.scaleM(m,offset,x,y,z) m 表示原矩阵，offset表示矩阵数组是用m的哪个index开始，x,y,z 表示缩放因子，等价于 m * (x,y,z)
   Matrix.scaleM(modelView, 0, mScaleX, mScaleY, 1.0f);
   //同时表示
   mMatrixReady = true;
}
其实就是前面说的设置对应的旋转，平移，缩放，然后跟正交投影矩阵得到最终的结果，然后改变 片段的像素位置，前面有介绍过，这里看下是怎么样是实现画面的放大缩小
这其实就是通过  mRectDrawable.setScale(zoomFactor); 来决定的，通过设置缩放比例之后，然后在后续获取到纹理缓冲的时候，应用上去
    
public FloatBuffer getTexCoordArray() {
    if (mRecalculate) {
       //Log.v(TAG, "Scaling to " + mScale);
       FloatBuffer parentBuf = super.getTexCoordArray();
       int count = parentBuf.capacity();

       if (mTweakedTexCoordArray == null) {
           ByteBuffer bb = ByteBuffer.allocateDirect(count * SIZEOF_FLOAT);
           bb.order(ByteOrder.nativeOrder());
           mTweakedTexCoordArray = bb.asFloatBuffer();
        }
        // Texture coordinates range from 0.0 to 1.0, inclusive.  We do a simple scale
        // here, but we could get much fancier if we wanted to (say) zoom in and pan
        // around.
        //纹理坐标范围从0.0到1.0。我们在这里做一个简单的缩放，但是如果我们想放大和平移的话，我们可以做得更好
        FloatBuffer fb = mTweakedTexCoordArray;
        float scale = mScale;
        //这里将纹理每个索引都按照缩放的比率重新进行计算，就可以得到放大的效果，比如从 1.0 变成 0.5 会放大一倍
        for (int i = 0; i < count; i++) {
             float fl = parentBuf.get(i);
             fl = ((fl - 0.5f) * scale) + 0.5f;
             fb.put(i, fl);
        }
        mRecalculate = false;
    }
    return mTweakedTexCoordArray;
}   

对于放大其实就是将正常的纹理数据做一个缩放就可以实现，缩小的话，就放大
```


### 总结
总体而言，这个项目还是非常不错的，非常有参考，学习的意义


### 参考链接
1. [OpenGL中的移动、缩放、旋转](https://ejin66.github.io/2018/11/02/opengl-study-2.html)
2. [Android学习随笔(001) 简谈lockCanvas(Rect dirty)](https://blog.csdn.net/gymsun/article/details/8516823)
3. [OpenGL Frame Buffer管理](https://blog.csdn.net/dayenglish/article/details/51385510)
4. [第三十五课 延迟渲染（一）](https://wiki.jikexueyuan.com/project/modern-opengl-tutorial/tutorial35.html)
5. [LOpenGL FBO (Frame Buffer Object) 帧缓冲对象](https://blog.csdn.net/davidsu33/article/details/11918335)
6. [SurfaceView, TextureView, SurfaceTexture等的区别](https://zhooker.github.io/2018/03/24/SurfaceTexture%E7%9A%84%E5%8C%BA%E5%88%AB/)
7. [坐标系统](https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/)
8. [帧缓冲](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/05%20Framebuffers/)
9. [混合](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/03%20Blending/)
