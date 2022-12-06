# Makefile编写

在当前的项目开发中，Makeflie是一个必不可少的工具。对于大型的项目，只用写好Makefile就可以帮助你自动编译，对于不了解项目架构的人来说实在是一个极好的工具。或许偷懒就是这个世界技术进步的动力把，嘻嘻~

在了解了gcc之后，如果我们想要编译一个test.cpp文件我们可能会使用gcc test.cpp -o test -lstdc++命令直接编译这个C++文件，但假设现在有两个.h,两个.cpp我们想要编译这个项目可能就要按照一定的顺序执行两三条编译命令了，当项目越来越大文件越来越多的时候，再由自己按顺序执行编译的命令显然是一件费时费力的事情了。而Makefile就是为了解决这种问题而生的，我们只需要将每个文件相互的调用编译关系&编译命令写入makefile文件，Makefile就会做一个拓扑将需要先编译的文件先编译，保证编译命令的成功执行，相当于程序员不需要自己去解决文件编译的顺序了，只需要搞清文件之间的调用关系就可以了。

## Makefile基本规则

```makefile
TARGET... :PREREQUISITES...
        COMMAND
        ...
        ...
```

以上这段是Makefile最基础的描述规则，也是最精华的部分，其他部分都是围绕着这个最基本的描述规则的。先来解释一下：

### **TARGET：**

规则生成的**目标文件**，通常是需要生成的程序名（例如前面出现的程序名test）或者过程文件（类似.o文件）。

如果TARGET是多个文件名，make时会依次遍历，如果当前遍历的目标文件没有生成的话，就会执行一次TARGET所对应的命令。我们可以通过$@每次取出需要操作的单个文件，进行我们想要进行的操作。由于对于多个文件每次都会检查当前文件是否生成，所以一般TARGERT中要写这个目标文件最终的位置和名称，如果写中间文件的话，由于中间文件最后位置的移动。如果再次make时，make将依旧找不到这个目标文件，就会再次执行对应的命令。

### **PREREQUISITES：**

**规则的依赖项**，比如前面的Makefile文件中我们生成test程序所依赖的就是test.cpp。

如果某个目标的依赖项没有生成，则makefile会先去找将这个目标作为目标文件的模块，这种形式是形成拓扑的关键。

### **COMMAND：**

规则所需执行的**命令行**，通常是编译命令。这里需要注意的是每一行命令都需要以[TAB]字符开头。

同时，Makefile为了防止未修改的文件增加编译量，规定了重新编译文件的规则：

1、 如果目标test文件不存在，根据规则创建它。

2、 目标test文件存在，并且test文件的依赖项中存在任何一个比目标文件更新（比如修改了一个函数，文件被更新了），根据规则重新生成它。

3、 目标test文件存在，并且它比所有的依赖项都更新，那么什么都不做。

相当于通过拓扑图去寻找需要重新生成的文件，这种通过拓扑图寻找需要生成文件的形式在第一次构造的时候也是一样，这是makefile工作的核心模式。

## 参数&符号&函数

[Makefile](Makefile%E7%BC%96%E5%86%99%201eff9d61e07e44caa730b75d91cbbf9e/Makefile%2052d8917e34a24ffab1bb37f34d50ff0a.csv)

## Makefile使用特性

### shell变量&makefile变量

```makefile
var=3                    # a
target:
       echo $(var)       # b
       var=4             # c  (注意，shell脚本中定义变量”=“旁边不能有空格)
       echo $(var)       # d
       echo $$var        # e
```

a：定义**Makefile中的变量var**，值为3

b：打印Makefile中的变量，值为3

c：定义shell命令中的变量var，值为4，Makefile的变量var不受影响

d：打印Makefile中的变量，值为3

e：**打印shell命令中的变量。此时var为未定义的变量。**

我们可能会奇怪，shell命令中的var明明在#c已经定义了，为什么是未定义呢？

原因：在Makefile的规则命令，如果相互之间没有使用';\'连接起来的话，相互之间是不能共享变量的。即使用‘\’相链接的命令才能共享变量。

改为以下形式：

```makefile
var=3                    # a
target:
       echo $(var)       # b
       var=4;\             # c  (注意，shell脚本中定义变量”=“旁边不能有空格)
       echo $(var);\       # d
       echo $$var;\        # e
```

### 查看文件夹是否存在

使用shell命令if；以前居然从来没有用过，还以为是makfile的命令QAQ

```bash
$(shell if [ -d src ];then echo "exit";else mkdir build;fi)
```

注意，在if 、src后面一定要加上空格，不然shell没办法正确区分”]“，会查询失败。

注意命令中echo ”exit“的部分，这里一定是”exit“，为什么不能是随便什么你想输出的提示呢？因为当前是在shell 命令中，echo ”exit“后，shell会执行exit命令，所以一定要输出符合条件的命令。

### 命令行开头符号

命令行以'@'打头的含义： 在执行到的时候不回显相应的命令内容，只显示命令的输出。

命令行以'+'打头的含义： makefile中以+开头的命令的执行不受到 make的-n,-t,-q三个参数的影响。我们知道，在make的时候，如果加上-n, -t, -q这样的参数，都是不执行相应命令的，而以'+'开头的命令，则无论make命令后面是否跟着三个参数，都会被执行。

## Makefile实例

### 1、多个文件，同个目录下编译

对于test.cpp，现有w1.h,w2.h,w3.h,w1.cpp,w2.cpp,w3.cpp。test.cpp引用了w1,w2,w3。

```makefile
test:test.o w1.o w2.o w3.o
        g++ test.o w1.o w2.o w3.o -o test
test.o:test.cpp
        g++ -c test.cpp -o test.o
w1.o:w1.cpp
        g++ -c w1.cpp -o w1.o
w2.o:w2.cpp
        g++ -c w2.cpp -o w2.o
w3.o:w3.cpp
        g++ -c w3.cpp -o w3.o
clean:
        rm -f test.o
        rm -f w*.
```

显然前面一种方式并不够优雅，我们需要一种优雅的改进：

```makefile
TARGET = test

SRC_FILE = $(wildcard *.cpp)
BUILD_FILE = $(patsubst %.cpp,%.o,$(SRC_FILE))

$(TARGET):$(BUILD_FILE)
        g++ $^ -o $@
$(BUILD_FILE):
        g++ -c $(*).cpp
clean:
        rm -f *.o
        rm -f test
```

在这种方式中g++ -c $(*).cpp是整个的关键，要知道虽然BUILD_FILE中有多个文件，但make时是依次遍历每个未生成的文件，所以使用$*取出的是单个的不带后缀的文件名。

### 2、多个文件，不同目录下编译

上一个例子中，我们成功用makefile编译了一个多文件的项目，但是这个项目中所有的文件都是在同一个目录下面的，在实际的项目中这显然是行不通的，我们需要将资源/头文件等分开储存。所以现在我们需要实现以下这个文件结构：建立src、include、build分别储存.cpp、.h、.o文件。

```makefile
TARGET = test

BUILD_PATH = ./build/
SRC_PATH = ./src/
INCLUDE_PATH = ./include/

SRC_FILE = $(wildcard $(SRC_PATH)*.cpp)
BUILD_FILE_NAME = $(patsubst %.cpp,%.o,$(notdir $(SRC_FILE)))
BUILD_FILE = $(patsubst $(SRC_PATH)%.cpp,$(BUILD_PATH)%.o,$(SRC_FILE))
INCLUDE_FILE = $(wildcard $(INCLUDE_PATH)*.h)

$(TARGET):$(BUILD_FILE)
#       echo $(shell ls -a)
        g++ $(BUILD_FILE) -o $@
$(BUILD_FILE):
#       echo $(BUILD_FILE)
        g++ -c $(SRC_FILE) -I $(INCLUDE_PATH)
        $(shell if [ -d $(BUILD_PATH) ];then echo "exit" ;else mkdir $(BUILD_PATH);fi)
        mv $(BUILD_FILE_NAME) $(BUILD_PATH)
clean:
        find . -name "*.o"|xargs rm -f
        rm -f test
```

上面的代码虽然可以实现make，但仿佛还不是足够的优雅。在实际操作过程中发现，由于所有的.cpp是被一起编译成为.o文件的，使用的是一条命令，所以当其中某个.cpp/.o文件被改变后，所有的文件都需要一起被再次编译，这显然是不符合make的精神的。所以我们需要一种能够一次编译每个.cpp文件的makefile代码：

```makefile
TARGET = test                                                                          
                                                                                       
BUILD_PATH = ./build/                                                                  
SRC_PATH = ./src/                                                                      
INCLUDE_PATH = ./include/                                                              
                                                                                       
SRC_FILE = $(wildcard $(SRC_PATH)*.cpp)                                                
#BUILD_FILE_NAME = $(patsubst %.cpp,%.o,$(notdir $(SRC_FILE)))                         
BUILD_FILE = $(patsubst $(SRC_PATH)%.cpp,$(BUILD_PATH)%.o,$(SRC_FILE))                 
INCLUDE_FILE = $(wildcard $(INCLUDE_PATH)*.h)                                          
                                                                                       
                                                                                       
$(TARGET):$(BUILD_FILE)                                                                
#       echo $(shell ls -a)                                                            
        g++ $(BUILD_FILE) -o $@                                                        
$(BUILD_FILE):                                                                         
#       echo $(*F)                                                                     
        g++ -c $(SRC_PATH)$(*F).cpp -I $(INCLUDE_PATH)                                 
        $(shell if [ -d $(BUILD_PATH) ];then echo "exit" ;else mkdir $(BUILD_PATH);fi) 
        mv $(@F) $(BUILD_PATH)                                                         
clean:                                                                                 
        find . -name "*.o"|xargs rm -f                                                 
        rm -f test
```

2020/8/6 @葛华盛