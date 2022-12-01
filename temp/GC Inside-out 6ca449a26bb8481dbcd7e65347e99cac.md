# GC Inside-out

MS → MS(bitmap marking) → Mark-Compact → Mark-Sweep-Compact

Mark-Copying

Generational-GC

## GC 简介

GC - Garbage Collection 即垃圾回收，作为计算机科学领域非常热的研究话题之一，最早可追溯到 1959 年，由 John McCarthy 在 **Lisp** 中实现来**简化内存管理**。早期的 **Lisp** 之所以被大众诟病慢，主要原因就是当时的 GC 实现相对简单，对程序的影响（**overhead**）比较严重。经过几十年的发展，GC 算法已经很成熟了，可以完全摆脱「速度慢」这个让人望而却步的标签。

单就 JVM 这个平台来说，GC 算法一直在优化、演变，从最初的串行到高吞吐量的并行，为了解决高延迟又演化出了 CMS（Concurrent Mark Sweep），为了解决碎片问题，又开发了 G1，Oracle 内部还在不断尝试新算法，比如 ZGC。

### 我们为什么需要GC？

[https://liujiacai.net/blog/2018/06/15/garbage-collection-intro/#引用计数（Reference-counting）](https://liujiacai.net/blog/2018/06/15/garbage-collection-intro/#%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%EF%BC%88Reference-counting%EF%BC%89)

[https://liujiacai.net/blog/2018/07/08/mark-sweep/](https://liujiacai.net/blog/2018/07/08/mark-sweep/)

[https://liujiacai.net/blog/2018/08/04/incremental-gc/](https://liujiacai.net/blog/2018/08/04/incremental-gc/)

[https://liujiacai.net/blog/2018/08/18/generational-gc/](https://liujiacai.net/blog/2018/08/18/generational-gc/)

[https://docs.microsoft.com/zh-cn/dotnet/standard/automatic-memory-management](https://docs.microsoft.com/zh-cn/dotnet/standard/automatic-memory-management)

[https://docs.microsoft.com/zh-cn/dotnet/standard/garbage-collection/fundamentals](https://docs.microsoft.com/zh-cn/dotnet/standard/garbage-collection/fundamentals)

[https://www.cnblogs.com/Tonarinototoro/p/14830804.html](https://www.cnblogs.com/Tonarinototoro/p/14830804.html)

cheney

[https://blog.csdn.net/hel_wor/article/details/50446567](https://blog.csdn.net/hel_wor/article/details/50446567)

[https://hllvm-group.iteye.com/group/topic/39376#post-257329](https://hllvm-group.iteye.com/group/topic/39376#post-257329)

[https://zhuanlan.zhihu.com/p/37996721#:~:text=Cheney 算法是一种采用复制的方式实现的垃圾回收算法。 它将堆内存一分为二%2C每一部分空间称为,semispace。 在这两个 semispace 空间中%2C只有一个处于使用中%2C另一个处于闲置状态。](https://zhuanlan.zhihu.com/p/37996721#:~:text=Cheney%20%E7%AE%97%E6%B3%95%E6%98%AF%E4%B8%80%E7%A7%8D%E9%87%87%E7%94%A8%E5%A4%8D%E5%88%B6%E7%9A%84%E6%96%B9%E5%BC%8F%E5%AE%9E%E7%8E%B0%E7%9A%84%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E7%AE%97%E6%B3%95%E3%80%82%20%E5%AE%83%E5%B0%86%E5%A0%86%E5%86%85%E5%AD%98%E4%B8%80%E5%88%86%E4%B8%BA%E4%BA%8C%2C%E6%AF%8F%E4%B8%80%E9%83%A8%E5%88%86%E7%A9%BA%E9%97%B4%E7%A7%B0%E4%B8%BA,semispace%E3%80%82%20%E5%9C%A8%E8%BF%99%E4%B8%A4%E4%B8%AA%20semispace%20%E7%A9%BA%E9%97%B4%E4%B8%AD%2C%E5%8F%AA%E6%9C%89%E4%B8%80%E4%B8%AA%E5%A4%84%E4%BA%8E%E4%BD%BF%E7%94%A8%E4%B8%AD%2C%E5%8F%A6%E4%B8%80%E4%B8%AA%E5%A4%84%E4%BA%8E%E9%97%B2%E7%BD%AE%E7%8A%B6%E6%80%81%E3%80%82)

[https://www.bilibili.com/video/BV1D741177rV/?spm_id_from=333.788.recommend_more_video.1](https://www.bilibili.com/video/BV1D741177rV/?spm_id_from=333.788.recommend_more_video.1)

[https://www.bilibili.com/video/BV1ek4y1z7V6?from=search&seid=8131313048323286355](https://www.bilibili.com/video/BV1ek4y1z7V6?from=search&seid=8131313048323286355)

[https://www.bilibili.com/video/BV1Qz411q7wC?from=search&seid=8131313048323286355](https://www.bilibili.com/video/BV1Qz411q7wC?from=search&seid=8131313048323286355)

Java : GCroot

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%201.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%201.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%202.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%202.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%203.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%203.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%204.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%204.png)

分代假设：

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%205.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%205.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%206.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%206.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%207.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%207.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%208.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%208.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%209.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%209.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2010.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2010.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2011.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2011.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2012.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2012.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2013.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2013.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2014.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2014.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2015.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2015.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2016.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2016.png)

![GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2017.png](GC%20Inside-out%206ca449a26bb8481dbcd7e65347e99cac/Untitled%2017.png)