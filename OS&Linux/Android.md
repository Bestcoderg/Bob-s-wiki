# Android

**一、Android为什么会选择Linux**

成熟的操作系统有很多，但是Android为什么选择采用Linux内核呢?这就与Linux的一些特性有关了，这也是很多

教材反复讲到的linux的重要特点。比如：

1、强大的内存管理和进程管理方案

2、基于权限的安全模式

3、支持共享库

4、经过认证的驱动模型

5、Linux本身就是开源项目

更多关于上述特性的信息可以参考Linux 2.6版内核的官方文档，这便于我们在后面的学习中更好地理解Android

所特有的功能特性。接下来分析Android与Linux的关系。其实实际上选择linux内核的手机系统很多，记得前几年

就见过三星的一款linux内核的手机，并且那款手机保持了linux系统的大部分特征，所以用起来感觉就像一个小巧

的linux系统。

**二、Android对Linux的改动**

原文作者说是“Android不是Linux”，关于这个观点，要看读者自己怎么看了，如果说Linux是说的内核，那

Android自然不是Linux。如果Linux是指Linux发行版，那Android当然是Linux，否则ubuntu，Fedora等都不是

linux了。

Android对linux系统的改动主要有以下几个方面：

**1.它没有glibc支持**

由于Android最初用于一些便携的移动设备上，所以，可能出于效率等方面的考虑，Android并没有采用glibc作为

C库，而是Google自己开发了一套Bionic Libc来代替glibc。

**2.它并不包括一整套标准的Linux使用程序**

Android并没有完全照搬Liunx系统的内核，除了修正部分Liunx的Bug之外，还增加了不少内容，比如：它基于ARM

构架增加的Gold-Fish平台，以及yaffs2 FLASH文件系统（如果学习了嵌入式的话就会知道yaffs2 FLASH文件系

统已经在基于linux的很多嵌入式设备上采用了，技术已经非常成熟）等。

**3.它没有本地基于X服务的窗口系统**

什么是本地窗口系统呢?本地窗口系统是指GNU/Linux上的X窗口系统，或者Mac OX X的Quartz等。不同的操作系统

的窗口系统可能不一样，Android并没有使用(也不需要使用)Linux的X窗口系统（对原作者的这个观点不是很赞

同，原文章这一点放在第一条，并说“这是Android不是Linux的一个基本原因”，这个不敢苟同，由于作者 没有

指明android用的什么显示系统，我也不好说）。

**4.Android专有的驱动程序**

除了上面这些不同点之外，Android还对Linux设备驱动进行了增强，主要如下所示。

**1)**Android Binder 基于OpenBinder框架的一个驱动，用于提供 Android平台的进程间通信(InterProcess

Communication，IPC)功能。源代码位于drivers/staging/android/binder.c。

**2)**Android电源管理(PM) 一个基于标准Linux电源管理系统的轻量级Android电源管理驱动，针对嵌入式设备做

了很多优化。源代码位于：

kernel/power/earlysuspend.c

kernel/power/consoleearlysuspend.c

kernel/power/fbearlysuspend.c

kernel/power/wakelock.c

kernel/power/userwakelock.c

**3)**低内存管理器(Low Memory Killer) 比Linux的标准的OOM(Out Of Memory)机制更加灵活，它可以根据需要

杀死进程以释放需要的内存。源代码位于 drivers/staging/ android/lowmemorykiller.c。

**4)**匿名共享内存(Ashmem) 为进程间提供大块共享内存，同时为内核提供回收和管理这个内存的机制。源代码位于

mm/ashmem.c。

**5)**Android PMEM(Physical) PMEM用于向用户空间提供连续的物理内存区域，DSP和某些设备只能工作在连续的物

理内存上。源代码位于drivers/misc/pmem.c。

**6)**Android Logger 一个轻量级的日志设备，用于抓取Android系统的各种日志。源代码位于

drivers/staging/android/logger.c。

**7)**Android Alarm 提供了一个定时器，用于把设备从睡眠状态唤醒，同时它还提供了一个即使在设备睡眠时也会

运行的时钟基准。源代码位于drivers/rtc/alarm.c。

**8)**USB Gadget驱动 一个基于标准 Linux USB gadget驱动框架的设备驱动，Android的USB驱动是基于gaeget框

架的。源代码位于drivers/usb/gadget/。

**9)**Android Ram Console 为了提供调试功能，Android允许将调试日志信息写入一个被称为RAM Console的设备

里，它是一个基于RAM的Buffer。源代码位于drivers/staging/android / ram_console.c。

**10)**Android timed device 提供了对设备进行定时控制的功能，目前支持vibrator和LED设备。源代码位于

drivers/staging/android /timed_output.c(timed_gpio.c)。

**11)**Yaffs2 文件系统 Android采用Yaffs2作为MTD nand flash文件系统，源代码位于fs/yaffs2/目录下。

Yaffs2是一个快速稳定的应用于NAND和NOR Flash的跨平台的嵌入式设备文件系统，同其他Flash文件系统相比，

Yaffs2能使用更小的内存来保存其运行状态，因此它占用内存小。Yaffs2的垃圾回收非常简单而且快速，因此能表

现出更好的性能。Yaffs2在大容量的NAND Flash上的性能表现尤为突出，非常适合大容量的Flash存储。

上面这些要点足以说明Android不是Linux。学习应用Android一般围绕Android的这些特有的部分展开，建议大家先复习一下Linux内核的基本知识。在具体学习之前，先来总体浏览一下

Android对Linux内核进行了哪些改动，在移植时就需要对这些改动加以调整。