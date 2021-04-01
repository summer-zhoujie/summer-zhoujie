---
layout:     post
title:      "Android Glide 3.7.0 源码解析(八) , RecyclableBufferedInputStream 的 mark/reset 实现"
subtitle:   ""
date:       2021-04-01 15:32:52
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

[个人博客传送门](http://summer-zhoujie.github.io/)
# 一、mark / reset 的作用
 [Android Glide 3.7.0 源码解析(七) , 细说图形变换和解码](/2021/03/31/markdown-glide3.7.0_7/index.html)有提到过RecyclableBufferedInputStream 对于 `mark(int marklimit)` 和 `reset()` 方法的作用, 本文则是探讨具体的实现思路

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210401101431339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
`mark(int marklimit)` 的作用是在流中创建一段起点是 markPos 长度是 markLimit 的可被重复读取区域, 当调用 `reset()` 方法时流的读取位置会回到 markPos , 实现 markLimit 可以被重复读取

注意:
* 当读取位置到达 readPos_2 时, `reset()` 方法会失效, 因为已经超出 markLimit 的长度范围了
* `mark(int marklimit)` 标记的起始点 markPos 为方法调用时流的读取位置( 上图中 mark 时, 读取位置是 readPos_0, 所以 markPos == readPos_0 )

>我们都知道 Stream 流只能被读取一遍

# 二、实现思路
上文中的图解还只是黑盒运行路线查看, 下面来分析具体实现的图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210401102442362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
由于 `InputStream` 只能被读取一遍就消耗了, 所以我们要做到重复读取必须借助一个外部工具, 此处用的是 `buf` 一个初始大小为 64K 的 byte[] 数组, 下面来对上图元素进行说明
* 右侧是还存在 `InputStream` 中的未被读取的数据
* 左侧是已经从流程读取出来的数据, 存在 `buf` 中
* `count` 代表的是 `buf` 中有效数据的数量 ( 你读到流的末尾了,可能会产生 `buf` 读不满的情况 )
* `markpos` 代表 mark() 标记的位置
* `蓝色条状` 则是需要支持被重复读取的部分了, `marklimit` 表示它的长度
* `pos` 代表当前正在读取的位置

现在我们有了 `buf` 作为缓存, 可以将 mark 的流数据缓存在 `buf` 中, 这样就可以重复读取了

> 注意: buf 大小不是恒定不变的, 当 marklimit > buf.length 并满足一些其他条件时, buf 会被2倍扩容

# 三、源码分析
看看 RecyclableBufferedInputStream 都有哪些变量(  其实在介绍思路的时候都讲过了 )

```java
public class RecyclableBufferedInputStream extends FilterInputStream {

  	// buf 缓存从流中读取出来的数据
    private volatile byte[] buf;
    // buf 有效数据的长度
    private int count;
    // mark 标记数据的长度
    private int marklimit;
    // mark 标记的位置, 默认 -1 表示没有标记
    private int markpos = -1;
    // 当前读取 buf 的位置
    private int pos;

}
```

## mark() 和 reset()
再来看看 mark 和 reset 方法

```java
	public synchronized void mark(int readlimit) {
        marklimit = Math.max(marklimit, readlimit);
        markpos = pos;
    }

	public synchronized void reset() throws IOException {
        if (buf == null) {
            throw new IOException("Stream is closed");
        }
        if (-1 == markpos) {
            throw new InvalidMarkException("Mark has been invalidated");
        }
        pos = markpos;
    }
```
mark 就做了两件事 赋值 + 赋值
* marklimit 和旧的作比较取一个最大值
* markpos 设置为当前读取位置

reset 就做了一件事
* 当前位置 pos 被重置成 markpos

## read()

来看看 read 方法干了啥

```java
	public synchronized int read() throws IOException {
        // Use local refs since buf and in may be invalidated by an
        // unsynchronized close()
        byte[] localBuf = buf;
        InputStream localIn = in;
        if (localBuf == null || localIn == null) {
            throw streamClosed();
        }

        // Are there buffered bytes available?
        if (pos >= count && fillbuf(localIn, localBuf) == -1) {
            // no, fill buffer
            return -1;
        }
        // localBuf may have been invalidated by fillbuf
        if (localBuf != buf) {
            localBuf = buf;
            if (localBuf == null) {
                throw streamClosed();
            }
        }

        // Did filling the buffer fail with -1 (EOF)?
        if (count - pos > 0) {
            return localBuf[pos++] & 0xFF;
        }
        return -1;
    }
```
这个 read() 方法是读取单个字符用的, 每次只读去 1 个byte 返回
*  pos >= count 代表 `buf` 被读完了,  调用 fillbuf(localIn, localBuf) 去重新向 `buf` 填充数据
* count - pos > 0 代表 `buf` 数据还够, 直接读取返回 localBuf[pos++] & 0xFF

> 这里 return 为毛是个 int ??? 说好的 byte 呢
>  因为返回的流数据里面是无符号的 byte 0~255 范围 , 而 java 里面的 byte 默认带符号, - 128 ~ 127, 数据范围不够, 只能用 int 来凑了, 注意这行代码:  `localBuf[pos++] & 0xFF` , 把高 24 位全部清零了, 达到返回 0~255 效果

这中间好像没有提到 mark 的部分, 猜测只能是在 `fillbuf` 函数中实现了, 跟进去看看

## fillbuf()
代码较长, 耐心看看
```java
	private int fillbuf(InputStream localIn, byte[] localBuf)
            throws IOException {
		// 没有标记, 或者读取区域超过了标记的范围, 直接从流中读取数据到 buf
        if (markpos == -1 || pos - markpos >= marklimit) {
            // Mark position not set or exceeded readlimit
            int result = localIn.read(localBuf);
            if (result > 0) {
                markpos = -1;
                pos = 0;
                count = result;
            }
            return result;
        }

        // [标记有效] 这里 buf.length 不够了, 扩容 2 倍
        if (markpos == 0 && marklimit > localBuf.length && count == localBuf.length) {
            // Increase buffer size to accommodate the readlimit
            int newLength = localBuf.length * 2;
            if (newLength > marklimit) {
                newLength = marklimit;
            }
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "allocate buffer of length: " + newLength);
            }
            byte[] newbuf = new byte[newLength];
            System.arraycopy(localBuf, 0, newbuf, 0, localBuf.length);
            localBuf = buf = newbuf;
        }

		// [标记有效] 标记位置 >0 buf中 标记位置之前还有一批数据可以被舍弃
		else if (markpos > 0) {
            System.arraycopy(localBuf, markpos, localBuf, 0, localBuf.length
                    - markpos);
        }
        // 重置读取位置和 buf 中有效数据的长度
        pos -= markpos;
        count = markpos = 0;
        // 如果 buf 不满(还有脏数据) 就开始从流中读取填充
        int bytesread = localIn.read(localBuf, pos, localBuf.length - pos);
        count = bytesread <= 0 ? pos : pos + bytesread;
        return bytesread;
    }
```
以上一共有三种情况
* 第一种: 没有 mark 参与或者mark已失效 (`markpos == -1 || pos - markpos >= marklimit`), 就是普通的从流中读取数据, 填满 buf , 然后把 pos 当前读取位置重置一下 ( 下图绿色表示扩容部分 )
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210401112114187.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
* 第二种, (`markpos == 0 && marklimit > localBuf.length && count == localBuf.length`)
   mark 有效, 且 markpos 左侧已经退不可退, 抵到 buf 的最左侧了;
   并且 marklimit 超过了 buf 的长度;
   且 `count == localBuf.length` 才进行 2 倍扩容

   >* 看`count == localBuf.length` 这个条件, 他表示 buf 里面的有效数据必须是满的才进行扩容, buf 只有在读到文件末尾时才可能不满, 当流都读完了, 没数据了, 即使 marklimit 超过 buf 的长度也没必要扩容了, 根本读不到那么多了;
   >* 看这几行代码
   >```java
   >int newLength = localBuf.length * 2;
   >if (newLength > marklimit) {
   >     newLength = marklimit;
   >}
   >```
   >如果扩充 2 倍后大于 marklimit 则以 marklimit为准, 否则, 就只扩充 2 倍, 不会一下子扩充到 marklimit 的大小, 对内存的使用可谓是抠搜到极致 ( 可以理解为懒申请, 用到了才去申请更大的 buf )


第二种情况 扩容一: marklimit < buf.length * 2 的情况
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210401134958485.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
2 倍超出 marklimit 的长度, 直接以 marklimit的长度为准, 上图绿色为扩容部分

第二种情况 扩容二: marklimit >= buf.length * 2 的情况

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210401135644502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
* 第三种情况 markpos != 0 并且不满足扩容条件

这种情况下, 只需要把 markpos 左边的数据清除, 然后再往 buf 右边空出的位置写入数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021040114061454.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
至此, `fillbuf()` 分析完毕, 一种分三种情况来填充 `buf` 上面都已做出图示说明


## read(byte[] buffer, int offset, int byteCount)

函数很长, 但是逻辑相对简单
```java
public synchronized int read(byte[] buffer, int offset, int byteCount) throws IOException {
        // 一系列错误检查就不看了, 非主线
        byte[] localBuf = buf;
        if (localBuf == null) {
            throw streamClosed();
        }

        if (byteCount == 0) {
            return 0;
        }
        InputStream localIn = in;
        if (localIn == null) {
            throw streamClosed();
        }

		// 先直接读取 buf
        int required;
        if (pos < count) {
            // There are bytes available in the buffer.
            int copylength = count - pos >= byteCount ? byteCount : count - pos;
            System.arraycopy(localBuf, pos, buffer, offset, copylength);
            pos += copylength;
            if (copylength == byteCount || localIn.available() == 0) {
            	// buf 里面就够读, 直接返回
                return copylength;
            }
            offset += copylength;
            // 不够读计算还需要多少
            required = byteCount - copylength;
        } else {
        	// buf 已经不能读了, 计算还需要多少
            required = byteCount;
        }

		// 开启循环读到够为止
        while (true) {
            int read;
            // 如果没有 mark 的影响, 直接从流本身读取
            if (markpos == -1 && required >= localBuf.length) {
                read = localIn.read(buffer, offset, required);
                if (read == -1) {
                	// 流读尽了直接返回
                    return required == byteCount ? -1 : byteCount - required;
                }
            } else {
            	// 开始利用 fillbuf() 填充 buf 数组
                if (fillbuf(localIn, localBuf) == -1) {
                    return required == byteCount ? -1 : byteCount - required;
                }
                // localBuf may have been invalidated by fillbuf
                if (localBuf != buf) {
                    localBuf = buf;
                    if (localBuf == null) {
                        throw streamClosed();
                    }
                }

				// 填充完开始读
                read = count - pos >= required ? required : count - pos;
                System.arraycopy(localBuf, pos, buffer, offset, read);
                pos += read;
            }
            required -= read;
            // 这里判断够不够数, 够的话直接返回
            if (required == 0) {
                return byteCount;
            }
            // 这里判断流有没有读尽, 读尽了直接返回已读的size
            if (localIn.available() == 0) {
                return byteCount - required;
            }
            offset += read;
        }
    }
```
逻辑其实很简单, 就是先读当前 buf 的内容, 不够的话进行下一步, 开启循环, 如果没有 mark , 则直接从流中读取, 否则, fillbuf 填充 buf 之后 从 buf 再读取一遍, 直至流被读尽或者读够想要的数字跳出循环返回
