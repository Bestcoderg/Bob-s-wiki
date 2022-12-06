# C++11 lambda表达式

从 C++11 开始，C++ 有三种方式可以创建/传递一个可以被调用的对象：

- **函数指针：**函数指针是从 C 语言老祖宗继承下来的东西，比较原始，功能也比较弱：
1. 无法直接捕获当前的一些状态，所有外部状态只能通过参数传递（不考虑在函数内部使用 static 变量）。
2. 使用函数指针的调用无法 inline（编译期无法确定这个指针会被赋上什么值）。
- **仿函数（Functor）**：仿函数其实就是让一个类（class/struct）的对象的使用看上去像一个函数，具体实现就是在类中实现 operator()。比如（相比函数指针，仿函数对象可通过成员变量来捕获/传递一些状态。缺点就是写起来很麻烦）

```cpp
class Plus {
 public:
  int operator()(int a, int b) {
    return a + b;
  }   
};
Plus plus; 
std::cout << plus(11, 22) << std::endl;   // 输出 33
```

- **Lambda 表达式**

### Lambda 表达式

首先我们要说明什么是Lambda表达式？Lambda 表达式（lambda expression）是一个匿名函数，Lambda表达式基于数学中的λ演算得名。Lambda表达式可以表示闭包（注意和数学传统意义上的不同）。

**构造闭包：能够捕获作用域中变量的匿名函数的对象，Lambda 表达式是纯右值表达式，其类型是独有的无名非联合非聚合类类型，被称为闭包类型（closure type），所以在声明的时候必须使用 auto 来声明。**

**虽然 lambda 的使用和函数对象的调用方式有相似之处，但他们并不是同一种东西，lambda 的类型是不可知的（在编译期决定），使用 sizeof 两者的大小也是不相同的，std::function 是函数对象，通过消除类型再重载 operator() 达到调用的效果，只要这个函数满足可以调用的条件，就可以使用std::function保存起来，这也是上面例子的体现。**

Lambda 表达式完整的声明格式如下：

```cpp
	[capture list] (params list) mutable exception-> return type { function body }
```

- capture list：捕获外部变量列表
- params list：形参列表
- mutable 指示符：用来说用是否可以修改捕获的变量
- exception：异常规范
- return type：返回类型
- function body：函数体

此外，我们还可以省略其中的某些成分来声明 “不完整” 的 Lambda 表达式，常见的有以下几种：

1. **[capture list] (params list) -> return type {function body}** : 格式 1 声明了 const 类型的表达式，这种类型的表达式不能修改捕获列表中的值。
2. **[capture list] (params list) {function body}** : 格式 2 省略了返回值类型，但**编译器可以根据以下规则推断出 Lambda 表达式的返回类型:**（1）如果 function body 中存在 return 语句，则该 Lambda 表达式的返回类型由 return 语句的返回类型确定;（2）如果 function body 中没有 return 语句，则返回值为 void 类型。
3. **[capture list] {function body}** : 格式 3 中省略了参数列表，类似普通函数中的无参函数。

在 Lambda 表达式中定义形参传递参数还有一些限制，主要有以下几点：

1. 参数列表中不能有默认参数
2. 不支持可变参数
3. 所有参数必须有参数名

所以lambda表达式主要会使用在什么情况下呢？下面我们举一个实例：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

bool cmp(int a, int b)
{
    return  a < b;
}

int main()
{
    vector<int> myvec{ 3, 2, 5, 7, 3, 2 };
    vector<int> lbvec(myvec);

    sort(myvec.begin(), myvec.end(), cmp); // 旧式做法
    cout << "predicate function:" << endl;
    for (int it : myvec)
        cout << it << ' ';
    cout << endl;

    sort(lbvec.begin(), lbvec.end(), [](int a, int b) -> bool { return a < b; });   // Lambda表达式
    cout << "lambda expression:" << endl;
    for (int it : lbvec)
        cout << it << ' ';
}
```

在 C++11 之前，我们使用 STL 的 sort 函数，需要提供一个定义的函数cmp（C++函数是无法在使用的时候定义的）如果使用 C++11 的 Lambda 表达式，我们只需要传入一个匿名函数即可，方便简洁，而且代码的可读性也比旧式的做法好多了。

### 捕获外部变量

Lambda 表达式可以使用其可见范围内的外部变量，但必须明确声明（明确声明哪些外部变量可以被该 Lambda 表达式使用）。那么，在哪里指定这些外部变量呢？Lambda 表达式通过在最前面的方括号 [] 来明确指明其内部可以访问的外部变量，这一过程也称过 Lambda 表达式 “捕获” 了外部变量。

```cpp
#include <iostream>
using namespace std;
int main()
{
    int a = 123;
    auto f = [a] { cout << a << endl; }; 
    f();   // 输出：123
}
```

类似参数传递方式（值传递、引入传递、指针传递），在 Lambda 表达式中，外部变量的捕获方式也有**值捕获、引用捕获、隐式捕获**。

- **值捕获 [ a ]**：值捕获和参数传递中的值传递类似，被捕获的变量的值在 Lambda 表达式创建时通过值拷贝的方式传入，因此**随后对该变量的修改不会影响影响 Lambda 表达式中的值**。这里需要注意的是，如果lambda表达式以值传递，如果没有使用**mutable**来声明lambda表达式，则在lambda表达式中**对于值传递的变量的改变是不被允许的**。
- **引用捕获 [ &a ]** : 使用引用捕获一个外部变量，只需要在捕获列表变量前面加上一个引用说明符 &。引用捕获允许改变变量且会影响外部变量。
- **隐式捕获 [=] [&]**： **值捕获和引用捕获**都需要我们在捕获列表中**显示列出 Lambda 表达式中使用的外部变量**。除此之外，我们还可以**让编译器根据函数体中的代码来推断需要捕获哪些变量**，这种方式称之为隐式捕获。隐式捕获有两种方式，分别是 [=] 和[&]。[=]表示以值捕获的方式捕获外部变量，[&]表示以引用捕获的方式捕获外部变量。

[lambda捕获](C++11%20lambda%E8%A1%A8%E8%BE%BE%E5%BC%8F%2044bc3321a9ee41eb9a9eb334f2d9dd84/lambda%E6%8D%95%E8%8E%B7%20a1f5b4d892274fa1aabc48a8228cd855.csv)

### 捕获列表初始化（C++14）

捕获列表初始化（Capture Initializers）是C++14支持的一种功能。如：

```cpp
// 按值捕获 target，但是在 Lambda 内部的变量名叫做 v
auto cnt =
    std::count_if(books.begin(), books.end(), [v = target](const Book& book) {
        return book.title.find(v) != std::string::npos;
    }); 

// 按引用捕获 target，但是在 Lambda 内部的名字叫做 r
auto cnt =
    std::count_if(books.begin(), books.end(), [&r = target](const Book& book) {
        return book.title.find(r) != std::string::npos;
    });
```

Lambda 捕获列表初始化最最最重要的一点是“支持 Capture by Move”。在 C++14 之前，Lambda 是不支持捕获一个 **Move-Only** 的对象的（如 unique_ptr ），比如在C++11中：

```
std::unique_ptr<int> uptr = std::make_unique<int>(123);
auto callback = [uptr]() {                               // 编译错误，uptr is move-only
    std::cout << *uptr << std::endl;
};
```

按引用捕获虽然可以编译通过，但往往是不符合要求的。比如下面的例子，离开作用域之后 uptr 会被析构掉。但是 callback 对象已经被传给另一个线程。

```cpp
std::unique_ptr<int> uptr = std::make_unique<int>(123);
auto callback = [&uptr]() {                               
    std::cout << *uptr << std::endl;
};  
// ... 将 callback 传给另一个线程
// return => uptr delete 掉指向的内存
```

通过捕获列表初始化，完成 Move-Only 对象的“Capture by Move”。

```
std::unique_ptr<int> uptr = std::make_unique<int>(123);
auto callback = [uptr = std::move(uptr)]() {    // 将 uptr 移动给 Lambda 表达式中的参数
    std::cout << *uptr << std::endl;
};
// ... 将 callback 传给另一个线程
// return => uptr 是 nullptr
```

### 表达式的延迟调用

一个容易出错的细节是关于 lambda 表达式的延迟调用的：

```cpp
int a = 0;
auto f = [=]{ return a; };      // 按值捕获外部变量
a += 1;                         // a被修改了
std::cout << f() << std::endl;  // 输出？
```

在这个例子中，lambda 表达式按值捕获了所有外部变量。在捕获的一瞬间，a 的值就已经被复制到f中了。之后 a 被修改，但此时 f 中存储的 a 仍然还是捕获时的值，因此，最终输出结果是 0。

如果希望 lambda 表达式在调用时能够即时访问外部变量，我们应当使用引用方式捕获。

### lambda 表达式的类型

lambda 表达式的类型在 C++11 中被称为“闭包类型（Closure Type）”。它是一个特殊的，匿名的非 nunion 的类类型。

因此，我们可以认为它是一个带有 operator() 的类，即仿函数。因此，我们可以使用 std::function 和 std::bind 来存储和操作 lambda 表达式：

```cpp
std::function<int(int)>  f1 = [](int a){ return a; };
std::function<int(void)> f2 = std::bind([](int a){ return a; }, 123);
```

另外，对于没有捕获任何变量的 lambda 表达式，还可以被转换成一个普通的函数指针：

```cpp
using func_t = int(*)(int);
func_t f = [](int a){ return a; };
f(123);
```

lambda 表达式可以说是就地定义仿函数闭包的“语法糖”。它的捕获列表捕获住的任何外部变量，最终均会变为闭包类型的成员变量。而一个使用了成员变量的类的 operator()，如果能直接被转换为普通的函数指针，那么 lambda 表达式本身的 this 指针就丢失掉了。而没有捕获任何外部变量的 lambda 表达式则不存在这个问题。

这里也可以很自然地解释为何按**值捕获无法修改捕获的外部变量**。因为按照 C++ 标准，**lambda 表达式的 operator() 默认是 const 的。一个 const 成员函数是无法修改成员变量的值的。而 mutable 的作用，就在于取消 operator() 的 const。**

需要注意的是，没有捕获变量的 lambda 表达式可以直接转换为函数指针，而捕获变量的 lambda 表达式则不能转换为函数指针。看看下面的代码：

```cpp
typedef void(*Ptr)(int*);
Ptr p = [](int* p){delete p;};  // 正确，没有状态的lambda（没有捕获）的lambda表达式可以直接转换为函数指针
Ptr p1 = [&](int* p){delete p;};  // 错误，有状态的lambda不能直接转换为函数指针
```

### lambda表达式工作原理

 编译器会把一个 lambda 表达式生成一个**匿名类的匿名对象**，并在类中重载函数调用运算符, 实现了一个 operator() 方法。类似于仿函数的写法：

```cpp
auto print = []{cout << "Hello World!" << endl; };
```

 编译器会把上面这一句翻译为下面的代码:

```cpp
class print_class
{
public:
	void operator()(void) const  //注意这个const
	{
		cout << "Hello World！" << endl;
	}
};
//用构造的类创建对象，print此时就是一个函数对象
auto print = print_class();
```

**我们再来看一下值捕获：**

```cpp
int year = 19900212;
char *name = "zhangxiang";
//采用值捕获，捕获所有的已定义的局部变量，如year，name
auto print = [=](){
	cout << year << ends << name << endl;
};
```

```cpp
int year = 19900212;
char *name = "zhangxiang";
class print_class
{
public:
	//根据捕获列表来决定构造函数的参数列表形式
	print_class(int year, char *name) :year(year), name(name) { }
	void operator()(void) const
	{
		cout << year << ends << name << endl;
	}
private:
	int year;
	char *name;
};
auto print = print_class(a, str);
```

**如果是引用捕获：**

```cpp
	print_class(int &year, char *&name) :year(year), name(name){}
	void operator()(void) const
	{	
			year++;   //编译通过，const对引用类型无效
			cout << year << ends << name << endl;
	}
```

混合捕获也是同理，主要就是在仿函数类的初始化函数的不同。

lambda在实现上更加趋向于仿函数的实现形式。

### 应用实例

就地定义匿名函数，不再需要定义函数对象，大大简化了标准库算法的调用。比如，在 C++11 之前，我们要调用 for_each 函数将 vector 中的偶数打印出来，如下所示。

【实例】lambda 表达式代替函数对象的示例。

```cpp
class CountEven
{
    int& count_;
public:
    CountEven(int& count) : count_(count) {}
    void operator()(int val)
    {
        if (!(val & 1))       // val % 2 == 0
        {
            ++ count_;
        }
    }
};
std::vector<int> v = { 1, 2, 3, 4, 5, 6 };
int even_count = 0;
for_each(v.begin(), v.end(), CountEven(even_count));
std::cout << "The number of even is " << even_count << std::endl;
```

这样写既烦琐又容易出错。有了 lambda 表达式以后，我们可以使用真正的闭包概念来替换掉这里的仿函数，代码如下：

```cpp
std::vector<int> v = { 1, 2, 3, 4, 5, 6 };
int even_count = 0;
for_each( v.begin(), v.end(), [&even_count](int val)
        {
            if (!(val & 1))  // val % 2 == 0
            {
                ++ even_count;
            }
        });
std::cout << "The number of even is " << even_count << std::endl;
```

**lambda 表达式的价值在于，就地封装短小的功能闭包，可以极其方便地表达出我们希望执行的具体操作，并让上下文结合得更加紧密**。