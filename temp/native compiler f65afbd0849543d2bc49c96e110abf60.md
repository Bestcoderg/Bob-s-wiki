# native compiler

本文尝试以GCC为例，解释一下什么是native compiler，什么是cross compiler。

首先介绍三个概念——build、host和target。

**build -** 编译GCC的平台

**host -** 运行GCC的平台

**target -** GCC编译产生的应用程序的运行平台

**三者全部相同（build = host = target）的就是native compiler**，例如我们在PC上装的Ubuntu或者Fedora里面带的GCC，就是native compiler。

如果build = host，但是target跟前两者不同，就是cross compiler。比如开发手机应用程序的编译器，通常运行在PC或Mac上，但是编译出来的程序无法直接在PC或Mac上执行。

三者都不同的话，叫做Canadian cross。（问：跟Canadian有什么关系？答：参见[http://en.wikipedia.org/wiki/Canadian_cross#Canadian_Cross](http://en.wikipedia.org/wiki/Canadian_cross#Canadian_Cross)）

其它的如：host = target，但是build不同，叫做crossed native；build = target，但是host不同，叫做crossback。