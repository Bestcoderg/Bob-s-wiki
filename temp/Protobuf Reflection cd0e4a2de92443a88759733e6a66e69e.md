# Protobuf Reflection

## 什么是反射

```cpp
既然我们可以构造 “有关某个外部世界表示” 的计算过程(程序), 并通过它来对那个外部世界进行推理; 那么我们也可以构造能够对自身表示和计算进行推理的计算过程 , 它包含负责管理有关自身的操作和结构表示的内部过程。
—— 1982 年 Smith, Brian Cantwell 博士论文首次提出
```

从某种角度来看，所谓编程实际上就是在构造 “关于外部世界” 的计算过程。如果用 F 表示这个构造过程，用 X 表示外部世界，那么编写一个计算系统可表示为 F(X)。**我们完全可以构造对上述过程本身进行描述和推理的计算过程。即将 F(X) 视为新的 “世界” 和 研究对象，构造 F(F(X))**。

我们平时编写的计算系统是面向特定领域的（通常是面向现实建模）, 系统中包含 `用来描述领域中的实体` 和 `实体间关系`的数据结构以及处理这些数据结构的规则。那么反射系统面向领域便是这个系统本身。

**所以我们对于反射的定义可以概括为 `计算机程序在运行时可以访问、检测和修改它本身状态或行为`。**

### 如何实现反射系统？

对于程序而言如何实现反射？简单来说可以概括为 **获取系统元信息(元数据)** 。实际上我们在 [CLR 公共语言运行时](CLR%20%E5%85%AC%E5%85%B1%E8%AF%AD%E8%A8%80%E8%BF%90%E8%A1%8C%E6%97%B6%20046cf021c5064abab630b4623ed67a8e.md) 提到过元数据的概念，元信息即系统的子描述信息，用于描述系统自身的信息。CLR在将 IL 编译为可执行文件的过程中便会生成用于描述自身系统信息的 元信息。

前面提到的CLR(公共语言运行时)编译过程中生成 元信息 实现的反射是基于有一个运行时虚拟机的形式，其实现了一种对于C#这种静态类型语言相对高程度的反射。那么对于C++这种没有运行时的纯静态类型语言，在运行时没有提供足够多的反射信息的情况下，我们该如何实现一个应用层的反射？Protobuf Reflection 为我们提供了一个样例。

## Protobuf Reflection

前面我们提到，实际上元信息就是一种对系统的自描述信息。实际上我们在使用Protobuf的第一步遍实现了这个操作，即定义 .proto 文件。在 .proto 文件中，这些元信息包括数据由哪些字段构成，字段又属于什么类型以及字段之间的组合关系等。

当然元信息也并非一定由 .proto 文件提供，它也可来自于网络或其它可能的输入，只要它满足 ProtoBuf Message 的定义语法即可。那么元信息的可能来源和处理就有：

- .proto 文件
    - 使用 ProtoBuf 内置的工具 protoc 编译器编译，protoc 将 .proto 文件内容编码并写入生成的代码中（.pb.cc 文件）
    - 使用 ProtoBuf 提供的编译 API 在运行时手动（指编码）解析 .proto 文件内容。实际上 protoc 底层调用的也正是这个编译 API。
- 非 .proto 文件
    - 从远程读取，如将数据与数据元信息一同进行 protobuf 编码并传输

**无论 .proto 文件来源于何处，我们都需要对其做进一步的处理，将其解析成内存对象，并构建其与实例的映射，同时也要计算每个字段的内存偏移**。可总结出如下步骤：

1. 提供 .proto （范指 ProtoBuf Message 语法描述的元信息）
2. 解析 .proto 构建 FileDescriptor、FieldDescriptor 等，即 .proto 对应的内存模型（对象）
3. 之后每创建一个实例，就将其存到相应的实例池中
4. 将 Descriptor 和 instance 的映射维护到表中备查
5. 通过 Descriptor 可查到相应的 instance，又由于了解 instance 中字段类型（FieldDescriptor），所以知道字段的内存偏移，那么就可以访问或修改字段的值

### Pb 反射实例

我们可以看一下使用 Pb 反射能力的一个例子，protobuf对于每个元素都有一个相应的descriptor，这个descriptor包含该元素的所有元信息。可以根据 type name 创建具体类型的 Message 对象，也能够动态获取和设置某个属性。

```cpp
/* 反射创建实例 */
auto descriptor = google::protobuf::DescriptorPool::generated_pool()->FindMessageTypeByName("Dog");
auto prototype = google::protobuf::MessageFactory::generated_factory()->GetPrototype(descriptor);
auto instance = prototype->New();

/* 反射相关接口 */
auto reflecter = instance.GetReflection();
auto field = descriptor->FindFieldByName("name");
reflecter->SetString(&instance, field, "鸡你太美") ;

// 获取属性的值.
std::cout<<reflecter->GetString(instance , field)<< std::endl ;
return 0 ;
```

- **代码第一步我们通过 DescriptorPool 的 FindMessageTypeByName 获得了元信息 Descriptor。**
    
    DescriptorPool 为元信息池，对外提供了诸如 FindServiceByName、FindMessageTypeByName 等各类接口以便外部查询所需的元信息。当 DescriptorPool 不存在时需要查询的元信息时，将进一步到 DescriptorDatabase 中去查找。DescriptorDatabase 可从硬编码或磁盘中查询对应名称的 .proto 文件内容，解析后返回查询需要的元信息。
    
    DescriptorPool 相当于缓存了文件的 Descriptor（底层使用 Map），查询时将先到缓存中查询，如果未能找到再进一步到 DB 中（即 DescriptorDatabase）查询，此时可能需要从磁盘中读取文件内容，然后再解析成 Descriptor 返回，这里需要消耗一定的时间。
    
    从上面的描述不难看出，DescriptorPool 和 DescriptorDatabase 通过缓存机制提高了反射运行效率，但这只是反射工程实现上的一种优化，我们更感兴趣的应该是 Descriptor 的来源。
    
    DescriptorDatabase 从磁盘中读取 .proto 内容并解析成 Descriptor 这一来源很容易理解，但我们大多数时候并不会采用这种方式，反射时也不会去读取 .proto 文件。那么我们的 .proto 内容在哪？
    
    实际上我们在使用 protoc 生成 `xxx.pb.cc`和 `xxx.pb.h` 文件时，其中不仅仅包含了读写数据的接口，还包含了 .proto 文件内容。阅读任意一个 `xxx.pb.cc`的内容，你可以看到如下类似代码
    
    ```cpp
    // XXX.pb.cc
    static void AddDescriptorsImpl() {
      InitDefaults();
    
      // .proto 内容
      static const char **descriptor**[] GOOGLE_PROTOBUF_ATTRIBUTE_SECTION_VARIABLE(protodesc_cold) = {
          "\n\022single_int32.proto\"\035\n\010Example1\022\021\n\010int3"
          "2Val\030\232\005 \001(\005\" \n\010Example2\022\024\n\010int32Val\030\377\377\377\377"
          "\001 \003(\005b\006proto3"
      };
    
      **// 注册 descriptor
      ::google::protobuf::DescriptorPool::InternalAddGeneratedFile(
          descriptor, 93);**
    
      **// 注册 instance
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedFile(
        "single_int32.proto", &protobuf_RegisterTypes);**
    }
    ```
    
    其中 descriptor 数组存储的便是 .proto 内容。这里当然不是简单的存储原始文本字符串，而是经过了 **SerializeToString** 序列化处理，而后将结果以硬编码的形式保存在 xxx.pb.cc 中，真是充分利用了自己的高效编码能力。硬编码的 .proto 元信息内容将以懒加载的方式（被调用时才触发）被 DescriptorDatabase 加载、解析，并缓存到 DescriptorPool 中。当然，如果遇到没有被触发的pb需要反射，则需要提前手动触发一下。
    
- **代码中的第二步是根据 MessageFactory 获得了一个实例。**
    
    MessageFactory 是实例工厂，对外提供了根据元信息 descriptor 获取相应实例的能力。
    
    其实在代码中已经涉及到该工厂，即：
    
    ```cpp
    // 注册对应 descriptor 的 instance 到 MessageFactory
    // InternalRegisterGeneratedFile 函数内部，会将创建一个实例并做好 descriptor 与 instance 的映射
    ::google::protobuf::MessageFactory::InternalRegisterGeneratedFile(
        "single_int32.proto", &protobuf_RegisterTypes);
    ```
    
    每次构建实例后，都将 descriptor 和 instance 维护到一个 _table 中，即映射表以便获取。后续所谓通过反射获得某个类的某个实例子，实际就是查表的过程。
    
- **代码中的第三步，就是对 instance 实例对象的属性进行读写。**
    
    实例对象的 reflection 里面存储了对象属性的偏移地址，而这些信息其实与 .proto 内容信息一样，在 protoc 编译时通过解析 proto 文件内容获得且记录在 `xxx.pb.cc` 中，阅读  `xxx.pb.cc` 代码，可看到如下类似代码：
    
    ```cpp
    const ::google::protobuf::uint32 TableStruct::offsets[] GOOGLE_PROTOBUF_ATTRIBUTE_SECTION_VARIABLE(protodesc_cold) = {
      ~0u,  // no _has_bits_
      // 将会计算实例与属性的内存偏移
      GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(::Example1, _internal_metadata_),
      ~0u,  // no _extensions_
      ~0u,  // no _oneof_case_
      ~0u,  // no _weak_field_map_
      GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(::Example1, int32val_),
      ~0u,  // no _has_bits_
      GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(::Example2, _internal_metadata_),
      ~0u,  // no _extensions_
      ~0u,  // no _oneof_case_
      ~0u,  // no _weak_field_map_
      GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(::Example2, int32val_),
    };
    ```
    
    有了属性的内存偏移，自然可以对属性进行读写操作，以例子中出现的 SetString 为例，其内部实现位于 `generated_message_reflection.cc` 中，核心代码如下：
    
    ```cpp
    // 获取属性内存地址指针，内部根据 __
    const std::string* default_ptr =
        &DefaultRaw<ArenaStringPtr>(field).Get();
    
    // DefaultRaw 底层调用：
    // reinterpret_cast<const uint8*>
    // (default_instance_) +
    //     OffsetValue(offsets_[field->index()], field->type());
    
    // .......
    
    // assign 赋值
    MutableField<ArenaStringPtr>(message, field)
        ->Mutable(default_ptr, GetArena(message))
        ->assign(std::move(value));
    ```
    
    其他 SetInt32、SetInt64、SetBool 等等接口原理类似。