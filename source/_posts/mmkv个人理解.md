---
layout: pager
title: mmkv个人理解
date: 2020-09-26 19:12:37
tags: [mmkv,Android,SharedPreferences]
description:  mmkv个人理解
---

### 概述

> 记录下 这俩天关于mmkv学习记录

<!--more-->

### 简介
MMKV 是微信于 2018 年 9 月 20 日开源的一个 K-V 存储库，它与 SharedPreferences 相似，但又在更高的效率下解决了其不支持跨进程读写等弊端。主要介绍下SharedPreferences的缺点以及mmkv的应对策略


### SharedPreferences的问题
SharedPreferences每次写一次数据的时候，都会把整个内存的数据写到文件里面,下面是写文件的基本逻辑

private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
	...
	if (!backupFileExists) {
        if (!mFile.renameTo(mBackupFile)) {
            return;
        }
    } else {
        mFile.delete();
    }
	...
	FileOutputStream str = createFileOutputStream(mFile);
    ...
    XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
    writeTime = System.currentTimeMillis();
    FileUtils.sync(str);
    fsyncTime = System.currentTimeMillis();
    str.close();
	
	// Writing was successful, delete the backup file if there is one.
    mBackupFile.delete();
	...
}
SharedPreferences的容错机制是通过备份文件实现的，对于当前修改的数据，当要写到文件的时候，会将原本文件的内容重命名为back.file，保存的是当前修改之前的内容，之后将当前修改的内容，重新写到
目标文件，如果写成功了，就会把back.file移除，如果失败了，因为有back.file也可以找到之前的内容,但是这个方法每次当调用commit或者apply的时候都会执行一次，如果当前只需要修改一个值，那么也需要将整个文件的
内容重新写入，不支持跨进程通讯



### MMKV
首先简单的分析下调用的流程,首先是执行初始化操作







