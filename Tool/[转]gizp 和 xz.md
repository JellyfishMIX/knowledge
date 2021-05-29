# [转]gizp 和 xz



## 正文

晚上由于处理磁盘报警的需要，进行了日志压缩，在此次压缩中分别使用了gzip和xz软件对文本进行了压缩，压缩的结果非常令人诧异。

出于对xz好奇的原因是因为在下载内核源代码时经常可以看到.xz格式的文件包，而且其大小比.gz和.bz2格式的文件都小一些。首先简单介绍一下gzip和xz：
gzip：GZIP最早由Jean-loup Gailly和Mark Adler创建，用于UNⅨ系统的文件压缩。我们在Linux中经常会用到后缀为.gz的文件，它们就是GZIP格式的。现今已经成为Internet 上使用非常普遍的一种数据压缩格式，或者说一种文件格式。关于gzip更详细的详细介绍可以参见百度百科。
xz：xz是一种压缩文件格式，采用LZMA SDK压缩，目标文件较gzip压缩文件(.gz或·tgz)小30%，较·bz2小15%。更详细的介绍可以参见维基百科。

这里由于磁盘空间有限，我们使用的是-9参数来进行压缩，使用每种压缩的最高压缩比，使用的名利如下：

```shell
# gzip
gzip -9 -c source_filename > source_filename.gz
# xz
xz -9 -c source_filename > source_filename.xz
```

最终测试结果如下图所示： 

![这里写图片描述](https://image-hosting.jellyfishmix.com/20210529121000.png)

从图中可以看出源文件iqas-2015-07-28的大小为35G，经过gzip的压缩，最终文件为4.6G的大小，压缩比大约为1:8，这已经是一个很好的成绩了，但是再来看下经过xz压缩后的文件的大小为401M，这相比gzip来说又小了一个数量级，压缩比接近1:90，这对于文本存储来说无疑是节省了大量的空间。但是需要注意的是xz的9级压缩非常耗时间和内存，如果时间和内存足够的情况下，可以考虑该方法，如果时间和内存比较紧，则建议使用gzip。



## 转自

[关于压缩软件gzip和xz的简单对比 - ucan23 - CSDN](https://blog.csdn.net/ucan23/article/details/47772237)