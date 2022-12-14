# 文件系统类型 & Linux 文件系统

这篇文章是在 [Linux 文件挂载](https://www.cnblogs.com/doggod/p/13371005.html)的启发下而来，在前文中我探索了 Linux 在操作系统层面对设备 & 磁盘的关系，Linux 下所有的设备 & 磁盘都被挂载到了 Linux 所虚拟的文件树上。所以在这后，我们向下深挖一层，来探究一下所挂载的磁盘是如何管理 & 访问 & 储存数据的，即文件系统的作用。

文件系统是什么？我认为文件系统就是对磁盘中数据进行储存 & 管理 & 保护等等我们想对文件进行操作的行为逻辑的一个集合。文件系统保证了我们对磁盘中数据的储存和读取以及保护等等功能。而在平时我们对文件系统最直接的接触应该就是进行安装系统 & 对分盘分区时会叫我们选择文件系统的类型（一个硬盘分区只能够使用一种文件系统）。

当然，文件系统会有很多，每种文件系统有自己的优缺点和独有特性。我们可以参考 [FileSystem](https://wiki.archlinux.org/index.php/File_systems_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))。

对于我们来说，只用熟悉一些经常使用的文件系统便是可以的了。**对于 Windows&Mac OS 用户**来说，是没有选择文件系统的权力的，对于这两个操作系统而言，分别只有一种选择，那就是 **NTFS 和 HFS+。**对于 Linux 用户而言，还是有着很多文件系统可以选择的，现在默认选择的是**广泛采用的 ext4**。现在也有着改用 **btrfs** 的趋势，不过 btrfs 相对 ext4 的优势对于普通用户来说实在是比较遥远，且 ext4 速度较快，所以说短时间普通用户可能还不会使用上 btrfs。

在我们具体讨论 Linux 的各个文件系统之前，我们可以使用 **lsblk -f 命令**先查看一下自己所挂载的硬盘所使用的文件系统。

![https://img2020.cnblogs.com/blog/1220845/202007/1220845-20200730170210954-393593226.png](https://img2020.cnblogs.com/blog/1220845/202007/1220845-20200730170210954-393593226.png)

可以看到，在我 Centos7 电脑下，**sda 下使用的 XFS 文件系统，sdb 下使用的是 ext4 文件系统**。由于是分配的虚拟机，我的推测是 Centos 安装时默认的文件系统是 XFS，然后挂载了一块硬盘，这块硬盘之后被初始化的文件系统时 ext4 。至于创建文件系统什么操作的我既不做了 QAQ

## ext4 & XFS & btrfs

ext4 是第四代扩展文件系统（英语：Fourth EXtended filesystem，缩写为 ext4）是 linux 系统下的日志文件系统，是 ext3 文件系统的后继版本 ext4 的文件系统容量达到 1EB，而文件容量则达到 16TB，这是一个非常大的数字了。对一般的台式机和服务器而言，这可能并不重要，但对于大型磁盘阵列的用户而言，这就非常重要了。ext3 目前只支持 32000 个子目录，而 ext4 取消了这一限制，理论上支持无限数量的子目录

xfs 是一种非常优秀的日志文件系统，它是 SGI 公司设计的。xfs 被称为业界最先进的、最具可升级性的文件系统技术。xfs 是一个 64 位文件系统，最大支持 8EB 的单个文件系统，实际部署时取决于宿主操作系统的最大块限制。对于一个 32 位 Linux 系统，文件和文件系统的大小会被限制在 16TB。xfs 在很多方面确实做的比 ext4 好，ext4 受限制于磁盘结构和兼容问题，可扩展性和 scalability 确实不如 xfs，另外 xfs 经过很多年发展，各种锁的细化做的也比较好。（**相对 ext4，XFS 是可以扩容的，不能缩小**）

btrfs 有很多不同的叫法，例如 Better FS、Butter FS 或者 B-Tree FS。它是一个几乎完全从头开发的文件系统。btrfs 出现的原因是它的开发者起初希望扩展文件系统的功能使得它包括快照、池化 pooling、校验以及其它一些功能。虽然和 ext4 无关，它也希望能保留 ext4 中能使消费者和企业受益的功能，并整合额外的能使每个人，尤其是企业受益的功能。对于使用大型软件以及大规模数据库的企业，让多种不同的硬盘看起来一致的文件系统能使他们受益并且使数据整合变得更加简单。删除重复数据能降低数据实际使用的空间，当需要镜像一个单一而巨大的文件系统时使用 btrfs 也能使数据镜像变得简单。用户当然可以继续选择创建多个分区从而无需镜像任何东西。考虑到这种情况，btrfs 能横跨多种硬盘，和 ext4 相比，它能支持 16 倍以上的磁盘空间。btrfs 文件系统一个分区最大是 16 exbibytes，最大的文件大小也是 16 exbibytes。

不幸的是，还不知道 btrfs 什么时候能到来。官方说，其下一代文件系统仍然被归类为 “不稳定”，但是如果用户下载最新版本的 Ubuntu，就可以选择安装到 btrfs 分区上。什么时候 btrfs 会被归类到 “稳定” 仍然是个谜， 直到真的认为它“稳定” 之前，用户也不应该期望 Ubuntu 会默认采用 btrfs。有报道说 Fedora 18 会用 btrfs 作为它的默认文件系统，因为到了发布它的时候，应该有了 btrfs 文件系统校验器。由于还没有实现所有的功能，另外和 ext4 相比性能上也比较缓慢，btrfs 还有很多的工作要做。

那么，究竟使用哪个更好呢？尽管性能几乎相同，但 ext4 还是赢家。为什么呢？答案在于**易用性以及广泛性**。对于桌面或者工作站， ext4 仍然是一个很好的文件系统。由于它是默认提供的文件系统，用户可以在上面安装操作系统。同时， ext4 支持最大 1 exabytes 的卷和 16 terabytes 的文件，因此考虑到大小，它也还有很大的进步空间。**btrfs 能提供更大的高达 16 exabytes 的卷以及更好的容错**，但是，到现在为止，它感觉更像是一个附加的文件系统，而部署一个集成到 Linux 操作系统的文件系统。比如，尽管 btrfs 支持不同的发行版，**使用 btrfs 格式化硬盘之前先要有 btrfs-tools 工具，这意味着安装 Linux 操作系统的时候它并不是一个可选项，即便不同发行版之间会有所不同**。

尽管传输速率非常重要，评价一个文件系统除了文件传输速度之外还有很多因素。btrfs 有很多好用的功能，例如写复制 Copy-on-Write、扩展校验、快照、清洗、自修复数据、冗余删除以及其它保证数据完整性的功能。和 ZFS 相比 btrfs 缺少 RAID-Z 功能，因此对于 btrfs， RAID 还处于实验性阶段。对于单纯的数据存储，和 ext4 相比 btrfs 似乎更加优秀，但时间会验证一切。迄今为止，对于桌面系统而言，ext4 似乎是一个更好的选择，因为它是默认的文件系统，传输文件时也比 btrfs 更快。btrfs 当然值得尝试、但要在桌面 Linux 上完全取代 ext4 可能还需要一些时间。数据场和大存储池会揭示关于 ext4、XCF 以及 btrfs 不同的场景和差异。以上部分内容转载自：[Linux 中国](https://linux.cn/article-7083-1.html)

至于文件系统是怎么做的，可以看[文件系统介绍](https://blog.csdn.net/wxh0000mm/article/details/81773252)