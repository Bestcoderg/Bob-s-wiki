# Unity JobSystem

## Job System

Unity 的 C# Job System 可以让你在 Unity 中编写简单且线程安全的多线程代码以提高游戏性能。

你可以将 C# Job System 和 Entity Component System (ECS) 结合使用，这种架构可以为所有平台创建高效的代码。ECS 将于今年晚些时候发布（预计 2018.3）。

### Job System 的目的

如果你在 Unity 中使用过多线程就会知道：Unity 中，Unity API 必须在主线程中使用，无法在子线程调用。

Job System 旨在让用户能编写与 Unity API 交互的多线程代码，同时使编写正确的代码更加容易。

### 什么是多线程？

当大多数人想到多线程时，他们会想到创建线程来运行代码，然后以某种方式与主线程同步结果。如果你有几项长时间运行的任务，这很有效。

但是在游戏中，这种[并行计算](https://en.wikipedia.org/wiki/Parallel_computing)的情况很少出现。通常你有很多很小的任务，但是如果你为它们创建了线程，那么你将会得到大量寿命很短的线程。

当然了，你可以通过建立一个[线程池](https://en.wikipedia.org/wiki/Thread_pool)来缓解线程生存期的问题。但即使你这样做，你也会同时拥有大量的线程。拥有比 CPU 内核更多的线程会导致线程相互争夺 CPU 资源，从而导致频繁的[上下文切换](https://en.wikipedia.org/wiki/Context_switch)。

当 CPU 执行上下文切换时，需要做大量工作来确保新线程的状态是正确的，这可是相当耗费资源的，应尽可能避免。

### 什么是 Job System（作业系统）？

Job System 以稍微不同的方式解决了并行的任务，通过创建 [Job（作业）](https://en.wikipedia.org/wiki/Job_(computing))而不是线程。Job 类似于函数调用，包括所需的参数和数据，**所有这些都放入 [job queue（作业队列）](https://en.wikipedia.org/wiki/Job_queue)中执行。Job 应该足够小，并只做一件特定的事情**。

Job System 有一组工作线程，通常每个 CPU 核心对应一个线程以避免上下文切换。工作线程从作业队列获取作业并执行。

如果每个 job 都是独立的，这就已经够了。然而在更复杂的系统中，所有的 job 完全独立几乎不可能，因为这会导致大量 job 做很多事情。所以通常前一个 job 会为下一个 job 准备数据。

为了让 **job 使用更容易，作业支持 [dependencies（依赖）](http://tutorials.jenkov.com/ood/understanding-dependencies.html)。如果作业 A 依赖作业 B，系统将保证在作业 A 开始执行之前作业 B 已完成。**

C# Job System 的一个重要方面是 Unity 引擎内部使用了 job system，这也是用自定义 API 而不是 C# 中现有的线程模型的原因。这意味着用户编写的代码和引擎会共享工作线程以避免创建比 CPU 核更多的线程——这会导致 CPU 资源争夺。

### Race conditions & safety system（竞争和安全系统）

在编写多线程代码时，总是存在 [Race condition（竞争）](https://en.wikipedia.org/wiki/Race_condition)的风险。竞争意味着某些操作的输出取决于某些无法控制的其他操作的时序。当写数据的同时其他代码要读这些数据，就会出现竞争状态。读到的数据取决于数据在读之前还是读之后执行，读取无法控制。

竞争条件并不总是 bug，但它始终是不确定行为的来源，当它确实导致 bug，如崩溃，死锁或不正确的输出时，可能很难找到问题的根源，因为它取决于时间顺序。这意味着该问题只能在极少数情况下重新创建，调试可能导致问题消失; 因为断点和日志也改变了时序。

在很大程度上，这使得编写多线程代码变得困难，但是不要害怕——已经有解决办法了。

为了更容易编写多线程代码，Unity 中的作业系统旨在检测所有潜在的竞争条件并保护你免受他们可能导致的错误。

实现这一目标的主要方式是确保作业仅对传递给它的所有数据的副本进行操作。如果没有人能够访问作业所运行的数据，那么它不可能导致竞争状况。以这种方式复制数据意味着作业只能访问 [blittable](https://en.wikipedia.org/wiki/Blittable_types) 数据，而不是 [Managed（托管）](https://en.wikipedia.org/wiki/Managed_code)类型。这会导致很大限制，因为你不能从任务中返回任何结果。

为了能够编写代码来解决真实世界的场景，复制数据的规则有一个例外。该例外是**NativeContainers**。Unity 附带了一组 NativeContainer（原生容器）：**[NativeArray](https://docs.unity3d.com/2018.1/Documentation/ScriptReference/Unity.Collections.NativeArray_1.html)**，**NativeList**，**NativeHashMap** 和 **NativeQueue**。所有原生容器受 safety system（安全系统）保护。Unity 跟踪所有正在写入或读取容器的对象。

例如，准备写入同一个 NativeArray 的两个作业，safety system 会抛出一个异常，并显示明确的错误消息，说明为什么以及如何解决问题。在这种情况下，你始终可以使用依赖项安排作业，以便第一个作业可以写入容器，一旦它执行完毕，下一个作业可以安全地读取并写入该容器。当然允许多个作业并行地读取相同的数据。从主线程访问数据的读写限制相同。

一些容器还具有特殊的规则，允许从 **ParallelFor** 作业进行安全且确定的写入访问。例如，**NativeHashMap.Concurrent** 允许你从 **IJobParallelFor** 中并行添加 item。