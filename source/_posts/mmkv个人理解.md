---
layout: pager
title: mmkv个人理解
date: 2020-09-26 19:12:37
tags: [mmkv,Android,SharedPreferences]
description:  mmkv个人理解
---
### 简介
MMKV 是微信于 2018 年 9 月 20 日开源的一个 K-V 存储库，它与 SharedPreferences 相似，但又在更高的效率下解决了其不支持跨进程读写等弊端。主要介绍下SharedPreferences的缺点以及mmkv的应对策略,github地址为 https://github.com/Tencent/MMKV/blob/master/readme_cn.md

### SharedPreferences的问题
SharedPreferences每次写一次数据的时候，都会把整个内存的数据写到文件里面,下面是写文件的基本逻辑
```java
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
```
### MMKV
MMKV 原理
>内存准备
通过 mmap 内存映射文件，提供一段可供随时写入的内存块，App 只管往里面写数据，由操作系统负责将内存回写到文件，不必担心 crash 导致数据丢失。
数据组织
数据序列化方面我们选用 protobuf 协议，pb 在性能和空间占用上都有不错的表现。
写入优化
考虑到主要使用场景是频繁地进行写入更新，我们需要有增量更新的能力。我们考虑将增量 kv 对象序列化后，append 到内存末尾。
空间增长
使用 append 实现增量更新带来了一个新的问题，就是不断 append 的话，文件大小会增长得不可控。我们需要在性能和空间上做个折中。

具体的调用流程可以参考这篇文章
[Android比 SharedPreferences 更高效？微信 MMKV 源码解析](https://blog.n0texpecterr0r.cn/2020/01/05/%e9%87%8d%e8%af%bb-mmkv%ef%bc%9a%e5%90%8c%e6%a0%b7%e7%9a%84%e4%bb%a3%e7%a0%81%ef%bc%8c%e4%b8%8d%e5%90%8c%e7%9a%84%e6%84%9f%e6%82%9f/)

这里大致介绍下MMKV关于数据校验和多进程读写的同步的问题

### 数据的校验
首先我们要知道mmvk里面数据的校验是通过crc来做到校验的，而且每个MMKV对象类似于SharedPreferences对象里面都有俩个文件，一个是用来存储原始数据的文件，一个是用来存储校验的文件，对应的变量 MMKVPath_t m_path; MMKVPath_t m_crcPath; 相应的在本地文件的路径为，当然这是demo的路径
![结果显示](/uploads/mmkv/存储文件路径.png)

接着在构造函数中将文件的内容读取使用mmap映射到内存，这里要考虑页的完整性,假设是第一次添加数据进来
>m_file(new MemoryFile(m_path, size, (mode & MMKV_ASHMEM) ? MMFILE_TYPE_ASHMEM : MMFILE_TYPE_FILE))//原始文件
m_metaFile(new MemoryFile(m_crcPath, DEFAULT_MMAP_SIZE, m_file->m_fileType))//用于存储元数据的文件对象
m_metaInfo(new MMKVMetaInfo())//存储元数据的对象

```java
//如果当前没有加密操作
auto itr = m_dic->find(key);
//如果key之前添加过
if (itr != m_dic->end()) {
    //如果key之前加载过，那么这里的 KeyValueHolder 就为 map集合的值
    auto ret = appendDataWithKey(data, itr->second, isDataHolder);
    if (!ret.first) {
        return false;
    }
    itr->second = std::move(ret.second);
} else {
    //如果key是第一次加载的
    auto ret = appendDataWithKey(data, key, isDataHolder);
    if (!ret.first) {
        return false;
    }
    m_dic->emplace(key, std::move(ret.second));
}

假设我们是第一次添加进来


KVHolderRet_t MMKV::appendDataWithKey(const MMBuffer &data, MMKVKey_t key, bool isDataHolder) {
#ifdef MMKV_APPLE
    auto oData = [key dataUsingEncoding:NSUTF8StringEncoding];
    auto keyData = MMBuffer(oData, MMBufferNoCopy);
#else
    //首先将key的内容封装到 keyData 中
    auto keyData = MMBuffer((void *) key.data(), key.size(), MMBufferNoCopy);
#endif
    return doAppendDataWithKey(data, keyData, isDataHolder, static_cast<uint32_t>(keyData.length()));
}

KVHolderRet_t
MMKV::doAppendDataWithKey(const MMBuffer &data, const MMBuffer &keyData, bool isDataHolder, uint32_t originKeyLength) {
    auto isKeyEncoded = (originKeyLength < keyData.length());
    auto keyLength = static_cast<uint32_t>(keyData.length());
    auto valueLength = static_cast<uint32_t>(data.length());
    if (isDataHolder) {
        valueLength += pbRawVarint32Size(valueLength);
    }

    // size needed to encode the key  计算key 需要的大小
    size_t size = isKeyEncoded ? keyLength : (keyLength + pbRawVarint32Size(keyLength));

    // size needed to encode the value  再加上 value 需要的大小
    size += valueLength + pbRawVarint32Size(valueLength);

    //由于当前需要写，我们需要的是独占锁
    SCOPED_LOCK(m_exclusiveProcessLock);

    //确保内存时足够的 size为当前存储key和value总共需要的内存
    bool hasEnoughSize = ensureMemorySize(size);
    if (!hasEnoughSize || !isFileValid()) {
        return make_pair(false, KeyValueHolder());
    }
    ...
    try {
        //首先写key
        if (isKeyEncoded) {
            m_output->writeRawData(keyData);
        } else {
            m_output->writeData(keyData);
        }
        if (isDataHolder) {
            m_output->writeRawVarint32((int32_t) valueLength);
        }
        //后面再写 value的值
        m_output->writeData(data); // note: write size of data
    } catch (std::exception &e) {
        MMKVError("%s", e.what());
        return make_pair(false, KeyValueHolder());
    }
    //我们让指针跳到合适的位置，用来记录当前写进去的内容的 校验值
    auto offset = static_cast<uint32_t>(m_actualSize);
    //此时还没有更新 m_actualSize 的值，也即是当前的ptr 指向为 当前要写入数据的前面
    auto ptr = (uint8_t *) m_file->getMemory() + Fixed32Size + m_actualSize;
#ifndef MMKV_DISABLE_CRYPT
    if (m_crypter) {
        m_crypter->encrypt(ptr, ptr, size);
    }
#endif
    //更新当前实际的大小
    m_actualSize += size;
    //写校验值
    updateCRCDigest(ptr, size);

    return make_pair(true, KeyValueHolder(originKeyLength, valueLength, offset));
}

其实第一次进来的话，内存是不够的，所以要完成扩充操作
bool MMKV::ensureMemorySize(size_t newSize) {
    if (!isFileValid()) {
        MMKVWarning("[%s] file not valid", m_mmapID.c_str());
        return false;
    }
    //如果内存空间不够了，
    //使用 append 实现增量更新带来了一个新的问题，就是不断 append 的话，文件大小会增长得不可控。例如同一个 key 不断更新的话，是可能耗尽几百 M 甚至上 G 空间，
    //而事实上整个 kv 文件就这一个 key，不到 1k 空间就存得下。这明显是不可取的。我们需要在性能和空间上做个折中：以内存 pagesize 为单位申请空间，
    //在空间用尽之前都是 append 模式；当 append 到文件末尾时，进行文件重整、key 排重，尝试序列化保存排重结果；排重后空间还是不够用的话，将文件扩大一倍，直到空间足够。
    if (newSize >= m_output->spaceLeft() || (m_crypter ? m_dicCrypt->empty() : m_dic->empty())) {
        // try a full rewrite to make space
        auto fileSize = m_file->getFileSize();
        //将当前map的内容，转成整个  pair<MMBuffer, size_t> 形式返回
        auto preparedData = m_crypter ? prepareEncode(*m_dicCrypt) : prepareEncode(*m_dic);
        //大小
        auto sizeOfDic = preparedData.second;
        ...
        //内存重整操作
        return doFullWriteBack(move(preparedData), nullptr);
    }
    return true;
}

size_t CodedOutputData::spaceLeft() {
    if (m_size <= m_position) {
        return 0;
    }
    return m_size - m_position;
}
第一次进来的时候，由于构造函数中完成了文件的映射，而且使用mmap完成页的扩充，所以这里是肯定不满足的，但是会满足   m_dic->empty(), 其中  mmkv::MMKVMap *m_dic;的数据类型为
using MMKVMap = std::unordered_map<std::string, mmkv::KeyValueHolder>; 其实就是对应文件的内容在内存中的隐射，第一次进来那么肯定会为空，

由于我们没有使用加密，所以会执行 prepareEncode(*m_dic);
/**
 * 将 map的内容，转成MMBuffer 的形式，返回整个内存的结果
 * @param dic
 * @return
 */
static pair<MMBuffer, size_t> prepareEncode(const MMKVMap &dic) {
    // make some room for placeholder
    size_t totalSize = ItemSizeHolderSize;
    for (auto &itr : dic) {
        auto &kvHolder = itr.second;
        totalSize += kvHolder.computedKVSize + kvHolder.valueSize;
    }
    return make_pair(MMBuffer(), totalSize);
}
constexpr uint32_t ItemSizeHolderSize = 4;
这里要注意ItemSizeHolderSize 这个变量，这里可以看出来，就算没有任何内容的时候，也会占用4个字节，其实这四个字节是很有用的，实际存储的是当前写指针的位置，后面可以用来多线程的校验
这里先了解下 ，接着执行 doFullWriteBack 函数


//重整
bool MMKV::doFullWriteBack(pair<MMBuffer, size_t> preparedData, AESCrypt *newCrypter) {
    auto ptr = (uint8_t *) m_file->getMemory();
    auto totalSize = preparedData.second;//第一次进来也为 4
    ...
    //接着创建一个CodedOutputData 对象，这个对象主要用来完成数据的写
    delete m_output;
    m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
    ...
    memmoveDictionary(*m_dic, m_output, ptr, encrypter, totalSize);
    m_actualSize = totalSize;
    ...
    recaculateCRCDigestWithIV(nullptr);
    m_hasFullWriteback = true;
    // make sure lastConfirmedMetaInfo is saved
    sync(MMKV_SYNC);
    return true;
}

接着执行  m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
CodedOutputData::CodedOutputData(void *ptr, size_t len) : m_ptr((uint8_t *) ptr), m_size(len), m_position(0) {
    MMKV_ASSERT(m_ptr);
}

这个对象主要是用来完成数据的写入操作的，而且这里的 ptr + Fixed32Size，偏移指针，看看Fixed32Size 是什么
constexpr uint32_t Fixed32Size = pbFixed32Size();
constexpr uint32_t pbFixed32Size() {
    return LittleEdian32Size;
}
constexpr uint32_t LittleEdian32Size = 4;

其实就是偏移了四个字节，前面说过，前面的四个字节是用来存储当前真正写了多少内容的，所以这里要预留出，防止覆盖,那么当前文件真正能写的大小就为  m_file->getFileSize() - Fixed32Size
接着执行  memmoveDictionary(*m_dic, m_output, ptr, encrypter, totalSize);


static void
memmoveDictionary(MMKVMap &dic, CodedOutputData *output, uint8_t *ptr, AESCrypt *encrypter, size_t totalSize) {
    //获取当前写的位置
    auto originOutputPtr = output->curWritePointer();
    // make space to hold the fake size of dictionary's serialization result  
    auto writePtr = originOutputPtr + ItemSizeHolderSize;
    ...
    // hold the fake size of dictionary's serialization result  持有字典序列化结果的假大小
    output->writeRawVarint32(ItemSizeHolder);
    auto writtenSize = static_cast<size_t>(writePtr - originOutputPtr);
#ifndef MMKV_DISABLE_CRYPT
    if (encrypter) {
        encrypter->encrypt(originOutputPtr, originOutputPtr, writtenSize);
    }
#endif
    assert(writtenSize == totalSize);
    output->seek(writtenSize - ItemSizeHolderSize);
}

这里又预留了4个字节，
auto writePtr = originOutputPtr + ItemSizeHolderSize;
接着执行  ,其中 constexpr uint32_t ItemSizeHolder = 0x00ffffff;  constexpr 改善之一就是生成常量表达式，允许程序利用编译时的计算能力
output->writeRawVarint32(ItemSizeHolder); 所以这里写了一个常亮进去,有点不懂有啥作用,同时更新  output->seek(writtenSize - ItemSizeHolderSize); 

接着回到了执行  recaculateCRCDigestWithIV(nullptr);

/**
 * 重整的调用，
 * @param iv
 */
void MMKV::recaculateCRCDigestWithIV(const void *iv) {
    auto ptr = (const uint8_t *) m_file->getMemory();
    if (ptr) {
        m_crcDigest = 0;
        m_crcDigest = (uint32_t) CRC32(0, ptr + Fixed32Size, (uint32_t) m_actualSize);
        writeActualSize(m_actualSize, m_crcDigest, iv, IncreaseSequence);
    }
}

这里要注意 ptr 是 不带偏移的，文件映射的内存首地址,接着使用 CRC32 得到当前的校验码，前面说过 文件的前面四个字节是用来当前写了多少内容的，所以这里要ptr + Fixed32Size 跳过
m_actualSize 就为当前写的数据的长度，像刚才，我们写了一个 ItemSizeHolder 到文件中，而且前面   m_actualSize = totalSize; 赋值了4，也即是当前写的长度
之后执行 writeActualSize(m_actualSize, m_crcDigest, iv, IncreaseSequence);

/**
 * 写当前的实际的大小，以及校验值
 * @param size
 * @param crcDigest
 * @param iv
 * @param increaseSequence
 * @return
 */
bool MMKV::writeActualSize(size_t size, uint32_t crcDigest, const void *iv, bool increaseSequence) {

    // backward compatibility 往 m_file 中保存当前真正写的长度 也就是  m_actualSize 存储在开头四个字节中
    oldStyleWriteActualSize(size);

    if (!m_metaFile->isFileValid()) {
        return false;
    }

    bool needsFullWrite = false;
    m_actualSize = size;
    m_metaInfo->m_actualSize = static_cast<uint32_t>(size);
    m_crcDigest = crcDigest;
    m_metaInfo->m_crcDigest = crcDigest;

    if (m_metaInfo->m_version < MMKVVersionSequence) {
        m_metaInfo->m_version = MMKVVersionSequence;
        needsFullWrite = true;
    }
#ifndef MMKV_DISABLE_CRYPT
    if (unlikely(iv)) {
        memcpy(m_metaInfo->m_vector, iv, sizeof(m_metaInfo->m_vector));
        if (m_metaInfo->m_version < MMKVVersionRandomIV) {
            m_metaInfo->m_version = MMKVVersionRandomIV;
        }
        needsFullWrite = true;
    }
#endif
    // 重置的内容，要写到文件中,每次发生内存重整的时候，就将序列号递增，将这个序列号也放到mmap内存中，每个进程内部也缓存一份，只需要对比序列号是否一致，就能够知道其他
    //进程是否触发了内存重整
    if (unlikely(increaseSequence)) {
        m_metaInfo->m_sequence++;
        m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize = static_cast<uint32_t>(size);
        m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest = crcDigest;
        if (m_metaInfo->m_version < MMKVVersionActualSize) {
            m_metaInfo->m_version = MMKVVersionActualSize;
        }
        needsFullWrite = true;
    }
#ifdef MMKV_IOS
    auto ret = guardForBackgroundWriting(m_metaFile->getMemory(), sizeof(MMKVMetaInfo));
    if (!ret.first) {
        return false;
    }
#endif
    if (unlikely(needsFullWrite)) {
        m_metaInfo->write(m_metaFile->getMemory());
    } else {
        m_metaInfo->writeCRCAndActualSizeOnly(m_metaFile->getMemory());
    }
    return true;
}

对于第一次操作来说，我们外面传递的参数为 writeActualSize(m_actualSize, m_crcDigest, iv, IncreaseSequence);
这个函数挺重要的，首先看 oldStyleWriteActualSize(size);

/**
 * m_file 中前面的四个字节 是用来存储当前真正写入的长度
 * @param actualSize
 */
void MMKV::oldStyleWriteActualSize(size_t actualSize) {
    MMKV_ASSERT(m_file->getMemory());
    m_actualSize = actualSize;
    memcpy(m_file->getMemory(), &actualSize, Fixed32Size);
}

所以验证了，我们前面说的，为什么要保留前面的四个字节，这里就是为了存储 m_actualSize 的大小的，接着执行

bool needsFullWrite = false;
m_actualSize = size;
m_metaInfo->m_actualSize = static_cast<uint32_t>(size);
m_crcDigest = crcDigest;
m_metaInfo->m_crcDigest = crcDigest;

像这些变量的赋值，就是就是为将当前的校验值等参数写到另一个文件 ，用于校验数据正确性的文件,接着执行 m_metaInfo->write(m_metaFile->getMemory());
void write(void *ptr) const {
    MMKV_ASSERT(ptr);
    memcpy(ptr, this, sizeof(MMKVMetaInfo));
}
所以就将当前的值保存到了内存中

接着返回 doFullWriteBack函数  执行到 sync(MMKV_SYNC); 函数
void MMKV::sync(SyncFlag flag) {
    SCOPED_LOCK(m_lock);
    if (m_needLoadFromFile || !isFileValid()) {
        return;
    }
    SCOPED_LOCK(m_exclusiveProcessLock);

    m_file->msync(flag);
    m_metaFile->msync(flag);
}
bool MemoryFile::msync(SyncFlag syncFlag) {
    if (m_ptr) {
        auto ret = ::msync(m_ptr, m_size, syncFlag ? MMKV_SYNC : MMKV_ASYNC);
        if (ret == 0) {
            return true;
        }
        MMKVError("fail to msync [%s], %s", m_name.c_str(), strerror(errno));
    }
    return false;
}
其实做的就是将内容同步到文件了，使用了msync
```
所以这里总结一下，对于第一次做的操作,往原始文件里面写了一个ItemSizeHolder 的内容，占四个字节，同时使用了crc校验当前写入的内容，保存到了校验文件中，接着回到doAppendDataWithKey

```java
KVHolderRet_t
MMKV::doAppendDataWithKey(const MMBuffer &data, const MMBuffer &keyData, bool isDataHolder, uint32_t originKeyLength) {
    auto isKeyEncoded = (originKeyLength < keyData.length());
    auto keyLength = static_cast<uint32_t>(keyData.length());
    auto valueLength = static_cast<uint32_t>(data.length());
    if (isDataHolder) {
        valueLength += pbRawVarint32Size(valueLength);
    }

    // size needed to encode the key  计算key 需要的大小
    size_t size = isKeyEncoded ? keyLength : (keyLength + pbRawVarint32Size(keyLength));

    // size needed to encode the value  再加上 value 需要的大小
    size += valueLength + pbRawVarint32Size(valueLength);

    //由于当前需要写，我们需要的是独占锁
    SCOPED_LOCK(m_exclusiveProcessLock);

    //确保内存时足够的 size为当前存储key和value总共需要的内存
    bool hasEnoughSize = ensureMemorySize(size);
    if (!hasEnoughSize || !isFileValid()) {
        return make_pair(false, KeyValueHolder());
    }
    ...
    try {
        //首先写key
        if (isKeyEncoded) {
            m_output->writeRawData(keyData);
        } else {
            m_output->writeData(keyData);
        }
        if (isDataHolder) {
            m_output->writeRawVarint32((int32_t) valueLength);
        }
        //后面再写 value的值
        m_output->writeData(data); // note: write size of data
    } catch (std::exception &e) {
        MMKVError("%s", e.what());
        return make_pair(false, KeyValueHolder());
    }
    //我们让指针跳到合适的位置，用来记录当前写进去的内容的 校验值
    auto offset = static_cast<uint32_t>(m_actualSize);
    //此时还没有更新 m_actualSize 的值，也即是当前的ptr 指向为 当前要写入数据的前面
    auto ptr = (uint8_t *) m_file->getMemory() + Fixed32Size + m_actualSize;
#ifndef MMKV_DISABLE_CRYPT
    if (m_crypter) {
        m_crypter->encrypt(ptr, ptr, size);
    }
#endif
    //更新当前实际的大小
    m_actualSize += size;
    //写校验值
    updateCRCDigest(ptr, size);

    return make_pair(true, KeyValueHolder(originKeyLength, valueLength, offset));
}
由于当前是没有加密的操作，执行 m_output->writeData(keyData);
/**
 * 写数据
 * @param value
 */
void CodedOutputData::writeData(const MMBuffer &value) {
    //写当前value1的长度
    this->writeRawVarint32((int32_t) value.length());
    //写value
    this->writeRawData(value);
}
/**
 * 将内容写到内存中， m_position 代表下一次要写入的位置
 * @param data
 */
void CodedOutputData::writeRawData(const MMBuffer &data) {
    size_t numberOfBytes = data.length();
    if (m_position + numberOfBytes > m_size) {
        auto msg = "m_position: " + to_string(m_position) + ", numberOfBytes: " + to_string(numberOfBytes) +
                   ", m_size: " + to_string(m_size);
        throw out_of_range(msg);
    }
    memcpy(m_ptr + m_position, data.getPtr(), numberOfBytes);
    m_position += numberOfBytes;
}

接着写 value，上面是写key
m_output->writeData(data); // note: write size of data
所以对于上面的写操作来说，首先写了key，后面再写value，而且写的时候，首先写了当前要写入数据的长度，然后写整个内容,接着执行 

//我们让指针跳到合适的位置，用来记录当前写进去的内容的 校验值
auto offset = static_cast<uint32_t>(m_actualSize);
//此时还没有更新 m_actualSize 的值，也即是当前的ptr 指向为 当前要写入数据的前面
auto ptr = (uint8_t *) m_file->getMemory() + Fixed32Size + m_actualSize;
	
//更新当前实际的大小
m_actualSize += size;
//写校验值，注意这里的ptr为当前写进数据开始的位置,接着执行 updateCRCDigest(ptr, size);

/**
 * 更新校验值
 * @param ptr
 * @param length
 */
void MMKV::updateCRCDigest(const uint8_t *ptr, size_t length) {
    if (ptr == nullptr) {
        return;
    }
    //计算 从 ptr开始 length长度的校验的值
    m_crcDigest = (uint32_t) CRC32(m_crcDigest, ptr, (uint32_t) length);

    //写校验值
    writeActualSize(m_actualSize, m_crcDigest, nullptr, KeepSequence);
}
首先计算当前写入数据的的校验值 用 m_crcDigest来记录，之后执行 writeActualSize()操作,这里又回到了上面分析的过程，这里只是将当前写入数据的长度，
保存到前面的四个字节中，之后，保存当前的crc校验值,保存到校验文件中

bool MMKV::setDataForKey(MMBuffer &&data, MMKVKey_t key, bool isDataHolder){
    ...
    if (!ret.first) {
       return false;
    }
    m_dic->emplace(key, std::move(ret.second));
}
最终将当前写入的内容保存到内存中，使用m_dic 来存储,那标识写操作完成
```
回到前面构造 mmkv对象的时候
```java
MMKV::MMKV(const string &mmapID, int size, MMKVMode mode, string *cryptKey, string *rootPath){
    ...
    loadFromFile();
}
执行 loadFromFile

//加载内容
void MMKV::loadFromFile() {
    //如果元数据文件存在，加载元数据内容
    if (m_metaFile->isFileValid()) {
        m_metaInfo->read(m_metaFile->getMemory());
    }
    ...
    //加载原始文件
    if (!m_file->isFileValid()) {
        m_file->reloadFromFile();
    }
    if (!m_file->isFileValid()) {
        MMKVError("file [%s] not valid", m_path.c_str());
    } else {
        // error checking  加载成功，检查错误
        bool loadFromFile = false, needFullWriteback = false;

        //检查数据的有效性
        checkDataValid(loadFromFile, needFullWriteback);
		
        auto ptr = (uint8_t *) m_file->getMemory();
        // loading
        if (loadFromFile && m_actualSize > 0) {
            MMKVInfo("loading [%s] with crc %u sequence %u version %u", m_mmapID.c_str(), m_metaInfo->m_crcDigest,
                     m_metaInfo->m_sequence, m_metaInfo->m_version);

            //将内容转成  MMBuffer 对象
            MMBuffer inputBuffer(ptr + Fixed32Size, m_actualSize, MMBufferNoCopy);
            if (m_crypter) {
                clearDictionary(m_dicCrypt);
            } else {
                clearDictionary(m_dic);
            }
            if (needFullWriteback) {
#ifndef MMKV_DISABLE_CRYPT
                if (m_crypter) {
                    MiniPBCoder::greedyDecodeMap(*m_dicCrypt, inputBuffer, m_crypter);
                } else
#endif
                {
                    MiniPBCoder::greedyDecodeMap(*m_dic, inputBuffer);
                }
            } else {
#ifndef MMKV_DISABLE_CRYPT
                if (m_crypter) {
                    MiniPBCoder::decodeMap(*m_dicCrypt, inputBuffer, m_crypter);
                } else
#endif
                {
                    MiniPBCoder::decodeMap(*m_dic, inputBuffer);
                }
            }
            m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
            m_output->seek(m_actualSize);
            if (needFullWriteback) {
                fullWriteback();
            }
        } else {
            // file not valid or empty, discard everything 加锁，这里加的是独占锁
            SCOPED_LOCK(m_exclusiveProcessLock);

            m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
            if (m_actualSize > 0) {
                writeActualSize(0, 0, nullptr, IncreaseSequence);
                sync(MMKV_SYNC);
            } else {
                writeActualSize(0, 0, nullptr, KeepSequence);
            }
        }
        auto count = m_crypter ? m_dicCrypt->size() : m_dic->size();
        MMKVInfo("loaded [%s] with %zu key-values", m_mmapID.c_str(), count);
    }

    m_needLoadFromFile = false;
}
首先  执行   checkDataValid(loadFromFile, needFullWriteback); 完成数据的校验
//检查文件的有效性
void MMKV::checkDataValid(bool &loadFromFile, bool &needFullWriteback) {
    // try auto recover from last confirmed location  尝试 自动恢复 最后确认的位置
    auto fileSize = m_file->getFileSize();

    //lambda表达式 用来寻找最后正确的位置
    auto checkLastConfirmedInfo = [&] {
        if (m_metaInfo->m_version >= MMKVVersionActualSize) {
            // downgrade & upgrade support
            uint32_t oldStyleActualSize = 0;
            memcpy(&oldStyleActualSize, m_file->getMemory(), Fixed32Size);

            if (oldStyleActualSize != m_actualSize) {
                MMKVWarning("oldStyleActualSize %u not equal to meta actual size %lu", oldStyleActualSize,m_actualSize);
                if (oldStyleActualSize < fileSize && (oldStyleActualSize + Fixed32Size) <= fileSize) {
                    if (checkFileCRCValid(oldStyleActualSize, m_metaInfo->m_crcDigest)) {
                        MMKVInfo("looks like [%s] been downgrade & upgrade again", m_mmapID.c_str());
                        loadFromFile = true;
                        writeActualSize(oldStyleActualSize, m_metaInfo->m_crcDigest, nullptr, KeepSequence);
                        return;
                    }
                } else {
                    MMKVWarning("oldStyleActualSize %u greater than file size %lu", oldStyleActualSize, fileSize);
                }
            }

            //如果上面没有检验成功，我们从 m_metaInfo 中拿到上一次校验的时候，当时实际的大小
            auto lastActualSize = m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize;
            if (lastActualSize < fileSize && (lastActualSize + Fixed32Size) <= fileSize) {
                //我们拿到最后一次校验的值
                auto lastCRCDigest = m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest;
                //如果上一次校验值之前的内容都可以校验通过的话，那么直接从这里恢复
                if (checkFileCRCValid(lastActualSize, lastCRCDigest)) {
                    loadFromFile = true;
                    //同时写当前实际的大小
                    writeActualSize(lastActualSize, lastCRCDigest, nullptr, KeepSequence);
                } else {
                    MMKVError("check [%s] error: lastActualSize %u, lastActualCRC %u", m_mmapID.c_str(), lastActualSize,lastCRCDigest);
                }
            } else {
                MMKVError("check [%s] error: lastActualSize %u, file size is %u", m_mmapID.c_str(), lastActualSize,fileSize);
            }
        }
    };

    //获取到实际文件的大小
    m_actualSize = readActualSize();

    //如果实际的文件大小小于当前的总的文件大小
    if (m_actualSize < fileSize && (m_actualSize + Fixed32Size) <= fileSize) {
        //首先校验总的文件内容是否正确
        if (checkFileCRCValid(m_actualSize, m_metaInfo->m_crcDigest)) {
            loadFromFile = true;
        } else {
            //如果总的文件内容不正确，那么就要检验上一次确认的正确位置
            checkLastConfirmedInfo();

            if (!loadFromFile) {
                auto strategic = onMMKVCRCCheckFail(m_mmapID);
                if (strategic == OnErrorRecover) {
                    loadFromFile = true;
                    needFullWriteback = true;
                }
                MMKVInfo("recover strategic for [%s] is %d", m_mmapID.c_str(), strategic);
            }
        }
    } else {
        //校验错误
        MMKVError("check [%s] error: %zu size in total, file size is %zu", m_mmapID.c_str(), m_actualSize, fileSize);

        //我们要找到最后确认的位置
        checkLastConfirmedInfo();

        if (!loadFromFile) {
            auto strategic = onMMKVFileLengthError(m_mmapID);
            if (strategic == OnErrorRecover) {
                // make sure we don't over read the file
                m_actualSize = fileSize - Fixed32Size;
                loadFromFile = true;
                needFullWriteback = true;
            }
            MMKVInfo("recover strategic for [%s] is %d", m_mmapID.c_str(), strategic);
        }
    }
}

// assuming m_file is valid  检测这块数据是否完整的
bool MMKV::checkFileCRCValid(size_t actualSize, uint32_t crcDigest) {
    auto ptr = (uint8_t *) m_file->getMemory();
    if (ptr) {
        m_crcDigest = (uint32_t) CRC32(0, (const uint8_t *) ptr + Fixed32Size, (uint32_t) actualSize);
        if (m_crcDigest == crcDigest) {
            return true;
        }
        MMKVError("check crc [%s] fail, crc32:%u, m_crcDigest:%u", m_mmapID.c_str(), crcDigest, m_crcDigest);
    }
    return false;
}

首先执行 readActualSize(); 获取到实际的大小,也即是读取前面的四个字节
//获取到实际文件的大小
size_t MMKV::readActualSize() {
    MMKV_ASSERT(m_file->getMemory());
    MMKV_ASSERT(m_metaFile->isFileValid());

    //获取到实际文件的大小, m_file中前面的四个字节就为  m_actualSize 的值
    uint32_t actualSize = 0;
    memcpy(&actualSize, m_file->getMemory(), Fixed32Size);

    if (m_metaInfo->m_version >= MMKVVersionActualSize) {
        if (m_metaInfo->m_actualSize != actualSize) {
            MMKVWarning("[%s] actual size %u, meta actual size %u", m_mmapID.c_str(), actualSize,m_metaInfo->m_actualSize);
        }
        return m_metaInfo->m_actualSize;
    } else {
        return actualSize;
    }
}
接着执行 checkFileCRCValid校验 crc数据是否完整,如果对不上，执行  checkLastConfirmedInfo();这是个lambda函数，如果对不上，会从 m_lastConfirmedMetaInfo 值中获取到,先看看这个值
是怎么样来赋值的，其实这个只有在
//重整
bool MMKV::doFullWriteBack(pair<MMBuffer, size_t> preparedData, AESCrypt *newCrypter) {
    ...
    recaculateCRCDigestWithIV(nullptr);
    ...
}
/**
 * 重整的调用，
 * @param iv
 */
void MMKV::recaculateCRCDigestWithIV(const void *iv) {
    auto ptr = (const uint8_t *) m_file->getMemory();
    if (ptr) {
        m_crcDigest = 0;
        m_crcDigest = (uint32_t) CRC32(0, ptr + Fixed32Size, (uint32_t) m_actualSize);
        writeActualSize(m_actualSize, m_crcDigest, iv, IncreaseSequence);
    }
}
bool MMKV::writeActualSize(size_t size, uint32_t crcDigest, const void *iv, bool increaseSequence) {
    ...
    // 重置的内容，要写到文件中,每次发生内存重整的时候，就将序列号递增，将这个序列号也放到mmap内存中，每个进程内部也缓存一份，只需要对比序列号是否一致，就能够知道其他
    //进程是否触发了内存重整
    if (unlikely(increaseSequence)) {
        m_metaInfo->m_sequence++;
        m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize = static_cast<uint32_t>(size);
        m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest = crcDigest;
        if (m_metaInfo->m_version < MMKVVersionActualSize) {
            m_metaInfo->m_version = MMKVVersionActualSize;
        }
        needsFullWrite = true;
    }
    ...
    return true;
}
先来看看这个结构体

struct MMKVMetaInfo {
    uint32_t m_crcDigest = 0;
    uint32_t m_version = MMKVVersionSequence;
    //内存重整的感知，这里使用一个单调递增的序列号，每次发生内存重整，就将序列号递增，将这个序列号也放到mmap内存中，每个进程内部也缓存一份，只需要对比
    //序列号是否一致，就能够知道其他进程是否触发了内存重整
    uint32_t m_sequence = 0; // full write-back count
    uint8_t m_vector[AES_KEY_LEN] = {};
    //内存感知的增长，我们可以通过查询文件的大小来获得，无需要在 mmap内存另外存放，写指针的同步，我们可以在每个进程内部缓存自己的写指针，然后在写入键值的同时
    //还要把最新的写指针位置也写到mmap内存中，这样每个进程只需要对比一下缓存的指针与mmap内存的写指针，如果不一样，说明其他进程进行了写操作，事实
    uint32_t m_actualSize = 0;

    // confirmed info: it's been synced to file
    struct {
        uint32_t lastActualSize = 0;
        uint32_t lastCRCDigest = 0;
        uint32_t _reserved[16] = {};
    } m_lastConfirmedMetaInfo;
    ...
}
也即是只有在内存重整的时候，才会给这个变量赋值，或者实际一点来说，只有进入了bool MMKV::ensureMemorySize(size_t newSize) 函数，满足了扩容的操作才会给这个变量赋值，
m_sequence 这个变量可以用来做同步操作，当俩个进程对应的这个值对不上的时候，代表有一方有发生内存重整，至于为什么需要内存重整是因为mmkv考虑到性能，对于添加，修改，删除都是直接
通过append的方式来进行的，所以当达到一定大小的时候，执行重整，保存最新key对应的value,所以当前要清除数据，重新的加载数据，对于第一次添加来说，也会进入里面
因为当时满足的是m_dict为空
```
对于数据的检验来说，是通过比较当前写入数据的实际的值使用crc算出检验码跟存储检验数据文件做对比，首先对比的是检验文件的 m_crcDigest，如果检验不通过，取lastCRCDigest，如果还是不通过,则回调通知校验失败

### MMKV的进程锁
首先对于进程锁的设计，可以参考官网的原理介绍，已经很详细了
[MMKV for Android 多进程设计与实现](https://github.com/Tencent/MMKV/wiki/android_ipc)

文件锁的基本认识
![结果显示](/uploads/mmkv/文件锁.png)
可以看出当只有双方都是共享方式的时候，才不会阻塞，对于其他的模式都是阻塞的，通过mmkv关于多进程锁的介绍可以知道，文件锁是不支持重入，也不支持读写锁升级/降级，接下来介绍下怎么实现这些特性,首先看下基本的数据结构

```java
首先看下mmkv对象中持有跟锁有关的对象
//锁对象,这个代表的是当前进程的锁对象，因为一个mmkv对应的是一个java的一个特定name对应的 SharedPreferences对象，所以一个进程里面，多个线程也是需要同步的，那么就可以使用这个锁
mmkv::ThreadLock *m_lock;
//文件锁对象，进程间锁对象
mmkv::FileLock *m_fileLock;
//共享文件锁封装对象 ,进程间锁对象
mmkv::InterProcessLock *m_sharedProcessLock;
//独立文件锁封装对象,  进程间锁对象
mmkv::InterProcessLock *m_exclusiveProcessLock;

首先看下 ThreadLock的定义
ThreadLock::ThreadLock() {
    //我们用pthread_ mutexattr_init函数对pthread_mutexattr结构进行初始化，用pthread_mutexattr_destroy函数对该结构进行回收。
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    //设置属性为互斥锁
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    //初始化这个锁
    pthread_mutex_init(&m_lock, &attr);
    pthread_mutexattr_destroy(&attr);
}

ThreadLock::~ThreadLock() {
    //解锁
    pthread_mutex_destroy(&m_lock);
}

void ThreadLock::initialize() {
    return;
}

void ThreadLock::lock() {
    //加锁
    auto ret = pthread_mutex_lock(&m_lock);
    if (ret != 0) {
        MMKVError("fail to lock %p, ret=%d, errno=%s", &m_lock, ret, strerror(errno));
    }
}

void ThreadLock::unlock() {
    //手动的解锁
    auto ret = pthread_mutex_unlock(&m_lock);
    if (ret != 0) {
        MMKVError("fail to unlock %p, ret=%d, errno=%s", &m_lock, ret, strerror(errno));
    }
}

//初始化，只执行一次
void ThreadLock::ThreadOnce(ThreadOnceToken_t *onceToken, void (*callback)()) {
    pthread_once(onceToken, callback);
}
其实就是使用了面向对象的方式封装了 pthread_mutex 对应的锁操作函数,还有一个ScopedLock 对象的封装

namespace mmkv {
template <typename T>
class ScopedLock {
    T *m_lock;

    void lock() {
        if (m_lock) {
            m_lock->lock();
        }
    }

    void unlock() {
        if (m_lock) {
            m_lock->unlock();
        }
    }

public:
    //构造函数中就完成加锁， explicit 是防止隐士调用
    explicit ScopedLock(T *oLock) : m_lock(oLock) {
        MMKV_ASSERT(m_lock);
        lock();
    }

    //虚构函数完成解锁
    ~ScopedLock() {
        unlock();
        m_lock = nullptr;
    }

    // just forbid it for possibly misuse
    explicit ScopedLock(const ScopedLock<T> &other) = delete;
    ScopedLock &operator=(const ScopedLock<T> &other) = delete;
};

} // namespace mmkv

#include <type_traits>

#define SCOPED_LOCK(lock) _SCOPEDLOCK(lock, __COUNTER__)
#define _SCOPEDLOCK(lock, counter) __SCOPEDLOCK(lock, counter)
//std::remove_pointer 去掉指针,下面是定义一个对象
#define __SCOPEDLOCK(lock, counter)                                                                                    \
    mmkv::ScopedLock<std::remove_pointer<decltype(lock)>::type> __scopedLock##counter(lock)

其实也是采用面向对象的方式，利用C++的构造函数，虚构函数的特性，做到自动的加锁，解锁操作，而真正的加锁，解锁是通过 传递进来的lock 来实现的，有点类似 装饰器模式

而对于 
//共享文件锁封装对象 ,进程间锁对象
mmkv::InterProcessLock *m_sharedProcessLock;
//独立文件锁封装对象,  进程间锁对象
mmkv::InterProcessLock *m_exclusiveProcessLock;

class InterProcessLock {
    FileLock *m_fileLock;
    LockType m_lockType;

public:
    InterProcessLock(FileLock *fileLock, LockType lockType)
        : m_fileLock(fileLock), m_lockType(lockType), m_enable(true) {
        MMKV_ASSERT(m_fileLock);
    }

    bool m_enable;

    //加锁操作
    void lock() {
        if (m_enable) {
            m_fileLock->lock(m_lockType);
        }
    }

    //尝试加锁
    bool try_lock() {
        if (m_enable) {
            return m_fileLock->try_lock(m_lockType);
        }
        return false;
    }

    //解锁操作
    void unlock() {
        if (m_enable) {
            m_fileLock->unlock(m_lockType);
        }
    }
};

内部也是通过持有真正操作的锁对象来完成的FileLock，那么当 这样调用的时候
//由于当前需要写文件，所以我们需要的锁问独占锁
SCOPED_LOCK(m_exclusiveProcessLock); 
就会在构造函数中调用 m_exclusiveProcessLock 中的lock函数，也就是  InterProcessLock 中的lock函数，进而执行到 m_fileLock->lock(m_lockType);函数，而真正的加锁操作是由FileLock函数完成的

enum LockType {
    SharedLockType,//共享锁
    ExclusiveLockType,//互斥锁
};

//由于创建于共享内存的pthread_mutex 是可以用作进程锁的，然后Android版的 pthread_mutex并不保证robust,也就是对 pthread_mutex
//加了锁的进程被kill 掉，系统不会进行清理操作，这个锁会一直存在，那么其他等锁的进程就会饿死，其他的IPC组件，
//MMKV 采用了文件锁，优点是天然的 robust，缺点是不支持递归加锁，也不支持读写锁升级/降级 需要自己实现
class FileLock {
    MMKVFileHandle_t m_fd;
    //共享加锁的次数
    size_t m_sharedLockCount;
    //独占加锁的次数
    size_t m_exclusiveLockCount;

    bool doLock(LockType lockType, bool wait);
    bool platformLock(LockType lockType, bool wait, bool unLockFirstIfNeeded);
    bool platformUnLock(bool unLockFirstIfNeeded);

    ...

    bool lock(LockType lockType);

    bool try_lock(LockType lockType);

    bool unlock(LockType lockType);

    // just forbid it for possibly misuse
    explicit FileLock(const FileLock &other) = delete;
    FileLock &operator=(const FileLock &other) = delete;
};
```
内部也就是使用  flock文件锁实现锁升级，重入锁的关键实现，这里要注意，对于共享锁和排它锁都是使用同一个FileLock对象
>m_fileLock(new FileLock(m_metaFile->getFd(), (mode & MMKV_ASHMEM)))//创建一个文件锁对象,这是当前进程的锁对象，主要用来同步当前进程内部线程安全的
m_sharedProcessLock(new InterProcessLock(m_fileLock, SharedLockType)) //进程间的锁
m_exclusiveProcessLock(new InterProcessLock(m_fileLock, ExclusiveLockType)) //进程间的锁
	
我们先看加锁的操作
```java
//加锁操作
bool FileLock::doLock(LockType lockType, bool wait) {
    if (!isFileLockValid()) {
        return false;
    }
    bool unLockFirstIfNeeded = false;

    //如果当前的类型是共享的话,多个进程可同时对同一个文件作共享锁定
    if (lockType == SharedLockType) {
        // don't want shared-lock to break any existing locks  当前锁的模式是共享模式，并且m_sharedLockCount 大于0，代表当前已经有了共享锁，所以直接m_sharedLockCount+1，直接返回
        if (m_sharedLockCount > 0 || m_exclusiveLockCount > 0) {
            m_sharedLockCount++;
            return true;
        }
    } else {
        // don't want exclusive-lock to break existing exclusive-locks 独占锁
        //自己实现递归锁，采用引用计数的方式，防止重复加锁，因为对于文件锁来说，不支持递归锁，也不支持锁的升级和降级
        if (m_exclusiveLockCount > 0) {
            m_exclusiveLockCount++;
            return true;
        }

        // prevent deadlock  如果当前独占锁的索引为0，但是共享锁的索引大于0,但是共享锁的索引大于0,这里要尝试的获取写锁，如果获取不成功，我们要放弃读锁
        if (m_sharedLockCount > 0) {
            unLockFirstIfNeeded = true;
        }
    }

    //根据平台执行加锁操作
    auto ret = platformLock(lockType, wait, unLockFirstIfNeeded);
    if (ret) {
        //加锁成功，根据类型设置 加锁的次数
        if (lockType == SharedLockType) {
            m_sharedLockCount++;
        } else {
            m_exclusiveLockCount++;
        }
    }
    return ret;
}

//根据不同的平台执行加锁操作
bool FileLock::platformLock(LockType lockType, bool wait, bool unLockFirstIfNeeded) {
#    ifdef MMKV_ANDROID
    if (m_isAshmem) {
        return ashmemLock(lockType, wait, unLockFirstIfNeeded);
    }
#    endif
    //如果是文件的方式,首先获取到锁的类型，根据文件的类型，决定当前的加锁类型是文件锁，还是共享锁
    //在日常使用中，文件锁还会使用在另外一种场景下，即进程首先尝试对文件加锁，当加锁失败时，不希望进程阻塞，而是希望 flock 返回错误信息，进程进行错误处理后，继续进行下面的处理。在这种情形下就需要使用 flock 的非阻塞模式。把flock 的工作模式设置为非阻塞模式非常简单，只要将原有的 operation 参数改为锁的类型与 LOCK_NB 常量进行按位或操作即可，例如：
    //int ret = flock(open_fd, LOCK_SH | LOCK_NB);
    //int ret = flock(open_fd, LOCK_EX | LOCK_NB);
    //在非阻塞模式下，加文件锁失败并不影响进程流程的执行，但要注意加入错误处理逻辑，在加锁失败时，不能对目标文件进行操作。
    auto realLockType = LockType2FlockType(lockType);
    //这里是非阻塞的
    auto cmd = wait ? realLockType : (realLockType | LOCK_NB);
    if (unLockFirstIfNeeded) {
        // try lock 返回值 返回0表示成功，若有错误则返回-1，错误代码存于errno。
        //加写锁时，如果当前已经持有读锁，那么先尝试加写锁，try_lock 失败说明其他进程持有了读锁，
        //我们需要先将自己的读锁释放掉，再进行加写锁操作，以避免死锁的发生。
        auto ret = flock(m_fd, realLockType | LOCK_NB);
        if (ret == 0) {
            return true;
        }
        //可见，只有两个进程都对文件加的都是共享锁时，进程可以正常执行，不会阻塞，其他情形下，后加锁的进程都会被阻塞。
        // let's be gentleman: unlock my shared-lock to prevent deadlock   LOCK_UN 解除文件锁定状态。
        ret = flock(m_fd, LOCK_UN);
        if (ret != 0) {
            MMKVError("fail to try unlock first fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
        }
    }
    //flock()会依参数operation所指定的方式对参数fd所指的文件做各种锁定或解除锁定的动作。此函数只能锁定整个文件，无法锁定文件的某一区域。,这里是阻塞
    auto ret = flock(m_fd, cmd);
    if (ret != 0) {
        MMKVError("fail to lock fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
        // try recover my shared-lock
        if (unLockFirstIfNeeded) {
            ret = flock(m_fd, LockType2FlockType(SharedLockType));
            if (ret != 0) {
                // let's hope this never happen
                MMKVError("fail to recover shared-lock fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
            }
        }
        return false;
    } else {
        return true;
    }
}

这里要注意如果当前是在写锁模式下，而且当前还有读锁，我们要尝试的去获取写锁，当获取不成功的时候，要防区读锁，后续再来获取写锁，这里尝试去获取锁，使用 LOCK_NB 标识就不会阻塞
后续释放了之后，再去获取的化，是阻塞的
```
解锁操作
```java
/**
 * 解锁操作
 * @param lockType
 * @return
 */
bool FileLock::unlock(LockType lockType) {
    if (!isFileLockValid()) {
        return false;
    }
    bool unlockToSharedLock = false;

    if (lockType == SharedLockType) {
        //如果当前计数器为0了，直接返回false，解锁失败
        if (m_sharedLockCount == 0) {
            return false;
        }

        // don't want shared-lock to break any existing locks
        //如果当前 共享锁的计数值大于1，或者当前含有独占锁
        if (m_sharedLockCount > 1 || m_exclusiveLockCount > 0) {
            m_sharedLockCount--;
            return true;
        }
    } else {
        //当前已经为0了，返回false，解锁失败
        if (m_exclusiveLockCount == 0) {
            return false;
        }
        //大于1的时候，直接让独占锁减一处理，返回解锁成功
        if (m_exclusiveLockCount > 1) {
            m_exclusiveLockCount--;
            return true;
        }

        // restore shared-lock when all exclusive-locks are done
        //如果到了这里，那就说明，m_exclusiveLockCount 为 1，m_sharedLockCount 大于0，那么我们可以让读占锁降级为共享锁
        if (m_sharedLockCount > 0) {
            unlockToSharedLock = true;
        }
    }

    // unlockToSharedLock 为判断是否需要将锁降级
    auto ret = platformUnLock(unlockToSharedLock);
    //解锁成功，更改对应的计数器
    if (ret) {
        if (lockType == SharedLockType) {
            m_sharedLockCount--;
        } else {
            m_exclusiveLockCount--;
        }
    }
    return ret;
}
再看释放锁, 解写锁时，假如之前曾经持有读锁，那么我们不能直接释放掉写锁，这样会导致读锁也解了。我们应该加一个读锁，将锁降级，对于添加操作来说

bool MMKV::setDataForKey(MMBuffer &&data, MMKVKey_t key, bool isDataHolder) {
    ...
    //加锁,当前进程的锁
    SCOPED_LOCK(m_lock);
    //由于当前需要写文件，所以我们需要的锁问独占锁
    SCOPED_LOCK(m_exclusiveProcessLock);
    ...
}
```
### 多进程的同步

多进程实现细节
>首先我们简单回顾一下 MMKV 原来的逻辑。MMKV 本质上是将文件 mmap 到内存块中，将新增的 key-value 统统 append 到内存中；到达边界后，进行重整回写以腾出空间，空间还是不够的话，就 double 内存空间；对于内存文件中可能存在的重复键值，MMKV 只选用最后写入的作为有效键值。那么其他进程为了保持数据一致，就需要处理这三种情况：写指针增长、内存重整、内存增长。但首先还得解决一个问题：怎么让其他进程感知这三种情况？

状态同步
>写指针的同步
我们可以在每个进程内部缓存自己的写指针，然后在写入键值的同时，还要把最新的写指针位置也写到 mmap 内存中；这样每个进程只需要对比一下缓存的指针与 mmap 内存的写指针，如果不一样，就说明其他进程进行了写操作。事实上 MMKV 原本就在文件头部保存了有效内存的大小，这个数值刚好就是写指针的内存偏移量，我们可以重用这个数值来校对写指针。

内存重整的感知
>考虑使用一个单调递增的序列号，每次发生内存重整，就将序列号递增。将这个序列号也放到 mmap 内存中，每个进程内部也缓存一份，只需要对比序列号是否一致，就能够知道其他进程是否触发了内存重整。

内存增长的感知
>事实上 MMKV 在内存增长之前，会先尝试通过内存重整来腾出空间，重整后还不够空间才申请新的内存。所以内存增长可以跟内存重整一样处理。至于新的内存大小，可以通过查询文件大小来获得，无需在 mmap 内存另外存放。

其实每次在添加数据，修改数据，或者删除数据之前，都会调用下面的这个方法，来完成数据的校验，下面也就是多进程的同步实现了
```java
/**
 * 检查数据
 */
void MMKV::checkLoadData() {
    if (m_needLoadFromFile) {
        SCOPED_LOCK(m_sharedProcessLock);

        m_needLoadFromFile = false;
        loadFromFile();
        return;
    }
    if (!m_isInterProcess) {
        return;
    }

    if (!m_metaFile->isFileValid()) {
        return;
    }
    SCOPED_LOCK(m_sharedProcessLock);

    //获取到当前元数据文件的内容
    MMKVMetaInfo metaInfo;
    metaInfo.read(m_metaFile->getMemory());
    //判断当前元数据文件的内容跟我们是否一样,不一样的话，我们清除当前加载的内容，重新的加载文件
    if (m_metaInfo->m_sequence != metaInfo.m_sequence) {
        MMKVInfo("[%s] oldSeq %u, newSeq %u", m_mmapID.c_str(), m_metaInfo->m_sequence, metaInfo.m_sequence);

        //由于我们需要重新的加载文件，所以我们要有一独占锁，只要我们获取到了独占锁，那么其他的进程就不可能写
        SCOPED_LOCK(m_sharedProcessLock);
        //清除当前的文件内容
        clearMemoryCache();
        //重新的加载文件，完成数据的校验
        loadFromFile();
        notifyContentChanged();
    } else if (m_metaInfo->m_crcDigest != metaInfo.m_crcDigest) { //说明别的进程在修改
        MMKVDebug("[%s] oldCrc %u, newCrc %u, new actualSize %u", m_mmapID.c_str(), m_metaInfo->m_crcDigest,metaInfo.m_crcDigest, metaInfo.m_actualSize);
        SCOPED_LOCK(m_sharedProcessLock);

        size_t fileSize = m_file->getActualFileSize();
        //当一个进程发现了mmap写指针增长，就意味着其他进程写入了新键值，这些新的键值都append在原有写指针的后面，可能跟前面的key重复，也可能是全新的key，而
        //原写指针前面的键值都是有效的，那么我们就要把这些键值都读出来，插入或替换原有的键值，并将写指针同步到最新的位置
        if (m_file->getFileSize() != fileSize) {
            MMKVInfo("file size has changed [%s] from %zu to %zu", m_mmapID.c_str(), m_file->getFileSize(), fileSize);
            clearMemoryCache();
            loadFromFile();
        } else {
            partialLoadFromFile();
        }
        notifyContentChanged();
    }
}
```

### 参考链接
1. [C++11的constexpr关键字](https://www.cnblogs.com/fushi/p/7792257.html)
2. [flock 文件锁](https://blog.csdn.net/lqt641/article/details/54605920)
3. [MMKV for Android 多进程设计与实现](https://github.com/Tencent/MMKV/wiki/android_ipc)
4. [MMKV 原理](https://github.com/Tencent/MMKV/wiki/design)
5. [Android 比 SharedPreferences 更高效？微信 MMKV 源码解析](https://blog.n0texpecterr0r.cn/2020/01/05/%e9%87%8d%e8%af%bb-mmkv%ef%bc%9a%e5%90%8c%e6%a0%b7%e7%9a%84%e4%bb%a3%e7%a0%81%ef%bc%8c%e4%b8%8d%e5%90%8c%e7%9a%84%e6%84%9f%e6%82%9f/)
6. [关于SharedPreferences你要知道的一切](https://www.jianshu.com/p/f6e82473a0b7)















