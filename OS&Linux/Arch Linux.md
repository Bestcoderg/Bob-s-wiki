# Arch Linux

[以官方Wiki的方式安装ArchLinux](https://www.viseator.com/2017/05/17/arch_install/)

[ArchLinux安装后的必须配置与图形界面安装教程](https://www.viseator.com/2017/05/19/arch_setup/)

然后选择 分区表的方式，通常有两种：

- **uefi + gpt + efi：** 推荐，比较适合 ssd 等大型新型磁盘 管理
- **legacy（BIOS） + mbr：** 以前老式 win7 的机械硬盘管理方式

安装并设置bootloader就可以让你的机器正常的启动而不是每次都要用Arch Linux LiveCD引导，挂载上述那些分区，然后arch-chroot过去。我们这里选择用GRUB，你也可以根据自己的喜好选择其他的bootloader。

pacman -S efibootmgr dosfstools grub os-prober

计算机启动引导流程：

BIOS

MBR

boot loader

内核文件

[https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722580.html](https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722580.html)

[https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722615.html](https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722615.html)

[https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722929.html](https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2722929.html)

[https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2723127.html](https://www.cnblogs.com/ggjucheng/archive/2012/10/13/2723127.html)

### **1、添加新的用户账号使用useradd命令，其语法如下：**

```
useradd 选项 用户名
```

参数说明：

- 选项:
    - c comment 指定一段注释性描述。
    - d 目录 指定用户主目录，如果此目录不存在，则同时使用-m选项，可以创建主目录。
    - g 用户组 指定用户所属的用户组。
    - G 用户组，用户组 指定用户所属的附加组。
    - s Shell文件 指定用户的登录Shell。
    - u 用户号 指定用户的用户号，如果同时有-o选项，则可以重复使用其他用户的标识号。
- 用户名:指定新账号的登录名。

### 实例1

```
# useradd –d  /home/sam -m sam
```

此命令创建了一个用户sam，其中-d和-m选项用来为登录名sam产生一个主目录 /home/sam（/home为默认的用户主目录所在的父目录）