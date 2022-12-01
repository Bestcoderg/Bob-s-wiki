# 什么是 vim/nvim

## NeoVim

如果您是系统管理员或软件开发人员，那么你每天都需要使用的工具中一定有一种强健的文本编辑器。您很可能已经使用过vi或vim编辑器，它们已经在Unix和Linux社区中用了几十年了。

虽然vim仍然是处于活跃开发的状态，但它已经包含了大约30万行的[C89](https://en.wikipedia.org/wiki/ANSI_C#C89)编码代码。因为Bram Moolenaar是唯一维护其大型代码库的人了，所以现在的vim除了难以维护之外，其问题和新的代码合并取请求都无法很快解决。

由于这些难题和缺乏对如异步插件等所需功能的支持，这促使NeoVim成为vim的一个分支。该项目的主要目标是完全重构vim，以便简化维护，并且实现快速添加新特性并将bug修复添加到源代码中。

neovim的特点：

1、重构vim代码库，保留vim的操作模式和编辑方法和思想

2、基本放弃对旧系统的支持

3、提供适用现代系统的默认设置

4、提供丰富的插件开发，支持与外部程序的通信，提供python和lua脚本支持

## VIM-LSP

### 什么是LSP？

LSP（Language Server Protocol）是由 redhat, microsoft, codenvy 联合推出的**语言服务器协议**，于**编辑器**和**语言服务器**之间的通信。**LSP 是一套通信协议**，遵从 LSP 规范的客户端（各种编辑器/IDE）可以通过众多 LSP 服务端按协议标准进行通信，由客户端完成用户界面相关的事情，由服务端提编程语言相关的：补全，定义引用查找，诊断，帮助文档，重构等服务。

我们知道市面上有很多种**不同的**编辑器，VSCode，IntelliJ IDEA，Emacs，Vim，……每一个编辑器也能支持多种**不同的**编程语言，Java，JavaScript，Python，Ruby，……

LSP 协议就是一套解偶合的标准，比如 C/C++ 补全，以前是 NotePad++ 里做一套，VS里做一套，Vim里做一套，Emacs里又做一套，C/C++ 还好，但是其他语言就惨了，各个编辑器实现的质量参差不齐。LSP 出现后能让编辑器专心提供最好的用户体验，而 LSP Server 则专心将具体语言的各种集成服务做透，做扎实。LSP还负责自动完成（auto complete），转到定义（go to definition），鼠标悬停展示文档（documentation on hover），……

![%E4%BB%80%E4%B9%88%E6%98%AF%20vim%20nvim%20609f7ff074ce4832bb63809f98fbdeb8/Untitled.png](%E4%BB%80%E4%B9%88%E6%98%AF%20vim%20nvim%20609f7ff074ce4832bb63809f98fbdeb8/Untitled.png)

这样同一个 LSP Server 可以服务于不同的编辑器 / IDE，不同的编辑器/IDE 一旦支持 LSP，则可以瞬间对接高质量的各种 LSP Servers。

### LSP 服务端和客户端怎么通信的？

说起 LSP 服务端大家不要误会成 http server 那样的常驻后台服务，好像编辑器只有通过 TCP发送请求上来才行。实际上，LSP Servers 都是一个个命令行程序，由编辑器（也就是客户端）启动，通过管道发送 **JSON RPC** 命令同 LSP Server 交流，退出编辑器，LSP 服务端也就关闭了。

### 有哪些 C/C++ 的 LSP 服务端？

目前 C/C++ 的 LSP Server 实现：

[clangd](https://link.zhihu.com/?target=https%3A//clang.llvm.org/extra/clangd.html)：目前看起来最稳定健壮，但是尚无索引系统，不支持多 Translation Units，所以可以补全，查找定义，但是无法搜索引用。

[cquery](https://link.zhihu.com/?target=https%3A//github.com/cquery-project/cquery)：支持多 Translation Units，可以搜索引用，性能不错，但是稳定性不如 clangd。

[ccls](https://link.zhihu.com/?target=https%3A//github.com/MaskRay/ccls)：cquery 的 fork，据说解决了一些 cquery 的痛点。

以下是vim使用中用到的一些LSP Client插件：

**1. LanguageClient-neovim** 异步，python插件，支持大部分language server操作，支持deoplete和ncm2两个补全框架

**2. vim-lsp** 异步，纯vim script插件，支持asyncomplete, deoplete和ncm2三个补全框架，部分lsp支持有问题

**3. coc.nvim** 8012年的黑马。异步，nodejs后端，支持全部language server操作，支持coc.nvim本身的补全框架

### 用 LSP 进行符号搜索同 gtags 的比较

前面的 gtags 也能搜索定义和引用，但是 gtags 基于静态符号分析，无法真的像编译器一样给出绝对准确的定义和引用，比如同一个函数有多个定义，根据宏不同条件编译的结果也不同，gtags 搜索定义会把他们全部搜索出来，而 cquery 却能准确的找到真的定义是哪个。

再比如搜索某对象的方法名，gtags 并不能知道该对象的类型，只能按照符号名搜索出一大堆的有可能不同类的同名方法出来，或者名字相同但是参数不同的一堆方法来，虽然这类情况不多，但搜索出来需要你人工二次过滤一下。cquery 就没有这个问题，基于 clang 的语义分析，真的能够定位到你调用的是什么方法来，特别是大型项目，能帮助你省掉不少时间。

但是，还是那句老话，整套 LSP 安装极度麻烦，支持的语言目前也不算多，除了安装 LCN 以外，你用一个语言就要安装那个语言的 LSP 服务端，还要保持日常更新，而 gtags 只需要配置一次，就能通杀 50 种语言。

所以我是同时使用 gtags 和 LSP 的，而 LSP 我也只使用了 cquery，碰到 C/C++ 项目的话我会用 LCN 搜索符号，碰到其他项目或者远程机器上没有配置 LSP ，我就用 gtags。而 cquery 巨费内存，相对于超大型 C/C++ 项目，我还是会切换回 gtags，比如 linux kernel。

cquery有点好处是你不在项目根目录建立 .cquery 文件，它就不会工作，所以我一般默认情况下先使用 gtags，碰到要花时间的项目我再建立这个文件把 cquery 运行起来。

所以没有说一定要用哪个代替哪个的关系，你可以把 gtags 作为一个保底方案先配置起来，一口气为 50 种语言提供符号索引，然后再针对日常用的多的开发语言配置一两个 LSP 服务端，然后两个看心情换着用。

## Vim-Plug

Vim-plug 是一个自由、开源、速度非常快的、极简的 vim 插件管理器。它可以并行地安装或更新插件。你还可以回滚更新。它创建浅层克隆 shallow clone 最小化磁盘空间使用和下载时间。它支持按需加载插件以加快启动时间。其他值得注意的特性是支持分支 / 标签 / 提交、post-update 钩子、支持外部管理的插件等。

```cpp
//例如，我们安装 “lightline.vim” 插件。为此，请在 ~/.vimrc(nvim在~/.config/nvim/init.vim) 的顶部添加以下行。
call plug#begin('~/.vim/plugged')
Plug 'itchyny/lightline.vim'
call plug#end()

//在 vim 配置文件中,在vim中使用以下命令检查状态：
:PlugStatus
//然后输入下面的命令，然后按回车键安装之前在配置文件中声明的插件。
:PlugInstall
//更新插件
```

## 补全框架

补全框架会根据你当前的输入提供补全建议，撸代码必备。补全框架的补全资源的来源大致有三种：

1. 来自 LS
2. 来自额外的补全资源，如 jedi, racer
3. 来自代码片段（snippets）

**[neoclide/coc.nvim**github.com](https://link.zhihu.com/?target=https%3A//github.com/neoclide/coc.nvim)

异步，node.js 后端，使用 coc.nvim 本身的 LSC。

coc 把 vscode 做得最好的那一部分带到了 vim，它支持很多非常前沿的特性，并且补全速度非常快。如果你还在纠结选哪个的话，建议直接上 coc。

**[ncm2/ncm2**github.com](https://link.zhihu.com/?target=https%3A//github.com/ncm2/ncm2)

异步，python 后端，支持 LanguageClient-neovim 和 vim-lsp 两个 LSC，支持很多额外补全资源。

**[Shougo/deoplete.nvim**github.com](https://link.zhihu.com/?target=https%3A//github.com/Shougo/deoplete.nvim)

异步，python 后端，支持 LanguageClient-neovim 和 vim-lsp 两个 LSC，支持的额外补全资源数量最多，但是补全速度非常慢，文档也很复杂。

**[prabirshrestha/asyncomplete.vim**github.com](https://link.zhihu.com/?target=https%3A//github.com/prabirshrestha/asyncomplete.vim)

异步，纯 vim script 后端，支持 vim-lsp，支持的额外补全资源数量较少，但是不依赖 python。

YCM ( [Valloric/YouCompleteMe](https://link.zhihu.com/?target=https%3A//github.com/Valloric/YouCompleteMe) )和上面四个插件的工作模式不太一样，上面四个是补全框架 + LSC + 额外补全资源 + 代码片段，而YCM的哲学是 "All in one"。YCM 虽然已经支持异步了，但是需要编译，而且只对 C 系的语言补全体验较好，而上面四个支持其它很多语言的补全。

## 模糊匹配插件

vim 里的模糊匹配插件可不是只能进行简单的搜索任务，你可以用这些插件来：搜索文本、对 grep 结果进行过滤、浏览和跳转 tags/symbols（就是函数和变量之类的）、浏览 buffer、浏览文件、跳转最近使用的 buffer 或文件(Most Recently Used, MRU)、浏览历史剪切板内容、切换 color scheme、甚至浏览你的 github 星标版本库。而这些模糊匹配的内容就是所谓的 source。

**[Yggdroot/LeaderF**github.com](https://link.zhihu.com/?target=https%3A//github.com/Yggdroot/LeaderF)

异步，依赖 python。它的算法非常厉害，如果按照匹配精准度和匹配速度来衡量模糊匹配插件的性能的话，它是这几个插件里性能最强的。