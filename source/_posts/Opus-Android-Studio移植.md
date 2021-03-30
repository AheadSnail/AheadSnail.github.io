---
layout: pager
title: Opus Android Studio移植
date: 2018-07-06 11:10:43
tags: [Opus,AndroidStudio,NDK]
description:  Opus Android Studio移植
---
### 简介
> 这篇文章会介绍怎么在Android Studio中编译Opus，至于为什么要在Android Studio中编译，是为了营造一个可以直接调试C代码的环境，这样可以方便我们开发人员查看代码，调试代码
前面一篇文章已经介绍了在Ubuntu 使用NDKr14-b来编译最新版的Opus，而且验证了在手机上是可以正常运行的，而且我们知道opus-tools，需要依赖ogg,flac,opus库，这些库对于我们
使用者来说，是完全没有必要去修改的，所以我们直接用Ubuntu中交叉编译的内容，然后查看他们是怎么编译的，需要什么样的参数，采集然后填充到我们的项目中，自己来编译
下面会采用ndk-build的模式来编译，至于为什么不采用cmakeList来编译，是因为使用cmakeList编译的时候，出现了录的声音在播放的时候会有停顿，但是采用ndk-build来编译就没有这个问题
至于为什么会这样，到目前为止我也没有发现，我猜可能是clang的问题，因为我使用ndk-build编译采用的是gcc，这两者都可以支持调试代码，都可以将生成的so自动的打包进apk，不同的地方就是cmakeList的语法，可能更加简单，毕竟Google 原本采用的是ndk-build，后才出来这个CmakeList

### Android Studio移植
> 我们将Ubuntu交叉编译之后产生的local,opus-tools文件,拿出来,创建一个NDK工程，我们直接创建一个CmakeList的NDK项目,将我们需要的文件拷贝到工程中
这里我们需要拷贝生成的lib目录，已经include目录，这里都只要oput-tools的依赖文件，ogg,flac,opus 还有直接将opus-tools的源码直接包含进来,
接着创建一个Android.mk文件，以及Application.mk文件,下面是文件的结构图

目录结构图:
![](/uploads/Opus交叉编译/OpusAndroidStudio移植.jpg)

1.修改app目录下的build.gradle文件

```cpp
defaultConfig{
    ...
    externalNativeBuild {
        ndkBuild {
	    //指定构建的架构
	    abiFilters "armeabi-v7a"
    }
}
	
//指定JNI的目录
sourceSets.main.jniLibs.srcDirs = ['/src/main/jni/']

externalNativeBuild {
        ndkBuild {
            //指定Android.mk文件的目录
            path "/src/main/jni/Android.mk"
    }
}
```

由于最新版的ndk_bundle 不支持了armeabi，如果构建一个armeabi 就会出现下面的这种错误
ABIs [armeabi] are not supported for platform. Supported ABIs are [armeabi-v7a, arm64-v8a, x86, x86_64]. 所以我们可以直接构建一个armeabi-v7a

下面是Android.mk的内容

```makefile
#此变量用于指定当前文件的路径。必须在 Android.mk 文件的开头定义它。 以下示例向您展示如何操作： LOCAL_PATH := $(call my-dir)
#CLEAR_VARS 指向的脚本不会清除此变量。因此，即使您的 Android.mk 文件描述了多个模块，您也只需定义它一次。

LOCAL_PATH := $(call my-dir)

#可以打印信息 E:/sdk/ndk-bundle/build//../build/core 而.代表当前的Android.mk路径
#LOCAL_PATH  H:/AndroidStudioWorkSpace3/OpusNativeBuild/app/src/main/jni 也即是当前Android.mk文件的路径
#要注意使用LOCAL_PATH的时候，不能$LOCAL_PATH这样使用，会打印 OCAL_PATH 要${LOCAL_PATH}或者$(LOCAL_PATH)
$(warning ${LOCAL_PATH})

#此变量指向的构建脚本用于取消定义下面“开发者定义的变量”一节中列出的几乎全部 LOCAL_XXX 变量。 在描述新模块之前，使用此变量包括此脚本
include $(CLEAR_VARS)

#当前模考的名称
LOCAL_MODULE    := FLAc

#.代表当前的目录，当前的目录是前面$(call my-dir)指定的跟android.mk文件同一级别的
LOCAL_SRC_FILES := ${LOCAL_PATH}/lib/libFLAC.a

#一个 BUILD_SHARED_LIBRARY 变量用于编译一个静态库。静态库不会复制到的APK包中，但是能够用于编译共享库。
# 根据您是使用共享 (.so) 还是静态 (.a) 库，添加 PREBUILT_SHARED_LIBRARY 或 PREBUILT_STATIC_LIBRARY。
include $(PREBUILT_STATIC_LIBRARY)

#在编译完一个模块之后，有必要调用CLEAR_VARS 用来清除之前使用的 LOCAL_XXX 变量 ，防止影响当前的模块的内容
include $(CLEAR_VARS)

LOCAL_MODULE    := Ogg

LOCAL_SRC_FILES := ${LOCAL_PATH}/lib/libogg.a

include $(PREBUILT_STATIC_LIBRARY)


include $(CLEAR_VARS)

LOCAL_MODULE    := Opus

LOCAL_SRC_FILES := ${LOCAL_PATH}/lib/libopus.a

include $(PREBUILT_STATIC_LIBRARY)

#此变量指向的构建脚本用于取消定义下面“开发者定义的变量”一节中列出的几乎全部 LOCAL_XXX 变量。（模块之前使用这个，会清除LOCAL的系统变量） 在描述新模块之前
include $(CLEAR_VARS)

LOCAL_MODULE        := opustool

#这是要编译的源代码文件列表。 只要列出要传递给编译器的文件
LOCAL_SRC_FILES     := \
        ${LOCAL_PATH}/opustools/src/opus_header.c \
        ${LOCAL_PATH}/opustools/src/picture.c \
        ${LOCAL_PATH}/opustools/src/resample.c \
        ${LOCAL_PATH}/opustools/src/audio-in.c \
        ${LOCAL_PATH}/opustools/src/diag_range.c \
        ${LOCAL_PATH}/opustools/src/flac.c \
        ${LOCAL_PATH}/opustools/src/lpc.c \
        ${LOCAL_PATH}/opustools/win32/unicode_support.c \
        ${LOCAL_PATH}/opustools/src/wav_io.c \
        ${LOCAL_PATH}/opustools/src/opusinfo.c \
        ${LOCAL_PATH}/opustools/src/info_opus.c \
        ${LOCAL_PATH}/opustools/src/mydecoder.c \
        ${LOCAL_PATH}/opustools/src/myencode.c


#可以使用此可选变量指定相对于 NDK root 目录的路径列表，以便在编译所有源文件（C、C++ 和 Assembly）时添加到 include 搜索路径。
#在通过 LOCAL_CFLAGS 或 LOCAL_CPPFLAGS 设置任何对应的 include 标志之前定义此变量。在使用 ndk-gdb 启动本地调试时，构建系统也会自动使用 LOCAL_C_INCLUDES 路径。
#也即是要在设置 LOCAL_CFLAGS 或者 LOCAL_CPPFLAGS 之前使用
LOCAL_C_INCLUDES := ${LOCAL_PATH}/include ${LOCAL_PATH}/include/opus ${LOCAL_PATH}/opustools/include

#“:=” 的意思是，它右边赋得值如果是变量，只能使用在这条语句之前定义好的，而不能使用本条语句之后定义的变量；
#“=”，当它的右边赋值是变量时，这个变量的定义在本条语句之前或之后都可以；

#此可选变量为构建系统设置在构建 C 和 C++ 源文件时要传递的编译器标志。 此功能对于指定额外的宏定义或编译选项可能很有用。是全局的引用，不是模块化的，只要设置了，对于后面的模块也是有效的
LOCAL_CFLAGS 	:= -w -std=c11 -Os -fno-strict-aliasing -fprefetch-loop-arrays
LOCAL_CFLAGS 	+= -DHAVE_CONFIG_H -DOPUSTOOLS -DSPX_RESAMPLE_EXPORT= -DRANDOM_PREFIX=opustools -DOUTSIDE_SPEEX -DFLOATING_POINT -fno-math-errno

#仅当构建 C++ 源文件时才会传递一组可选的编译器标志。 它们将出现在编译器命令行中的 LOCAL_CFLAGS 后面。
LOCAL_CPPFLAGS 	:= -DBSD=1 -ffast-math -Os -funroll-loops -std=c++11
LOCAL_CPPFLAGS  += -DHAVE_CONFIG_H -DOPUSTOOLS -DSPX_RESAMPLE_EXPORT= -DRANDOM_PREFIX=opustools -DOUTSIDE_SPEEX -DFLOATING_POINT

#此变量包含在构建共享库或可执行文件时要使用的其他链接器标志列表。 它可让您使用 -l 前缀传递特定系统库的名称。 例如，以下示例指示链接器生成在加载时链接到 /system/lib/libz.so
#的模块： LOCAL_LDLIBS := -lz
LOCAL_LDLIBS 	:= -llog

#LOCAL_STATIC_LIBRARIES此变量用于存储当前模块依赖的静态库模块列表。如果当前模块是共享库或可执行文件，此变量将强制这些库链接到生成的二进制文件。如果当前模块是静态库，
此变量只是指示，依赖当前模块的模块也会依赖列出的库
#LOCAL_SHARED_LIBRARIES此变量是此模块在运行时依赖的共享库模块列表。 此信息在链接时需要，并且会在生成的文件中嵌入相应的信息。
LOCAL_STATIC_LIBRARIES := FLAc Ogg Opus

#指向编译脚本，根据所有的在 LOCAL_XXX 变量把列出的源代码文件编译成一个共享库。 注意，必须至少在包含这个文件之前定义 LOCAL_MODULE 和 LOCAL_SRC_FILES。
include $(BUILD_SHARED_LIBRARY)
```

下面是Appliation.mk文件的内容

```makefile
APP_PLATFORM := android-14
APP_ABI := armeabi-v7a
NDK_TOOLCHAIN_VERSION := 4.9
APP_STL := gnustl_static

#APP_OPTIM 将此可选变量定义为 release 或 debug。在构建应用的模块时可使用它来更改优化级别。
#APP_CFLAGS 此变量用于存储构建系统在为任何模块编译任何 C 或 C++ 源代码时传递到编译器的一组 C 编译器标志 也即是全局的
#APP_LDFLAGS 此变量包含构建系统在仅构建 C++ 源文件时传递到编译器的一组 C++ 编译器标志 也即是全局的
#APP_PLATFORM  此变量包含目标 Android 平台的名称。例如，android-3 指定 Android 1.5 系统映像
#APP_STL  默认情况下，NDK 构建系统为 Android 系统提供的最小 C++ 运行时库 (system/lib/libstdc++.so) 提供 C++ 标头
#NDK_TOOLCHAIN_VERSION 将此变量定义为 4.9 或 4.8 以选择 GCC 编译器的版本。 64 位 ABI 默认使用版本 4.9 ，32 位 ABI 默认使用版本 4.8
```

### 源码修改
从Android.mk文件中有俩个文件是经过修改过后的
${LOCAL_PATH}/opustools/src/mydecoder.c
${LOCAL_PATH}/opustools/src/myencode.c

> 原本的文件为decoder.c encoder.c，对于decoder.c提供的功能可以将一个opus文件，变成一个wav文件，对于encoder.c文件，可以将一个wav文件变成opus文件
但是我们在使用的时候，并不会说提供一个wav文件，然后让你去压缩变成一个opus文件，播放语音的时候也不会说提供一个opus文件，经过解压缩变成一个wav文件，然后再来播放这个wav文件
这样太慢了，对于解压缩，跟压缩的过程都是有时间延迟的，而且时长越大，延迟也就越大，所以我们需要修改原本的代码,建议在Visual Studio中阅读代码，然后修改,Visual Studio是在Pc上
而如果采用NDK中调试，修改代码属于手机端，对应方便来说肯定是PC方便，而且Visual Studio 用来开发c/c++ 是非常强大的，非常方便的 ,而且从git clone的代码来看官网对visual Studio默认是支持的
可能人家开发采用的就是Visual Studio ,所以对于前面的文章 Visual Studio中搭建这个环境的必要,下面会直接贴出这些修改文件的全部内容

压缩提供的接口为

```java
private native int nativeInitEncoder(String fileName);

/**
* opus的压缩，要传递原始的录音的大小，这里传递200个字节
* @param sourceIn
* @return
*/
private native int nativeEncodeBytes(byte[] sourceIn,int length);

/**
 * opus压缩的资源的释放
 *  如果返回的值不是0，代表初始化失败
 */
private native int nativeReleaseEncoder();
```

myencode.c实现：

```cpp
/*
 * Class:     vide_m4399_com_newopusdemo_OpusEncoder
 * Method:    nativeInitEncoder
 * Signature: (Ljava/lang/String;)V
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusEncoder_nativeInitEncoder
  (JNIEnv * env , jobject object, jstring value)
 {
     char * destFileName = Jstring2CStr(env,value);
     int ret = 0;
     if (destFileName == NULL)
     {
     	ret = -1;
     	LOGD("MyOpus_Init func errot destFileName == NULL ret == %d\n",ret);
     	return ret;
     }
     //将变量重置默认值
     opus_resert_variably();

     opt_ctls_ctlval = NULL;
     frange = NULL;
     range_file = NULL;
     in_format = NULL;
     inopt.channels = chan;
     inopt.rate = coding_rate = rate;
     /* 0 dB gain is recommended unless you know what you're doing */
     inopt.gain = 0;
     inopt.samplesize = 16;
     inopt.endianness = 0;
     inopt.rawmode = 0;
     inopt.ignorelength = 0;
     inopt.copy_comments = 1;
     inopt.copy_pictures = 1;

     start_time = time(NULL);
     //生成一个随机数种子
     srand(((getpid() & 65535) << 15) ^ start_time);
     //产生一个随机数,用来校验 文件
     serialno = rand();
     //打印版本号
     opus_version = opus_get_version_string();
     /*Vendor string should just be the encoder library,
     the ENCODER comment specifies the tool used.*/
     //指针做函数参数间距的修改主调函数的值
     comment_init(&inopt.comments, &inopt.comments_length, opus_version);
     //最多从源串中拷贝n-1个字符到目标串中，然后再在后面加一个0。所以如果目标串的大小为n的话...,将opusenc from opus-tools 0.1.10写到ENCODER_string数组中
     snprintf(ENCODER_string, sizeof(ENCODER_string), "opusenc from %s %s", PACKAGE_NAME, PACKAGE_VERSION);
     comment_add(&inopt.comments, &inopt.comments_length, "ENCODER", ENCODER_string);

    //压缩完之后的目标文件
    outFile = destFileName;
    inopt.rawmode = 1;
    in_format = &raw_format;
    //设置一些参数
    in_format->open_func(NULL, &inopt, NULL, 0);

    if (inopt.rate<100 || inopt.rate>768000)
	{
		/*Crazy rates excluded to avoid excessive memory usage for padding/resampling.*/
		exit(1);
	}

	if (inopt.channels>255 || inopt.channels<1)
	{
		exit(1);
	}

	if (downmix == 0 && inopt.channels>2 && bitrate>0 && bitrate<(16000 * inopt.channels))
	{
		if (!quiet)LOGD("Notice: Surround bitrate less than 16kbit/sec/channel, downmixing.\n");
		downmix = inopt.channels>8 ? 1 : 2;
	}

	if (downmix > 0 && downmix < inopt.channels)
	{
		downmix = setup_downmix(&inopt, downmix);
	}
	else
	{
		downmix = 0;
	}

	//设置一些默认值
	rate = inopt.rate;
	chan = inopt.channels;
	inopt.skip = 0;

	/*In order to code the complete length we'll need to do a little padding*/
	setup_padder_from_my(&inopt, &original_samples);

	if (rate > 24000)
	{
		coding_rate = 48000;
	}
	else if (rate > 16000)
	{
		coding_rate = 24000;
	}
	else if (rate > 12000)
	{
		coding_rate = 16000;
	}
	else if (rate > 8000)
	{
		coding_rate = 12000;
	}
	else
	{
		coding_rate = 8000;
	}

	//默认的inopt.rate=coding_rate=rate; = 48000
	frame_size = frame_size / (48000 / coding_rate);

	/*Scale the resampler complexity, but only for 48000 output because
	the near-cutoff behavior matters a lot more at lower rates.*/
	if (rate != coding_rate)
	{
		setup_resample(&inopt, coding_rate == 48000 ? (complexity + 1) / 2 : 5, coding_rate);
	}

	if (rate != coding_rate&&complexity != 10 && !quiet)
	{
		LOGD("Notice: Using resampling with complexity<10.\n");
		LOGD("Opusenc is fastest with 48, 24, 16, 12, or 8kHz input.\n\n");
	}

	/*OggOpus headers*/ /*FIXME: broke forcemono*/
	//给header 结构体赋值
	header.channels = chan;
	header.channel_mapping = header.channels>8 ? 255 : chan>2;
	header.input_sample_rate = rate;
	header.gain = inopt.gain;

	/*Initialize Opus encoder*/
	/*Frame sizes <10ms can only use the MDCT modes, so we switch on RESTRICTED_LOWDELAY
	to save the extra 4ms of codec lookahead when we'll be using only small frames.*/
	//OpusMSEncoder      *st;
	st = opus_multistream_surround_encoder_create(coding_rate, chan, header.channel_mapping, &header.nb_streams, &header.nb_coupled,
		header.stream_map, frame_size<480 / (48000 / coding_rate) ? OPUS_APPLICATION_RESTRICTED_LOWDELAY : OPUS_APPLICATION_AUDIO, &ret);

	if (ret != OPUS_OK)
	{
		LOGD("Error cannot create encoder: %s\n", opus_strerror(ret));
		exit(1);
	}

	//min_bytes 跟 max_frame_bytes大小是一样的
	min_bytes = max_frame_bytes = (1275 * 3 + 7)*header.nb_streams;
	//分配packet的内存空间
	packet = malloc(sizeof(unsigned char)*max_frame_bytes);
	if (packet == NULL)
	{
		LOGD("Error allocating packet buffer.\n");
		exit(1);
	}

	if (bitrate<0)
	{
		/*Lower default rate for sampling rates [8000-44100) by a factor of (rate+16k)/(64k)*/
		bitrate = ((64000 * header.nb_streams + 32000 * header.nb_coupled)*
			(IMIN(48, IMAX(8, ((rate<44100 ? rate : 48000) + 1000) / 1000)) + 16) + 32) >> 6;
	}

	if (bitrate>(1024000 * chan) || bitrate<500)
	{
		LOGD("Error: Bitrate %d bits/sec is insane.\nDid you mistake bits for kilobits?\n", bitrate);
		LOGD("--bitrate values from 6-256 kbit/sec per channel are meaningful.\n");
		exit(1);
	}
	bitrate = IMIN(chan * 256000, bitrate);
	ret = opus_multistream_encoder_ctl(st, OPUS_SET_BITRATE(bitrate));
	if (ret != OPUS_OK)
	{
		LOGD("Error OPUS_SET_BITRATE returned: %s\n", opus_strerror(ret));
		exit(1);
	}
	ret = opus_multistream_encoder_ctl(st, OPUS_SET_VBR(!with_hard_cbr));//1
	if (ret != OPUS_OK)
	{
		LOGD("Error OPUS_SET_VBR returned: %s\n", opus_strerror(ret));
		exit(1);
	}
	if (!with_hard_cbr)
	{
		ret = opus_multistream_encoder_ctl(st, OPUS_SET_VBR_CONSTRAINT(with_cvbr));//0
		if (ret != OPUS_OK)
		{
			LOGD("Error OPUS_SET_VBR_CONSTRAINT returned: %s\n", opus_strerror(ret));
			exit(1);
		}
	}
	ret = opus_multistream_encoder_ctl(st, OPUS_SET_COMPLEXITY(complexity));
	if (ret != OPUS_OK)
	{
		LOGD("Error OPUS_SET_COMPLEXITY returned: %s\n", opus_strerror(ret));
		exit(1);
	}
	ret = opus_multistream_encoder_ctl(st, OPUS_SET_PACKET_LOSS_PERC(expect_loss));
	if (ret != OPUS_OK)
	{
		LOGD("Error OPUS_SET_PACKET_LOSS_PERC returned: %s\n", opus_strerror(ret));
		exit(1);
	}
#ifdef OPUS_SET_LSB_DEPTH
	ret = opus_multistream_encoder_ctl(st, OPUS_SET_LSB_DEPTH(IMAX(8, IMIN(24, inopt.samplesize))));
	if (ret != OPUS_OK)
	{
		LOGD("Warning OPUS_SET_LSB_DEPTH returned: %s\n", opus_strerror(ret));
	}
#endif

	/*We do the lookahead check late so user CTLs can change it*/
	ret = opus_multistream_encoder_ctl(st, OPUS_GET_LOOKAHEAD(&lookahead));
	if (ret != OPUS_OK)
	{
		LOGD("Error OPUS_GET_LOOKAHEAD returned: %s\n", opus_strerror(ret));
		exit(1);
	}
	inopt.skip += lookahead;
	/*Regardless of the rate we're coding at the ogg timestamping/skip is
	always timed at 48000.*/
	header.preskip = inopt.skip*(48000. / coding_rate);
	/* Extra samples that need to be read to compensate for the pre-skip */
	inopt.extraout = (int)header.preskip*(rate / 48000.);

	if (strcmp(outFile, "-") == 0)
	{
#if defined WIN32 || defined _WIN32
		_setmode(_fileno(stdout), _O_BINARY);
#endif
		fout = stdout;
	}
	else {
		//打开文件，二进制的方式
		fout = fopen_utf8(outFile, "wb");
		if (!fout)
		{
		        //打开失败
			perror(outFile);
			exit(1);
		}
	}
	/*Initialize Ogg stream struct*/
	if (ogg_stream_init(&os, serialno) == -1)
	{
		LOGD("Error: stream init failed\n");
		exit(1);
	}

	/*Write header*/
	{
		/*The Identification Header is 19 bytes, plus a Channel Mapping Table for
		mapping families other than 0. The Channel Mapping Table is 2 bytes +
		1 byte per channel. Because the maximum number of channels is 255, the
		maximum size of this header is 19 + 2 + 255 = 276 bytes.*/
		unsigned char header_data[276];
		int packet_size = opus_header_to_packet(&header, header_data, sizeof(header_data));
		op.packet = header_data;
		op.bytes = packet_size;
		op.b_o_s = 1;
		op.e_o_s = 0;
		op.granulepos = 0;
		op.packetno = 0;

		ogg_stream_packetin(&os, &op);

		while ((ret = ogg_stream_flush(&os, &og)))
		{
			if (!ret)break;
			/*头单独的做为一个page的存在，写进输出文件里面
			set pointers in the ogg_page struc
			og->header = os->header;
			og->header_len = os->header_fill = vals + 27;
			og->body = os->body_data + os->body_returned;
			og->body_len = bytes;
			*/
			ret = oe_write_page(&og, fout);
			if (ret != og.header_len + og.body_len)
			{
				LOGD("Error: failed writing header to output stream\n");
				exit(1);
			}
			//标记已经写了多少个字节
			bytes_written += ret;
			//页数加一,已经写了一页
			pages_out++;
		}
		//重新的给ogg_packet赋值
		comment_pad(&inopt.comments, &inopt.comments_length, comment_padding);
		op.packet = (unsigned char *)inopt.comments;
		op.bytes = inopt.comments_length;
		op.b_o_s = 0;
		op.e_o_s = 0;
		op.granulepos = 0;
		op.packetno = 1;
		ogg_stream_packetin(&os, &op);
	}
	//写剩余Opus header packets信息
	/* writing the rest of the Opus header packets */
	while ((ret = ogg_stream_flush(&os, &og)))
	{
		if (!ret)
		{
			break;
		}
		ret = oe_write_page(&og, fout);
		if (ret != og.header_len + og.body_len)
		{
			LOGD("Error: failed writing header to output stream\n");
			exit(1);
		}
		bytes_written += ret;
		pages_out++;
	}

	free(inopt.comments);

    //输入分配的内存大小为160 * 1 *4
	input = malloc(sizeof(float)*frame_size*chan);
	if (input == NULL)
	{
		LOGD("Error: couldn't allocate sample buffer.\n");
		exit(1);
	}
	id++;
	return ret;
 }

int MyOpusEncoder(signed char * srcDataBuff, int readSize)
{
	int ret = 0;
	int size_segments, cur_frame_size;
	nb_samples = -1;
	//将读取到的数据，进一步处理
	nb_samples = inopt.read_samples_my(inopt.readdata, input, srcDataBuff, readSize, frame_size);
	total_samples += nb_samples;
	//刚好到了文件的末尾，一点数据都没有读取到
	if (nb_samples == 0)
	{
		op.e_o_s = 1;
		return 0;
	}

	if (start_time == 0)
	{
		start_time = time(NULL);
	}
	//当前的fragme_size
	cur_frame_size = frame_size;
	//到了文件的末尾的话,正常的话，nb_samples跟cur_frame_size，这俩个的值是一样的，小于的话，可能读取到了文件的末尾
	if (nb_samples<cur_frame_size)
	{
		op.e_o_s = 1;
		/*Avoid making the final packet 20ms or more longer than needed.*/
		cur_frame_size -= ((cur_frame_size - (nb_samples>0 ? nb_samples : 1)) / (coding_rate / 50))*(coding_rate / 50);
		/*No fancy end padding, just fill with zeros for now.*/
		//剩余的要填充0
		for (i = nb_samples*chan; i < cur_frame_size*chan; i++)
		{
			input[i] = 0;
		}
		LOGD("Error: arriver record end reflash data length == %d\n",nb_samples);
	}

	/*Encode current frame*/
	VG_UNDEF(packet, max_frame_bytes);
	VG_CHECK(input, sizeof(float)*chan*cur_frame_size);
	//进行压缩数据,返回压缩之后的字节，如果小于0，表示压缩失败,将压缩内容写到packet里面
	nbBytes = opus_multistream_encode_float(st, input, cur_frame_size, packet, max_frame_bytes);
	//如果小于0，说明压缩出现了问题
	if (nbBytes<0)
	{
		LOGD("Encoding failed: %s. Aborting.\n", opus_strerror(nbBytes));
		return 0;
	}
	VG_CHECK(packet, nbBytes);
	VG_UNDEF(input, sizeof(float)*chan*cur_frame_size);
	nb_encoded += cur_frame_size;
	enc_granulepos += cur_frame_size * 48000 / coding_rate;
	//记录已经压缩过的总大小
	total_bytes += nbBytes;
	//一个片段是255个字节，来判断压缩之后是有多少个片段
	size_segments = (nbBytes + 255) / 255;
	peak_bytes = IMAX(nbBytes, peak_bytes);
	min_bytes = IMIN(nbBytes, min_bytes);

	if (frange != NULL)
	{
		opus_uint32 rngs[256];
		OpusEncoder *oe;
		for (i = 0; i<header.nb_streams; i++)
		{
			ret = opus_multistream_encoder_ctl(st, OPUS_MULTISTREAM_GET_ENCODER_STATE(i, &oe));
			ret = opus_encoder_ctl(oe, OPUS_GET_FINAL_RANGE(&rngs[i]));
		}
		save_range(frange, cur_frame_size*(48000 / coding_rate), packet, nbBytes, rngs, header.nb_streams);
	}

	/*Flush early if adding this packet would make us end up with a
	continued page which we wouldn't have otherwise.*/
	//当片段达到255的时候，就要写数据了，缓冲区已满     max_ogg_delay=48000; /*48kHz samples*/
	while ((((size_segments <= 255) && (last_segments + size_segments>255)) || (enc_granulepos - last_granulepos>max_ogg_delay)) &&
#ifdef OLD_LIBOGG
		ogg_stream_flush(&os, &og))
	{
#else
		ogg_stream_flush_fill(&os, &og, 255 * 255))
	{
#endif
		if (ogg_page_packets(&og) != 0)
		{
			last_granulepos = ogg_page_granulepos(&og);
		}
		last_segments -= og.header[26];
		ret = oe_write_page(&og, fout);
		if (ret != og.header_len + og.body_len)
		{
			LOGD("Error: failed writing data to output stream\n");
			exit(1);
		}
		//已经写了多少个字节
		bytes_written += ret;
		//页数加一
		pages_out++;
	}
	//填写packet
	op.packet = (unsigned char *)packet;
	op.bytes = nbBytes;
	op.b_o_s = 0;
	op.granulepos = enc_granulepos;
	if (op.e_o_s)
	{
		/*We compute the final GP as ceil(len*48k/input_rate)+preskip. When a
		resampling decoder does the matching floor((len-preskip)*input_rate/48k)
		conversion, the resulting output length will exactly equal the original
		input length when 0<input_rate<=48000.*/
		op.granulepos = ((original_samples * 48000 + rate - 1) / rate) + header.preskip;
	}
	op.packetno = 2 + id;
	//将当前压缩后的packet的内容，封装到os里面
	ogg_stream_packetin(&os, &op);
	//用来判断是否是片段到了255个
	last_segments += size_segments;

	/*If the stream is over or we're sure that the delayed flush will fire,
	go ahead and flush now to avoid adding delay.*/
	//读取到了文件的末尾
	while ((op.e_o_s || (enc_granulepos + (frame_size * 48000 / coding_rate) - last_granulepos>max_ogg_delay) || (last_segments >= 255)) ?
#ifdef OLD_LIBOGG
		/*Libogg > 1.2.2 allows us to achieve lower overhead by
		producing larger pages. For 20ms frames this is only relevant
		above ~32kbit/sec.*/
		ogg_stream_flush(&os, &og) :
		ogg_stream_pageout(&os, &og))
	{
#else
		ogg_stream_flush_fill(&os, &og, 255 * 255) :
		ogg_stream_pageout_fill(&os, &og, 255 * 255))
	{
#endif
        if(op.e_o_s)
        {
            LOGD("Error: reflash data.\n");
        }
		if (ogg_page_packets(&og) != 0)
		{
			last_granulepos = ogg_page_granulepos(&og);
		}
		last_segments -= og.header[26];
		ret = oe_write_page(&og, fout);
		if (ret != og.header_len + og.body_len)
		{
			LOGD("Error: failed writing data to output stream\n");
			exit(1);
		}
		bytes_written += ret;
		pages_out++;
	}
	return ret;
}

/*
 * Class:     vide_m4399_com_newopusdemo_OpusEncoder
 * Method:    nativeEncodeBytes
 * Signature: ([BI)I
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusEncoder_nativeEncodeBytes
  (JNIEnv * env, jobject object, jbyteArray  encoder_insrc, jint length)
{
    jbyte* opus_data_encoder = (*env)->GetByteArrayElements(env, encoder_insrc, 0);
    int ret = MyOpusEncoder(opus_data_encoder,length);

    (*env)->ReleaseByteArrayElements(env, encoder_insrc, opus_data_encoder, 0);
    if ((*env)->ExceptionCheck(env))
    {
    	return -1;
    }
    return ret;
}

/*
 * Class:     vide_m4399_com_newopusdemo_OpusEncoder
 * Method:    nativeReleaseEncoder
 * Signature: ()V
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusEncoder_nativeReleaseEncoder
  (JNIEnv * env, jobject object)
  {
    int ret = 0;

    //终止的时候，要刷新数据,将最后的数据写进文件,这里是人为的调用这个方法，
    signed char srcDataBuff[320] = {0};
    MyOpusEncoder(srcDataBuff,10);
    //不可以调用下面的方法直接的进行刷新数据,这样会导致，最后面有杂音
    //if (ogg_page_packets(&og) != 0)
    //{
    //		last_granulepos = ogg_page_granulepos(&og);
    //}
    //last_segments -= og.header[26];
    //ret = oe_write_page(&og, fout);
    //if (ret != og.header_len + og.body_len)
    //{
    //	LOGD("Error: failed writing data to output stream\n");
    //	exit(1);
    //}
    //bytes_written += ret;
    //pages_out++;

    LOGD("Debug: opus clear memory begin.\n");
    opus_multistream_encoder_destroy(st);
	ogg_stream_clear(&os);
	free(packet);
	free(input);
	if (opt_ctls)
	{
		free(opt_ctls_ctlval);
	}
	if (rate != coding_rate)
	{
		clear_resample(&inopt);
	}
	clear_padder(&inopt);
	if (downmix)
	{
		clear_downmix(&inopt);
	}
	in_format->close_func(inopt.readdata);
	if (fout)
	{
	    fclose(fout);
	}
	//释放由一开始分配的内存空间Jstring2CStr
	if(outFile)
	{
	    free(outFile);
	}

	if (frange)
	{
	    fclose(frange);
	}
	 LOGD("Debug: opus clear memory success.\n");
	return ret;
}
```

解码对应的java接口

```java
/**
* opus解码器的初始化,初始话成功返回0
*/
private native int nativeInitDecoder();

/**
* opus流的解析.返回解压缩之后的大小
*
* @param in 要解析的数组
* @param readSize 读取的大小
* @param out 解析之后的数组
* @return
*/
private native int nativeDecodeBytes(byte[] in,int readSize,short[] out);

/**
 * opus解码器的释放,释放成功返回0
*/
private native int nativeReleaseDecoder();
```

JNI接口对应的源码实现

```cpp
int MyOpusDecoder(char * srcData,int nb_read,short * destData)
{
	int ret = 0;
    if (srcData == NULL || destData == NULL || nb_read <= 0)
    {
    	ret = -1;
    	LOGD("MyOpusDecoder func error srcData == NULL  descData == NULL || nb_read <= 0 ret == %d\n",ret);
    	return ret;
    }

    char * data;
    /*Get the ogg buffer for writing*/
    data = ogg_sync_buffer(&oy, 200);
    //内存的拷贝
    memcpy(data, srcData, nb_read);
    //标识当前已经读取了多少个字节  return((char *)oy->data + oy->fill);
    ogg_sync_wrote(&oy, nb_read);

    //标识每一页能解析多少个字节
    int pageCount = 0;

    int i;
    /*Loop for all complete pages we got (most likely only one)*/
    //从读取的内容中，找到一个page的内容，最有可能找到一页,只要找到一页的数据才会去解析
    while (ogg_sync_pageout(&oy, &og) == 1)
    {
    	//如果ogg_stream_state os没有被初始化,标识ogg_stream已经被初始化了
    	if (stream_init == 0)
    	{
    		//初始化ogg_stream_state os 分配内存空间，指定文件的唯一值
    		ogg_stream_init(&os, ogg_page_serialno(&og));
    		//标识已经初始化了
    		stream_init = 1;
    	}
    	//比较serialno是否是相等，ogg_stream_state os 的serialno本来就是og的
    	if (ogg_page_serialno(&og) != os.serialno)
    	{
    		/* so all streams are read. */
    		ogg_stream_reset_serialno(&os, ogg_page_serialno(&og));
    	}
    	/*Add page to the bitstream*/
    	ogg_stream_pagein(&os, &og);
    	page_granule = ogg_page_granulepos(&og);
    	/*Extract all available packets*/
    	//解析出所有的packet，将每个packet变成原始的数据
    	while (ogg_stream_packetout(&os, &op) == 1)
    	{
    		/*OggOpus streams are identified by a magic string in the initial
    		stream header.*/
    		//解析每个packet的前面8个字节，是否是OpusHead，
    		if (op.b_o_s && op.bytes >= 8 && !memcmp(op.packet, "OpusHead", 8))
    		{
    			if (has_opus_stream && has_tags_packet)
    			{
    				/*If we're seeing another BOS OpusHead now it means
    				the stream is chained without an EOS.*/
    				has_opus_stream = 0;
    				if (st)
    				{
    					opus_multistream_decoder_destroy(st);
    				}
    				st = NULL;
    				LOGD("\nWarning: stream %" I64FORMAT " ended without EOS and a new stream began.\n", (long long)os.serialno);
    			}

    			if (!has_opus_stream)
    			{
    				if (packet_count>0 && opus_serialno == os.serialno)
    				{
    					LOGD("\nError: Apparent chaining without changing serial number (%" I64FORMAT "==%" I64FORMAT ").\n",
    							(long long)opus_serialno, (long long)os.serialno);
    					quit(1);
    				}

    				opus_serialno = os.serialno;
    				//标识有opus_stream的流
    				has_opus_stream = 1;
    				has_tags_packet = 0;
    				link_out = 0;
    				//解析一开始的时候，将这个packet的数量置为0
    				packet_count = 0;
    				eos = 0;
    				total_links++;
    			}
    			else
    			{
    				LOGD("\nWarning: ignoring opus stream %" I64FORMAT "\n", (long long)os.serialno);
    			}
    		}

    		//判断每个packet的合法性，是否有opus_stream,还有判断文件的唯一标识是否相等
    		if (!has_opus_stream || os.serialno != opus_serialno)
    		{
    			break;
    		}
    		//如果第一个packet 是一个合法的  处理opus的头部
    		/*If first packet in a logical stream, process the Opus header*/
    		if (packet_count == 0)
    		{
    			int old_channels = channels;
    			//解析opus的头部，并且初始化opus的解码器，获取到opus头文件中的速录rate
    			st = process_header(&op, &decoder_rate, &mapping_family, &channels, &preskip, &gain, manual_gain, &streams, wav_format, decoder_quiet);
    			//st为空，代表初始化失败，直接退出
    			if (!st)
    			{
    				quit(1);
    			}

    			if (output && channels != old_channels)
    			{
    				LOGD("\nError: Apparent chaining changes channel count from %d to %d.\n", old_channels, channels);
    				LOGD("This is currently unhandled by opusdec.\n");
    				quit(1);
    			}

    			if (ogg_stream_packetout(&os, &op) != 0 || og.header[og.header_len - 1] == 255)
    			{
    				/*The format specifies that the initial header and tags packets are on their
    				own pages. To aid implementors in discovering that their files are wrong
    				we reject them explicitly here. In some player designs files like this would
    				fail even without an explicit test.*/
    				LOGD("Extra packets on initial header page. Invalid stream.\n");
    				quit(1);
    			}

    			/*Remember how many samples at the front we were told to skip
    			so that we can adjust the timestamp counting.*/
    			gran_offset = preskip;

    			/*Setup the memory for the dithered output*/
    			//如果没有分配内存空间，分配内存空间
    			if (!shapemem.a_buf)
    			{
    				shapemem.a_buf = calloc(channels, sizeof(float) * 4);
    				shapemem.b_buf = calloc(channels, sizeof(float) * 4);
    				//保存帧
    				shapemem.fs = decoder_rate;
    			}
    			//分配存储解压缩后的内容
    			if (!output)
    			{
    				output = malloc(sizeof(float)*MAX_FRAME_SIZE*channels);
    			}

    			if (!shapemem.a_buf || !shapemem.b_buf || !output)
    			{
    				LOGD("Memory allocation failure.\n");
    				quit(1);
    			}

    			/*Normal players should just play at 48000 or their maximum rate,
    			as described in the OggOpus spec.  But for commandline tools
    			like opusdec it can be desirable to exactly preserve the original
    			sampling rate and duration, so we have a resampler here.*/
    			if (decoder_rate != 48000 && resampler == NULL)
    			{
    				int err;
    				resampler = speex_resampler_init(channels, 48000, decoder_rate, 5, &err);
    				if (err != 0)
    				{
    					LOGD("resampler error: %s\n", speex_resampler_strerror(err));
    				}
    				speex_resampler_skip_zeros(resampler);
    			}
    		}
    		else if (packet_count == 1)//第二次写的coment信息
    		{
    			if (!decoder_quiet)
    			{
    				print_comments((char*)op.packet, op.bytes);
    			}
    			has_tags_packet = 1;
    			if (ogg_stream_packetout(&os, &op) != 0 || og.header[og.header_len - 1] == 255)
    			{
    				LOGD("Extra packets on initial tags page. Invalid stream.\n");
    				quit(1);
    			}
    		}
    		else
    		{
    			//这边才是开始解析压缩的数据
    			int ret;
    			opus_int64 maxout;
    			opus_int64 outsamp;

    			//默认丢失帧为0
    			int lost = 0;
    			if (loss_percent > 0 && 100 * ((float)rand()) / RAND_MAX < loss_percent)
    			{
    				lost = 1;
    			}

    			/*End of stream condition*/
    			//标识已经读取到了文件的结尾
    			if (op.e_o_s && os.serialno == opus_serialno)
    			{
    				eos = 1; /* don't care for anything except opus eos */
    			}

    			/*Are we simulating loss for this packet?*/
    			//标识我们是否有丢包的情况
    			if (!lost)
    			{
    				/*Decode Opus packet*/
    				//解析packet 片段的内容,输出到output里面
    				ret = opus_multistream_decode_float(st, (unsigned char*)op.packet, op.bytes, output, MAX_FRAME_SIZE, 0);
    			}
    			else
    			{
    				/*Extract the original duration.
    				Normally you wouldn't have it for a lost packet, but normally the
    				transports used on lossy channels will effectively tell you.
    				This avoids opusdec squaking when the decoded samples and
    				granpos mismatches.*/
    				opus_int32 lost_size;
    				lost_size = MAX_FRAME_SIZE;
    				if (op.bytes>0)
    				{
    					opus_int32 spp;
    					spp = opus_packet_get_nb_frames(op.packet, op.bytes);
    					if (spp>0)
    					{
    						spp *= opus_packet_get_samples_per_frame(op.packet, 48000/*decoding_rate*/);
    						if (spp > 0)
    						{
    							lost_size = spp;
    						}
    					}
    				}
    				/*Invoke packet loss concealment.*/
    				ret = opus_multistream_decode_float(st, NULL, 0, output, lost_size, 0);
    			}

    			//解压缩返回的值，如果小于0的话，说明解压缩出现了错误
    			/*If the decoder returned less than zero, we have an error.*/
    			if (ret<0)
    			{
    				LOGD("Decoding error: %s\n", opus_strerror(ret));
    				break;
    			}
    			//解压缩之后返回的值
    			decoder_frame_size = ret;
    			/*Apply header gain, if we're not using an opus library new enough to do this internally.*/
    			if (gain != 0)
    			{
    				for (i = 0; i < decoder_frame_size*channels; i++)
    				{
    					output[i] *= gain;
    				}
    			}

    			/*This handles making sure that our output duration respects
    			the final end-trim by not letting the output sample count
    			get ahead of the granpos indicated value.*/
    			maxout = ((page_granule - gran_offset)*decoder_rate / 48000) - link_out;
    			//返回解压缩之后的大小
    			outsamp = audio_write(output, channels, decoder_frame_size, resampler, &preskip, dither ? &shapemem : 0, 1, 0>maxout ? 0 : maxout, fp, destData+pageCount);
    			//将当前读取的长度，加到总共解压缩之后的大小
    			link_out += outsamp;

    			audio_size += (fp ? 4 : 2)*outsamp*channels;

    			pageCount += outsamp;
    		}
    		//在一页中的标识packet的数量，
    		packet_count++;
    	}

    	if (pageCount > 0)
    	{
    		//写的是short *,将首地址变成char * ，只是每次写俩个字节，就可以做到写short*
    		//fwrite((char *)destData, (fp ? 4 : 2)*channels, pageCount, fout);
    		LOGD("packet_count == %lld pageCount == %d\n", packet_count,pageCount);
    	}
    	//printf("packet_count == %lld pageCount == %d\n", packet_count,pageCount);

    	/*We're done, drain the resampler if we were using it.*/
    	//文件的结尾
    	if (eos && resampler)
    	{
    		float *zeros;
    		int drain;

    		zeros = (float *)calloc(100 * channels, sizeof(float));
    		if (!zeros)
    		{
    			LOGD("Memory allocation failure.\n");
    			quit(1);
    		}
    		drain = speex_resampler_get_input_latency(resampler);
    		do {
    			opus_int64 outsamp;
    			int tmp = drain;
    			if (tmp > 100)
    			{
    				tmp = 100;
    			}
    			outsamp = audio_write(zeros, channels, tmp, resampler, NULL, &shapemem, 1, ((page_granule - gran_offset)*decoder_rate / 48000) - link_out, fp,destData+pageCount);
    			link_out += outsamp;
    			audio_size += (fp ? 4 : 2)*outsamp*channels;

    			pageCount += outsamp;
    			drain -= tmp;
    		} while (drain>0);
    		free(zeros);
    		speex_resampler_destroy(resampler);
    		resampler = NULL;

    		if (pageCount > 0)
    		{
    			//写的是short *,将首地址变成char * ，只是每次写俩个字节，就可以做到写short*
    			//fwrite((char *)destData, (fp ? 4 : 2)*channels, pageCount, fout);
    			LOGD("packet_count == %lld pageCount == %d\n", packet_count,pageCount);
    		}
    		LOGD("opusDecoder success \n");
    	}
    }
    return pageCount;
}

/*
 * Class:     vide_m4399_com_newopusdemo_OpusDecoder
 * Method:    nativeInitDecoder
 * Signature: ()V
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusDecoder_nativeInitDecoder
  (JNIEnv * env, jobject obj)
{
    int ret = 0;
    opus_decoder_resert_variably();
    //对于指针的类型一般赋值为NULL，比较好理解
    output = NULL;
    //0 跟NULL '\0' 都是一样的，都可以赋值
    shapemem.a_buf = NULL;
    shapemem.b_buf = NULL;
    shapemem.mute = 960;
    shapemem.fs = 0;
    /*Output to a file or playback?*/
    //判断是否有输出文件的参数的指定
    wav_format = 1;
    /*If the output is floating point, don't dither.*/
    dither = 1;
    //初始化oy里面的内容，全部变成0
    ogg_sync_init(&oy);
    return ret;
}

/*
 * Class:     vide_m4399_com_newopusdemo_OpusDecoder
 * Method:    nativeDecodeBytes
 * Signature: ([BI[S)I
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusDecoder_nativeDecodeBytes
  (JNIEnv * env, jobject obj, jbyteArray decoder_insrc, jint length, jshortArray decoder_out)
{
    int ret = 0;
    //将java中的数组变成c对应的数组
    jshort* pcm_data_decoder = (*env)->GetShortArrayElements(env, decoder_out,0);
	jbyte* opus_data_decoder = (*env)->GetByteArrayElements(env, decoder_insrc,0);

    //解压缩，返回解压缩之后的原始的长度
    //int MyOpusDecoder(char * srcData,int nb_read,short * destData)
	ret = MyOpusDecoder(opus_data_decoder, length,pcm_data_decoder);

    //释放内存
    (*env)->ReleaseShortArrayElements(env, decoder_out, pcm_data_decoder, 0);
    (*env)->ReleaseByteArrayElements(env, decoder_insrc, opus_data_decoder, 0);
    if ((*env)->ExceptionCheck(env))
    {
    	return -1;
    }

    return ret;
}

/*
 * Class:     vide_m4399_com_newopusdemo_OpusDecoder
 * Method:    nativeReleaseDecoder
 * Signature: ()V
 */
JNIEXPORT jint JNICALL Java_com_example_administrator_opusnativebuild_OpusDecoder_nativeReleaseDecoder(JNIEnv * env, jobject obj)
{
    int ret = 0;

    LOGD("opusDecoder free start\n");

  	if (!total_links)
  	{
  		LOGD("error This doesn't look like a Opus file\n");
  	}

  	if (stream_init)
  	{
  		ogg_stream_clear(&os);
  	}

  	ogg_sync_clear(&oy);

  	if (shapemem.a_buf)
  	{
  		free(shapemem.a_buf);
  	}
  	if (shapemem.b_buf)
  	{
  		free(shapemem.b_buf);
  	}

  	if (output)
  	{
  		free(output);
  	}

  	LOGD("opusDecoder free success\n");
  	return ret;
}
```
java代码

```java
public class OpusDecoder
{
    public static final String TAG = "OpusDecoder";

    private int frequency = 8000; // Capture mono data at 8kHz

    private int channelConfiguration = AudioFormat.CHANNEL_CONFIGURATION_MONO;

    private int audioEncoding = AudioFormat.ENCODING_PCM_16BIT; // raw encoding

    private final int bufferSize = 200;

    /**
     * opus解码器的初始化,初始话成功返回0
     */
    private native int nativeInitDecoder();

    /**
     * opus流的解析.返回解压缩之后的大小
     *
     * @param in 要解析的数组
     * @param readSize 读取的大小
     * @param out 解析之后的数组
     * @return
     */
    private native int nativeDecodeBytes(byte[] in,int readSize,short[] out);

    /**
     * opus解码器的释放,释放成功返回0
     */
    private native int nativeReleaseDecoder();

    /**
     * 要decoder的opus文件路径
     */
    private String mSrcDecoderpath;

    /**
     * 具备播放的音频
     */
    private AudioTrack track;

    /**
     * 文件流
     */
    private FileInputStream mFileInputStream;

    /**
     * 每次读取的文件流
     */
    private byte[] readBuffer;

    /**
     * 代表了最后一块
     */
    private boolean mIsLastPart;

    /***
     * 每次解析返回的大小
     */
    private int mDecoderSize;

    /**
     * 已经播放读取的大小
     */
    private int mBytesRead;


    public OpusDecoder(String srcPath)
    {
        this.mSrcDecoderpath = srcPath;
        int ret = nativeInitDecoder();
        if(ret != 0)
        {
            Log.d(TAG, "opusDecoder initError");
            return;
        }
        initializeAndroidAudio(8000);
    }

    private void initializeAndroidAudio(int sampleRate)
    {
        // 可以解决部分机型的问题
        int minBufferSize = 2 * AudioTrack.getMinBufferSize(sampleRate, channelConfiguration,
                                                            audioEncoding);
        track = new AudioTrack(AudioManager.STREAM_VOICE_CALL, frequency, channelConfiguration,
                               audioEncoding, minBufferSize, AudioTrack.MODE_STREAM);
    }

    /**
     * 解析的思路是按照你写进来的方式来进行的，每一个页里面可以包含有多个包，每一个页的数据，都是有包含有一些 特殊的信息的，有文件的检验，oggs
     * 标准的信息等
     *
     * @throws Exception
     */
    public void decode() throws Exception
    {
        File cacheFile = new File(mSrcDecoderpath);
        mFileInputStream = new FileInputStream(cacheFile);
        readBuffer = new byte[bufferSize];
        short[] decoded = new short[65535];
        while (true)
        {
            int read = mFileInputStream.read(readBuffer);
            if (read != bufferSize)//如果读取到了文件的结尾的化，返回值不是bufferSize大小
            {
                mIsLastPart = true;
            }

            if ((mDecoderSize = nativeDecodeBytes(readBuffer,read, decoded)) > 0)
            {
                Log.d(TAG, "nativeDecodeBytes length "+mDecoderSize);

                mBytesRead += track.write(decoded, 0, mDecoderSize);
                track.setStereoVolume(1.0f, 1.0f);// 设置当前音量大小
                track.play();
            }
            //读取到了文件的结尾
            if(mIsLastPart)
            {
                break;
            }
        }

        track.stop();
        track.release();
        // 释放这个资源
        int ret = nativeReleaseDecoder();
        if(ret != 0)
        {
            Log.d(TAG, "opusDecoder free error");
        }
        mFileInputStream.close();
        mFileInputStream = null;
    }
}

public class Recording
{
    private static String TAG = "Recording";

    private String audioFolder = "/example"; // Folder in which the audio file
    // is written.
    private String audioFile = "voice.opus"; // Name of the audio file.
    // Status flags
    public boolean isRecording = false;
    public boolean isRecordFinished = false;

    // Fields
    public File recordedFile = null;

    // 设置音频采样率，44100是目前的标准，但是某些设备仍然支持22050，16000，11025
    // Audio config
    // 8kHz,CHANNEL_CONFIGURATION_MONO

    //设置音频的录制的声道CHANNEL_IN_STEREO为双声道，CHANNEL_CONFIGURATION_MONO为单声道
    private int channelConfiguration = AudioFormat.CHANNEL_CONFIGURATION_MONO;

    private int audioEncoding = AudioFormat.ENCODING_PCM_16BIT;
    // 音频获取源
    private int audioSource = MediaRecorder.AudioSource.MIC;

    private boolean shouldStopRecording = false;
    // 缓冲区字节大小
    private int bufferSizeInBytes = 0;

    private int frequency = 8000; // Capture mono data at 8kHz

    /**
     * 每次读取320个字节
     */
    private final int bufferSize = 320;

    /**
     * 开始录制声音
     */
    public void recordToFile()
    {
        this.shouldStopRecording = false;
        this.isRecording = true;
        // 获得缓冲区字节大小
        bufferSizeInBytes = AudioRecord.getMinBufferSize(frequency, channelConfiguration,
                                                         audioEncoding);

        // 创建AudioRecord对象
        AudioRecord audioRecord = new AudioRecord(audioSource, frequency, channelConfiguration,
                                                  audioEncoding, bufferSizeInBytes);

        audioRecord.startRecording();

        File file = initFile();

        final OpusEncoder encoder = new OpusEncoder(file.getPath());
        byte[] buffer = new byte[bufferSize];

        while (!this.shouldStopRecording)
        {
            int recordLength = audioRecord.read(buffer, 0, bufferSize);
            Log.d("Recording", recordLength+"");
            encoder.write(buffer,recordLength/2);

        }
        audioRecord.stop();
        audioRecord.release();
        encoder.close();
        this.isRecording = false;
        this.isRecordFinished = true;
    }


    /**
     * Creates the output file and folder.
     *
     * @return The output stream for the created file.
     */
    private File initFile()
    {
        File sdCard = Environment.getExternalStorageDirectory();
        File dir = new File(sdCard.getAbsolutePath() + this.audioFolder);
        dir.mkdirs();
        File audioFile = new File(dir, this.audioFile);

        if (audioFile.length() > 0)
        {
            audioFile.delete();
            audioFile = new File(dir, this.audioFile);
        }
        this.recordedFile = audioFile;
        return recordedFile;
    }

    /**
     * 获取录音的文件路径
     * @return
     */
    public String getRecordFilePath()
    {
        return recordedFile.getAbsolutePath();
    }

    /**
     * 停止录音
     */
    public void stopRecording()
    {
        this.shouldStopRecording = true;
    }
}
```
生成结果
![](/uploads/Opus交叉编译/opusSo生成成功.png)

