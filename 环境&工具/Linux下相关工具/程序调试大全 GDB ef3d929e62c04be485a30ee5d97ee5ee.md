# 程序调试大全 / GDB

## GDB命令大全

[GDB](%E7%A8%8B%E5%BA%8F%E8%B0%83%E8%AF%95%E5%A4%A7%E5%85%A8%20GDB%20ef3d929e62c04be485a30ee5d97ee5ee/GDB%20edf9da06789e426092ea545f2132fe17.csv)

### 如何进入GDB调试？

- 查看进程id : ps -ef |grep 你的用户名
- 挂载对应进程 : gdb -p 进程id

### GDB是如何查找源码的？

GDB 之所以能够知道对应的源代码，是因为调试版的可执行程序中记录了源代码的位置；因为源代码的位置在编译之后可能会移动到其它地方，所以 GDB 还会在当前目录中查找源代码，另外 GDB 也允许明确指定源代码的搜索位置。默认情况下，GDB 在编译时目录中搜索，如果失败则在当前目录中搜索，即 $cdir:$cwd，其中 $cdir 指的是编译时目录（compilation directory），$cwd 指的是当前工作目录（current working directory）。可以在GDB中输入 show directories，可以看到源码的搜索路径为 $cdir:$cwd。

值得注意的是，$cdir 是编译时目录，指的应该是CMakeLists.txt 执行时所在目录。可以通过

```cpp
readelf -p .debug_str <elffile>
```

的方式，打印一下信息，然后在里面找到路径，可以看到其存储了一个绝对路径。

![Untitled](%E7%A8%8B%E5%BA%8F%E8%B0%83%E8%AF%95%E5%A4%A7%E5%85%A8%20GDB%20ef3d929e62c04be485a30ee5d97ee5ee/Untitled.png)

## dmesg + addr2line

如果有的时候没有生成core文件，可以使用dmesg+addr2line的方式来查看进程崩溃的信息。Linux会记录程序异常退出时候的指令寄存器地址，将这条指令地址用 addr2line 命令解析为具体的行数。注意在编译过程中需要添加 -g 选项保留debug信息。

```bash
g++ test.cpp -o test **-g**
**dmesg -T**
//traps: test[123867] trap divide error ip:400cac sp:7ffc60d18520 error:0 in test[400000+3000]
**addr2line -e test -a 0x400cac -fC**
```

## readelf

当我们需要查看符号表，或是-g导出的信息时，我们可以使用readelf这个命令。一般来说，除了 -g 的debug信息，其他信息全部使用 readelf -a 来输出，其中最重要的是符号表信息，这个操作与 nm 命令相似。当需要查看 -g 的debug信息时，需要添加参数 —debug-dump 的参数。

```bash
readelf test -a | grep func 
readelf test --debug-dump > elf.info
```

*@Bestcoderg 2021/2/1*