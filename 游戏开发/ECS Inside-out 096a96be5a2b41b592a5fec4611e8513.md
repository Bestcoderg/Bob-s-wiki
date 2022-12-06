# ECS Inside-out

## ECS基本概念

**ECS架构开发的游戏基本结构图**

![ECS%20Inside-out%20096a96be5a2b41b592a5fec4611e8513/Untitled.png](ECS%20Inside-out%20096a96be5a2b41b592a5fec4611e8513/Untitled.png)

先有一个World，它是系统和实体的集合，而实体就是一个ID，这个ID对应了组件的集合。组件用来存储游戏状态并且没有任何行为，系统拥有处理实体的行为但是没有状态。

### **Entity Component System 的定义**

**实体**

- 实体是通用对象。通常，它仅包含唯一的ID。
- **实体仅由一个ID和一个组件容器组成。**
- 它将每个粗糙的游戏对象标记为一个单独的项目，通常使用普通整数实现。
- 实体只是一个概念上的定义，指的是存在与游戏世界中的一个独特物体，是一系列组件的集合。**为了方便区分不同的实体，在代码层面上一般用一个ID来进行表示。所有组成这个实体的组件将会被这个ID标记**，从而明确哪些组件属于该实体。由于其是一系列组件的集合，因此完全可以在运行时动态地为实体增加一个新的组件或是将组件从实体中移除。比如，玩家实体因为某些原因（可能陷入昏迷）而丧失了移动能力，只需简单地将移动组件从该实体身上移除，便可以达到无法移动的效果了。

**组件**

- 对象某一方面的原始数据，以及它与世界的交互方式。 用于标记实体具有的特定方面，实现通常使用结构，类或关联数组等。
- 一个组件是一堆数据的集合，可以使用C语言中的结构体来进行实现。它没有方法，即不存在任何的行为，只用来存储状态。一个经典的实现是：每一个组件都继承（或实现）同一个基类（或接口），通过这样的方法，我们能够非常方便地在运行时动态添加、识别、移除组件。每一个组件的意义在于描述实体的某一个特性。例如，PositionComponent（位置组件），其拥有x、y两个数据，用来描述实体的位置信息，拥有PositionComponent的实体便可以说在游戏世界中拥有了一席之地。当组件们单独存在的时候，实际上是没有什么意义的，但是当多个组件通过系统的方式组织在一起，才能发挥出真正的力量。同时，我们还可以用空组件（不含任何数据的组件）对实体进行标记，从而在运行时动态地识别它。如，EnemyComponent这个组件可以不含有任何数据，拥有该组件的实体被标记为“敌人”。
- 组件需要带上基础的方法，这样可以保证类的封装性

**系统**

- 对拥有与该系统相同方面的组件的每个实体执行全局操作。
- 系统便是ECS架构中用来处理游戏逻辑的部分。何为系统，一个系统就是对拥有一个或多个相同组件的实体集合进行操作的工具，它只有行为，没有状态，即不应该存放任何数据。举个例子，游戏中玩家要操作对应的角色进行移动，由上面两部分可知，角色是一个实体，其拥有位置和速度组件，那么怎么根据实体拥有的速度去刷新其位置呢，MoveSystem（移动系统）登场，它可以得到所有拥有位置和速度组件的实体集合，遍历这个集合，根据每一个实体拥有的速度值和物理引擎去计算该实体应该所处的位置，并刷新该实体位置组件的值，至此，完成了玩家操控的角色移动了。

## ECS核心概念

通过前几节的学习咱们已经知道了 ECS 包含三个部分：**实体（entities）、数据（components）、行为（system）**。**ECS 架构的核心是数据**，这也是为什么 Unity 会将这一套技术栈命名为 DOTS。System 会读取 entity 上面的 component 的数据流，并处理数据。**实体在这里其实更像是索引，它本身并不包含任何数据和逻辑。**

具体看下图：

![http://upload-images.jianshu.io/upload_images/78733-b51d465347468a1e.png](http://upload-images.jianshu.io/upload_images/78733-b51d465347468a1e.png)

这个图中，System 读取了多个实体的`Translation`和`Rotation`组件，然后经过计算处理，将结果更新到`LocalToWorld`组件中。

从图中你可以看到，实体 A 和 B 还有 Renderer 组件，但是 C 并没有。不过这并不会影响 System 的计算逻辑，因为这个系统不关心`Renderer`组件。

你还可以写一个系统，需要处理 Renderer 组件，这样系统就会忽略实体 C。你还可以写一个系统排除包含 Renderer 组件的实体，这样系统就会忽略实体 A 和 B。

下面对 ECS 中比较重要的几个核心概念做一个梳理：**（以下内容皆是基于UnityECS）**

[ECS concepts](https://docs.unity3d.com/Packages/com.unity.entities@0.1/manual/ecs_core.html)

### 原型 Archetypes

**多个组件的组合叫做一个原型，类比面向对象的概念中实体更像是对象而原型更像是类。**比方说一个人需要位置、朝向、背包三个组件，所有人都需要这三个组件，那么这三个组件的结合方法就是一种原型Archetypes。

比如一个 3D 物体可能会包含用于 transform 的组件，包括移动、旋转、渲染，每个 3D 物体对应一个实体，但是他们都有同样的组件，所以 ECS 会把他们分类成是一类原型。

![http://upload-images.jianshu.io/upload_images/78733-cccd2566347cf5cb.png](http://upload-images.jianshu.io/upload_images/78733-cccd2566347cf5cb.png)

在上图中，实体 A 和 B 的原型都是 M，实体 C 的原型是 N。

你也可以通过在运行时添加或者移除 component 来改变一个实体的原型。例如：如果将实体 B 的 Renderer 组件移除，实体 B 的原型就会变成 N。

### 内存块 Memory Chunks

为什么要先讲原型的概念呢，因为一个实体的原型是什么，决定了 ECS 会将实体的 components 也就是数据存在什么地方。**ECS 按块分配内存，每块用一个`ArchetypeChunk`对象表示。**

一个块只包含一种原型，可以包含的多个实体的数据。如果一个块的内存满了，ECS 会分配一个新的块来存储新的实体的 components。

**如果你修改了实体的组件，那就相当于修改了实体的原型，这时候 ECS 会将实体的组件数据移到另外一个块中**。

Memory Chunk解决的痛点：1、当你想找到匹配要求的Enitity时，不需要再遍历Entity列表了，只需要找到相关的Archetypes，然后遍历Memory Chunk中的Entity就可以了。

![http://upload-images.jianshu.io/upload_images/78733-7ee4fc49f366ef53.png](http://upload-images.jianshu.io/upload_images/78733-7ee4fc49f366ef53.png)

原型和内存块的关系是一对多的关系。这就意味着，如果想查询给定的一组 component 类型的所有实体，只需要在这些原型中搜索即可。这样会比在所有的实体中**查找效率高**很多。

ECS 在存储实体到内存块中没有特殊的排序，当创建一个实体或者实体的原型发生变化时，ECS 会将它放到对应原型的第一个还有空间的内存块中。**内存块中的数据会紧密排列。如果一个实体要被移出当前原型的内存块，这时候会有个空位，ECS 会把这个内存块最后的实体数据移动到这个空位中。**

**注意**：原型中的**共享组件**（后面会具体说这是个什么东东）的数据也会影响实体会被存在哪个内存块。同一个内存块中的所有实体的共享组件中的数据值都是相同的。如果你修改了共享组件中的数据，这个实体会被移到另外一个块中，有点类似修改了实体的原型。

将共享组件的实体分到一个内存块中会提高处理他们的速度。比如 Hybird Renderer（混合渲染）定义了 RenderMesh 组件来达成这个目的。

**ECS这个系统具有很大的优势：它不仅可以提高缓存效率，缩短访问时间；它还支持在需要使用这种数据对齐方式的现代 CPU 中采用先进技术（自动矢量化 / SIMD）这为游戏提供了所需的默认性能**

### 实体查询

一个 System 根据什么来决定处理哪些实体呢？这时候会用到一个叫**实体查询 (Entity Query)** 的东西。实体查询首先需要一些组件类型，然后根据你传入的组件类型的组合，在包含这些组件的原型中查询符合要求的实体。查询时可以指定下面三种类型：

- **All** 必须包含 All 中所有的组件类型
- **Any** 必须包含 Any 中至少一个组件类型
- **None** 不能包含 None 中任意一个组件类型

一次实体查询的结果会返回所有符合查询要求的内存块，你可以使用`IJobChunk`来迭代遍历所有的组件。（IJobChunk 后面会讲。）

我认为这种实体查询的方式就是三种类型查询需求的一种Unity的封装。

### Jobs 作业

ECS 配合 Job 使用才能发挥多线程的威力。ECS 提供了`SystemBase`类，其中包含`Entities.ForEach`方法，还包含了`IJobChunk`的`Schedule()`和`ScheduleParallel()`方法，可以在子线程中处理数据。`Entities.ForEach`是最简单的方法，只需要几行代码就能实现。`IJobChunk`可以用来处理比较复杂的情况。

ECS 会在主线程调度 Job，根据 System 的顺序。当 job 调度后，ECS 会追踪哪些 job 在读写哪些组件。需要读权限的 job 需要等待前面写权限的 job 执行完，反之亦然。Job 调度器会使用 job 依赖来决定哪些 job 可以并行，哪些必须串行。

[Unity JobSystem](Unity%20JobSystem%2060acfa631e8549cfada0e37ba14b6535.md) 

### System 的组织

ECS 通过`World`和`group`来组织 system。默认情况下，ECS 会创建一个默认的 World，包含一些预定义的 group 组。它会找到工程中所有的 System，实例化他们，并添加到预定义的 group 中。

你可以指定同一个 group 中 system 的 Update 的执行顺序。Group 也是一种 system，所以你可以将一个 group 添加到另外一个 group 中。如果你没有指定顺序，system 的执行顺序会不太确定，并不会按照它们创建的顺序。不过，同一个 group 中的所有 system 都会比下一个 group 中的 system 先执行。

**System 的 Update 是在主线程中执行的，不过可以使用 Job 将工作分配到子线程中**。

## 详解Components

在UnityECS中，component实际上有五个基类或是说继承了五种接口。我们可以当作UnityECS中有五种类型的conponent，分别是IComponentData、ISharedComponentData、ISystemStateComponentData、ISharedSystemStateComponentData、DynamicBufferComponent。

### 普通组件数据 GeneralComponentData

从定义和思想上来说，普通组件是指仅仅包含数据的一个结构体，且其不能包含方法。

但是从我自己的实践上来说，在处理组件的数据上是会有很多共通的方法，所以目前我处理这个问题的方式是将组件逻辑化，其实就是将单个组件内的方式放在组件内，其实是一种面向对象的方式来解决的。

### 共享组件数据 SharedComponentData

共享组件（SharedComponentData）是一种特殊的数据组件，您可以使用它来根据共享组件中的具体值来细分实体（它们的原型除外）。当你将共享组件添加到实体时，EntityManager 将具有相同共享数据值的所有实体放入同一个 Chunk 中。当然，这边说的将具有相同SharedComponentData的实体放入相同Chunk是在单个原型所属Chunk下的操作，对于一个Archetype原型来说，所有的实体依旧属于其对应的原型下的Chunk的。

举一个例子，对于一个拥有某个SharedComponentData组件的Archetype A，其Chunk在内存中的组织形式如下图左侧。从图中可以看出Entity A~E具有相同的Shared Component data，而F~G是相同的，现在我们将Entity E的共享组件改变到与F~G一致后，此时则需要将Entity E从chunk2移动到chunk3。

**其实关于共享组件，我认为其主要是在Archetype之下，对于实体进行了进一步的分类管理。同时，在原型层面，我们选择的是静态式的分类方式，某个原型有什么组件在注册时就已经固定了。而在进一步的细分中，我们采取的是一种动态的方式，通过共享组件来进一步做实体的分类管理。**

所以说，共享组件总的来说有两点好处：一是能够节约空间，对于同一具有相同Shared Component Data 数据值的多个实体将被组合存放在同个一块(Chunk)中，SharedComponentData 的索引将被一次性存储在此 Chunk 中，而不是为每个实体各存储一次。第二点是能够对实体再做一层细分，对于具有相同的实体在内存上组织在一起，具有相对效率较高的操作效率。尤其是在查询、遍历这一类操作中，这种内存组织形式显得非常的搞笑。

同时需要注意的是过度使用共享组件会导致较差的块利用率，因为它是根据原型和每个共享组件的字段的每个特定值，来组合扩展所需的内存块数。

![ECS%20Inside-out%20096a96be5a2b41b592a5fec4611e8513/Untitled%201.png](ECS%20Inside-out%20096a96be5a2b41b592a5fec4611e8513/Untitled%201.png)

### 系统组件 SystemStateComponentData

所谓“系统”，指的并不是操作系统或是程序系统，而是ECS中的S，即System。SystemStateComponent从名字我们就可以看出，这是一种指示system工作时某些状态的数据。设计这种组件目的其实就是因为system在工作时，需要的一些状态是由Entity实体的数据来决定的，所以需要在设计一种组件来存储system的运行时所需的状态。SystemStateSharedComponent&SystemStateComponent的关系则与普通组件&共享数据组件的关系基本一致。

### 动态缓冲区组件 DynamicBufferComponent

我们前所说的chunk是一种管理conponent的良好方式，但是在使用中我们会发现，如果希望使用chunk来管理component，那么每个component的大小都要一致，这意味着我们无法动态的向component中加入一些数据，或是说需要开一个容量较大的静态数组来保存的数据。这显然不是一种优雅的方式，UnityECS解决这个问题的方案是引入了动态缓冲区组件，用法类似于模板的方式，需要先指定缓存区中所存数据的类型。这样我们就拥有了向component中动态加入数据的手段。

UntityECS中这个组件作为“弹性”的缓冲区，容器内部允许承载一定数量的单元，如果内部容量不足时，会再重新分配堆内存。使用此缓冲区时，内存的管理是完全自动的。与DynamicBuffers 相关联的内存由EntityManager 管理，以便在删除 DynamicBuffer 组件时，这些引用的堆内存也会自动释放。

## 详解System

终于，来到了UnityECS中的S（System）部分。UnityECS同样提供了多种类型的system，这些system在用户的使用过程中，只会使用到ComponentSystem&JobComponentSystem，其他的基本上是一些特化的system，为其他的system提供支持。

### ComponentSystem

这是system最常使用也是逻辑部分的核心，主要负责对实体进行操作，理论上所有组件间的逻辑都是在system中实现的。ComponentSystem仅包含方法，不能包含数据实例。

### JobComponentSystem

前面已经介绍了Unity的Job作业系统，实则是一个封装且易用的多线程系统，只不过使用job的形式来展开。JobComponentSystem目的是管理各个组件间的依赖关系，为多个job之间的依赖管理提供支持。

JobComponentSystem的具体工作形式如下：所有作业和系统都会声明它们读取或写入的 ComponentTypes。 因此，当 JobComponentSystem 返回 JobHandle 时，它会自动向EntityManager注册，包括所有的组件类型以及相关的读写的信息（如果一个系统写入组件 A，而稍后另一个系统从组件 A 读取，则 JobComponentSystem 会查看它正在读取的组件类型列表，从而向您传递该作业对第一个系统的依赖关系）

**JobComponentSystem 根据需要，为各个作业串联出各方的依赖关系，因此不会产生主线程上的停顿**。但是，如果非作业系统访问相同的数据会发生什么？ 由于声明了所有访问权限，因此在调用 OnUpdate 之前，ComponentSystem 会自动完成针对系统使用的组件类型运行的所有作业 （非作业系统运行在主线程，主线程会因此卡住）

## UnityECS & JobSystem

以下几个内容皆是Unity为了ECS+Job所做的一些策略，由于比较杂就干脆都写在一起了。

### Unity Entity Command Buffer

实体命令缓冲从字面意义上就可以看出是一个存储对实体操作命令的一个缓存，实际上和消息队列的缓存yi'ge'yi'si，实际上解决了两个重要的问题：

1. 当在 Job 中时，无法访问`EntityManager`
2. 当您访问`EntityManager` （比如说，修改一个实体）时，这将使所有与该实体关联的数据数组和 EntityQuery 对象无效

实际上把实体命令缓存（EntityCommandBuffer)这个概念抽象出来，为了是将对实体或者组件进行的修改（来自作业或主线程）进行队列缓存，以便它们后续可以在主线程上得以执行。

### System Update Order 系统更新顺序

system在Unity中的更新顺序的管理策略，具体略。。。