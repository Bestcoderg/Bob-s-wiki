# Unity 基础

## Unity 三种模式

Unity 编辑器界面有三种模式:

- **Play Mode**： 按下 Play 按钮进入游戏的这种模式，Unity 称之为 Play Mode
- **Edit Mode**：正常的 Unity 编辑状态就是 Edit Mode
- **Prefab Mode**：Unity 进入 prefab 时对应的编辑状态叫 Prefab Mode

Unity 里继承 MonoBehaviour 的脚本，并不是一直都会执行的。Unity 默认只有在 Play Mode 下，才会运行游戏当前运行场景里的 GameObject 挂载的脚本。

为了满足实际需求，Unity 支持通过[**ExecuteInEditMode**]或[**ExecuteAlways**]两种参数使脚本在 Play Mode 以外的状态下被执行，[**ExecuteEditMode**]支持脚本在 Edit Mode 下运行，[ExecuteAlways]是在 Unity2018.3 及以后的版本新加入的功能，能够支持脚本一直运行。（ps: 由于[**ExecuteInEditMode**] 并没有考虑 Prefab Mode，严格意义上讲 Prefab Mode 也属于 Edit Mode，所以这个功能会逐渐被 Unity 弃用，最后应该会被[**ExecuteAlways**]所替代）

Unity保证第二次进入playmode时，前面一次的类的内存是被卸载的（当然也有可能不是卸载，而是加载的时候另起了一篇内存） 总的来说 ，静态构造函数在再次进入palymode时会被调用。卸载的时机是不确定的。

### **ExecuteInEditMode**

值得注意的是，与 PlayMode 不同的是，函数并不会不停的执行。

- `Awake`和`Start`：加载时调用，也就是脚本赋给物体的时候被调用
- `Update` : 只有当场景中的某个物体发生变化时，才调用，当进程切出去再回来，也会调用一次。
- `OnGUI` : 当 GameView 接收到一个 Event 时才调用。
- `OnRenderObject` 和其他的渲染回调函数 : SceneView 或者 GameView 重绘时，比如，一直移动鼠标的时候`OnRenderObject`会被调用。

## 什么是.meta文件

.meta 文件中包含了两项重要的部分，guid 和资源的 import settings；

### **一、GUID**

资源导入时，unity 会自动为该资源生成一个 guid(Global Unique Identifier)；同时一个相同文件名（包含后缀）的. meta 文件将会被创建，例如 data.json 对应 data.json.meta；guid 是. meta 中最重要的数据，这个 guid 代表了这个文件，无论这个文件是什么类型（甚至是文件夹）。换句话说，通过 GUID 就可以找到工程中的这个文件，无论它在项目的什么位置，但是如果. meta 的 guid 被修改了，其它资源对这个资源的引用就会失效；在编辑器中使用 AssetDatabase.GUIDToAssetPath 和 AssetDatabase.AssetPathToGUID 进行互转。**注意：.meta 文件的文件名和文件是相同的，而且必须和相应的资源文件在同一目录下；如果在 unity 编辑器内移动或者重命名了一个资源文件，unity 会自动将相应的. meta 文件重命名或者移动到对应目录；但是如果在编辑器外移动或者重命名了一个资源，那么当你重新进入编辑器时，unity 编辑器就会把这个资源当作一个新的资源，并为其重新生成一个新的. meta 文件并删除之前的. meta 文件 (除非你在移动或者重命名文件时，手动将对应的. meta 文件移动或者修改)；重新生成会导致 Guid 的重新生成，从而导致其它对象对该资源的引用失效；**如果在工程中找不到. meta 文件，可能是被设置了隐藏，可以打开 window 的查看隐藏文件或者在工程设置内设置不要隐藏. meta 文件；

注意：如果打开了两个 unity 编辑器，直接从一个编辑器向另一个编辑器导入资源，也会重新生成 meta 文件的；

### **二、Import Settings**

meta 文件存储的**两个重要信息**，一个是 guid，另一个就是 import settings（可以在属性面板中查看），修改 import settings 也会导致 meta 文件的修改，但是并不影响引用；

导入资源将会被处理，unity 会将资源转换成其内置的 game-ready 版本，**原始的资源会被放置在 assets 文件夹内，处理和转换后的数据会被放置在 Library 文件夹内**；作为开发者，并不需要关心 Library 文件夹，它的所有数据都是从 Assets 和 ProjectSettings 文件夹中存储的内容生成的，这也意味着 Library 文件夹不应包含在版本控制中；在项目备份时，应包含 Unity 项目文件夹中的 Assets 和 ProjectSettings 文件夹，而应该忽略 Library 和 Temp 文件夹；.meta 文件还包含资源的导入设置，在属性面板中可以查看和修改导入配置；如果更改了导入设置，那么这些更改会被存储在相应的. meta 文件中，而且会在项目的 Library 文件夹中更新相应的 game-ready 数据；

## Unity 崩溃日志目录

1，一般日志路径：C:/Users/xxxx/ AppData/Local/Unity/Editor，此文件夹下有三个文件 ，如下图：Editor.log, Editor-prev.log, upm.log

2，崩溃日志路径：**这个日志是unity编辑器崩溃时才生成的，没崩溃过则没有**。而且，个人版的unity没有的。

崩溃日志的路径大概是这样子的，：C:/User/xxxx/**AppData/local/Temp/Unity/Editor/Crashes/Crash_2019-06-30_125823955**

## Editor目录与非Editor目录

unity在添加脚本后会自动生成VS的解决方案的程序集，如图所示。一般直接添加非Editor目录的.cs脚本会被自动加入 Assembly-CSharp 这个解决方案下，而Editor目录下的.cs脚本则会被添加到 Assembly-CSharp-Editor 解决方案下。这两者的脚本文件更改并在VS保存后都会被自动编译为 .dll 文件。非Editor目录下的脚本无法调用Editor目录下的定义或是声明。如果将脚本挪到自己的解决方案下则不会被自动编译，如图中的 MoonClient 。

![Untitled](Unity%20%E5%9F%BA%E7%A1%80%20a0f9d2a03e60429cb13cdc16fecfdb8b/Untitled.png)