# C++11 特性

1998 年，C++ 标准委员会发布了第一版 C++ 标准，并将其命名为 C++ 98 标准。经过作者的不断迭代，一本书往往会先后发布很多个版本，其中每个新版本都是对前一个版本的修正和更新。C++ 编程语言的wq发展也是如此。截止到目前（2020 年），C++ 的发展历经了以下 3 个个标准：

- 2011 年，新的 C++ 11 标准诞生，用于取代 C++ 98 标准。此标准还有一个别名，为“C++ 0x”；
- 2014 年，C++ 14 标准发布，该标准库对 C++ 11 标准库做了更优的修改和更新；
- 2017 年底，C++ 17 标准正式颁布。

以上 3 个标准中，相比对前一个版本的修改和更新程度，C++ 11 标准无疑是颠覆性的，该标准在 C++ 98 的基础上修正了约 600 个 C++ 语言中存在的缺陷，同时添加了约 140 个新特性，这些更新使得 C++ 语言焕然一新。读者可以这样理解 C++ 11 标准，它在 C++ 98/03 标准的基础上孕育出了全新的 C++ 编程语言，造就了 C++ 新的开始。

## auto 类型推导

在编程语言分类中，C/C++常常被认为是静态类型的语言。而有的编程语言则号称是“动态类型”的，一般来说动态语言是在运行时来进行类型检查，而C++这种中类型检查是在编译阶段完成的。

事实上，类型推导也可以用于静态类型语言中。如在 `auto a = 1;`  我们还是能够轻松地推导出a的类型是int的。这里使用auto关键字来要求编译器对变量name的类型进行了自动推导。这里编译器根据它的初始化表达式的类型，推导出a的类型为int。但值得注意的是C++类型推导与动态语言类型推导之间的区别，C++是一种强类型的语言，类型的确定需要在编译阶段确定。所以C++的类型推导需在编译阶段完成，而动态语言则是在运行阶段完成的。在了解auto之前，需先了解C++的模板类型推导：[C++ 模板类型推导](C++%20%E6%A8%A1%E6%9D%BF%E7%B1%BB%E5%9E%8B%E6%8E%A8%E5%AF%BC%204a455d94f4724b4e834917c5bd3f92c5.md) 

在了解了C++的模板类型推导后，再看auto的类型推导就显得非常简单了，两者基本是一致的，函数模板推断的任务是：

```cpp
template <typename T>
void f(ParamType param);

f(expr);   // 根据expr类型推导出T和ParamType的类型
```

编译器要根据expr类型推导出T和ParamType的类型。移植到auto上是那么容易：把auto看成函数模板中的T，而把变量的实际类型看成ParamType。这样我们可以把auto类型推断转换成函数模板类型推断，还是例子说话：

```cpp
// auto推断例子
auto x = 10;
const auto cx = x;
const auto& rx = x;

// 传化为模板类型推断
template <typename T>
void f1(T param);
f1(10);          

template <typename T>
void f2(const T param);
f2(x);  

template <typename T>
void f3(const T& param);
f3(x);
```

显然，很容易推断出各个变量的类型。前面说到，函数模板类型推断有三种情况，那么对于auto来说，仍然有三种情形： 1. 类型修饰符是一个指针或者引用，但是不是通用引用； 2. 类型修饰符是一个通用引用； 3. 类型修饰符不是指针，也不是引用。

```cpp
const int N = 2;
auto x = 10;   // 情形3: int
const auto cx = x; // 情形3： const int
const auto& rx = x;  // 情形1：const int&
auto y = N;         // 情形3： int
auto *p1 = &x;   //p1 为 int *，auto 推导为 int
auto  p2 = &x;   //p2 为 int*，auto 推导为 int*

// 情形2
auto&& y1 = x;   // 左值：int&
auto&& y2 = cx;  // 左值: const int&
auto&& y3 = 10;  // 右值：int&&
```

可以看到，auto与函数模板类型推断本质上是一致的。但是有一个特殊情况，那就是C++11支持统一初始化方式（其实也可以理解，毕竟模板类型推导是C++98的东西，而auto是和std::initialzer_list<int> 在C++11一起进的）：

```cpp
// 等价的初始化方式
int x1 = 9;
int x2(9);
// 统一初始化
int x3 = {9};
int x4{9};
```

上面的4种方式都可以用来初始化一个值为9的int变量，那么你可能会想下面的代码是同样的效果：

```cpp
auto x1 = 9;    //int
auto x2(9);     //int
auto x3 = {9};  //std::initialzer_list<int>
auto x4{9};     //std::initialzer_list<int>
```

但是实际上不是这样：对于前两个，确实是初始化了值为9的int类型变量，但是后两者确是得到了包含元素9的std::initialzer_list<int>对象（初始化列表），这算是auto的一个特例吧。但是这对函数模板类型推断并不适用：

```cpp
auto x = {1, 3, 5}  // 合法：std::initializer_list<int>类型

template<typename T>
void f(T param);

f({1, 3, 5});  // 非法，无法编译：不能推断出T的类型

// 可以修改成下面
template <typename T>
void f2(std::initializer_list<T> param);

f2({1, 3, 5});  // 合法:T是int，param是std::initializer_list<int>
```

上面讲的都是关于auto用于变量定义时的类型推断。但是C++14中auto还可以用于函数返回类型的推断以及泛型lambda表达式（其参数支持自动推断类型）。如下面的例子：

```cpp
// C++14功能
// 定义一个判断是否大于10的泛型lambda表达式
auto isGreaterThan10 = [] (auto i) { return i > 10;};

bool r = isisGreaterThan10(20);  // true

// auto用于函数返回类型自动推断
auto multiplyBy2Lambda(int x)
{
    return [x] {return 2 * x;};
}
auto f = multiplyBy2Lambda(4);
cout << f() << endl;   // 8
```

### auto的限制

1) auto 不能在函数的参数中使用。

这个应该很容易理解，我们在定义函数的时候只是对参数进行了声明，指明了参数的类型，但并没有给它赋值，只有在实际调用函数的时候才会给参数赋值；而 auto 要求必须对变量进行初始化，所以这是矛盾的。

2) auto 不能作用于类的非静态成员变量（也就是没有 static 关键字修饰的成员变量）中。

3) auto 关键字不能定义数组，比如下面的例子就是错误的：

```cpp
char url[] = "http://c.biancheng.net/";
auto  str[] = url;  //arr 为数组，所以不能使用 auto
```

4) auto 不能作用于模板的参数，请看下面的例子：

```cpp
template <typename T>
class A{
    //TODO:
};
int  main(){
    A<int> C1;
    A<auto> C2 = C1;  //错误
    return 0;
}
```

### auto的使用

auto 用于泛型编程：auto 的另一个应用就是当我们不知道变量是什么类型，或者不希望指明具体类型的时候，比如泛型编程中。我们接着看例子：

```cpp
#include <iostream>
using namespace std;
class A{
public:
    static int get(void){
        return 100;
    }
};
class B{
public:
    static const char* get(void){
        return "http://c.biancheng.net/cplus/";
    }
};
template <typename T>
void func(void){
    auto val = T::get();
    cout << val << endl;
}
int main(void){
    func<A>();
    func<B>();
    return 0;
}
```

## decltype 类型推导

decltype 是 C++11 新增的一个关键字，它和 auto 的功能一样，都用来在编译时期进行自动类型推导。decltype 是“declare type”的缩写，译为“声明类型”。既然已经有了 auto 关键字，为什么还需要 decltype 关键字呢**？因为 auto 并不适用于所有的自动类型推导场景，在某些特殊情况下 auto 用起来非常不方便，甚至压根无法使用，所以 decltype 关键字也被引入到 C++11 中**。

auto 和 decltype 关键字都可以自动推导出变量的类型，但它们的用法是有区别的：

```cpp
auto varname = value;
decltype(exp) varname = value;
```

其中，varname 表示变量名，value 表示赋给变量的值，exp 表示一个表达式。

**auto 根据=右边的初始值 value 推导出变量的类型，而 decltype 根据 exp 表达式推导出变量的类型，跟=右边的 value 没有关系。另外，auto 要求变量必须初始化，而 decltype 不要求。这很容易理解，auto 是根据变量的初始值来推导出变量类型的，如果不初始化，变量的类型也就无法推导了。**

decltype 可以写成下面的形式：`decltype(exp) varname;` 举个例子：

```cpp
int a = 0;
decltype(a) b = 1;  //b 被推导成了 int
decltype(10.8) x = 5.5;  //x 被推导成了 double
decltype(x + 100) y;  //y 被推导成了 double, y没有赋初始值，这和auto不同
```

可以看到，decltype 能够根据变量、字面量、带有运算符的表达式推导出变量的类型

decltype有三种调用方式：

- **decltype + 变量**

当使用`decltype(var)`的形式时，decltype 会直接返回变量的类型（包括顶层 const 和引用），不会返回变量作为表达式的类型。

decltype 加指针也会返回指针的类型。decltype 加数组，不负责把数组转换成对应的指针，所以其结果仍然是个数组。总之 `devltype(var)` 完美保留了变量的类型

- **decltype + 表达式**

当使用`decltype(expr)`的形式时，decltype 会返回表达式结果对应的类型。因此，`decltype(expr)`的结果根据 expr 的结果不同而不同：expr 返回左值，得到该类型的左值引用；expr 返回右值，得到该类型。

- **decltype + 函数**

C++ 中通过函数的返回值和形参列表，定义了一种名为**函数类型**的东西。当直接使用函数名作为参数如`decltype(func_name)`的形式时，decltype 会返回对应的函数类型，不会自动转换成相应的函数指针。

```cpp
int add_to(int &des, int ori);
decltype(add_to) *pf = add_to;
```

### decltype 推导规则

上面的例子让我们初步感受了一下 decltype 的用法，但你不要认为 decltype 就这么简单，它的玩法实际上可以非常复杂。当程序员使用 decltype(exp) 获取类型时，编译器将根据以下三条规则得出结果：

- 如果 exp 是一个不被括号`( )`包围的表达式，或者是一个类成员访问表达式，或者是一个单独的变量，那么 decltype(exp) 的类型就和 exp 一致，这是最普遍最常见的情况。
- 如果 exp 是函数调用，那么 decltype(exp) 的类型就和函数返回值的类型一致。需要注意的是，exp 中调用函数时需要带上括号和参数，但这仅仅是形式，并不会真的去执行函数代码。
- **如果 exp 是一个左值，或者被括号`( )`包围，那么 decltype(exp) 的类型就是 exp 的引用**；假设 exp 的类型为 T，那么 decltype(exp) 的类型就是 T&。(注意这里的“左值“和第一条中“一个单独的变量”可能有一点奇异，这里可以一般解释为通过表达式得到的左值如：n=n+m)

```cpp
//简单情况
int n = 0;
const int &r = n;
Student stu;
decltype(n) a = n;  //n 为 int 类型，a 被推导为 int 类型
decltype(r) b = n;     //r 为 const int& 类型, b 被推导为 const int& 类型

//函数
int& func_int_r(int, char);  //返回值为 int&
int&& func_int_rr(void);  //返回值为 int&&
int func_int(double);  //返回值为 int
const int& fun_cint_r(int, int, int);  //返回值为 const int&
const int&& func_cint_rr(void);  //返回值为 const int&&
//> decltype类型推导
int n = 100;
decltype(func_int_r(100, 'A')) a = n;  //a 的类型为 int&
decltype(func_int_rr()) b = 0;  //b 的类型为 int&&
decltype(func_int(10.5)) c = 0;   //c 的类型为 int
decltype(fun_cint_r(1,2,3))  x = n;    //x 的类型为 const int &
decltype(func_cint_rr()) y = 0;  // y 的类型为 const int&&

//带有括号的表达式
decltype(obj.x) a = 0;  //obj.x 为类的成员访问表达式，符合推导规则一，a 的类型为 int
decltype((obj.x)) b = a;  //obj.x 带有括号，符合推导规则三，b 的类型为 int&。
//加法表达式
int n = 0, m = 0;
decltype(n + m) c = 0;  //n+m 得到一个右值，符合推导规则一，所以推导结果为 int
decltype(n = n + m) d = c;  //n=n+m 得到一个左值，符号推导规则三，所以推导结果为 int&
```

实际使用：

```cpp
template <typename T>
class Base {
public:
    void func(T& container) {
        m_it = container.begin();
    }
private:
    typename T::iterator m_it;  //不使用decltype
		decltype(T().begin()) m_it;  //注意这里
		
};
```

## auto和decltype的区别

auto 和 decltype 都是 C++11 新增的关键字，都用于自动类型推导，但是它们的语法格式是有区别的，如下所示：

```cpp
auto varname = value;  //auto的语法格式
decltype(exp) varname [= value];  //decltype的语法格式， []可有可无
```

auto 和 decltype 都会自动推导出变量 varname 的类型：

- auto 根据`=`右边的初始值 value 推导出变量的类型。auto 要求变量必须初始化，也就是在定义变量的同时必须给它赋值；
- decltype 根据 exp 表达式推导出变量的类型，跟`=`右边的 value 没有关系。decltype因此不要求变量初始化。

auto 将变量的类型和初始值绑定在一起，而 decltype 将变量的类型和初始值分开；虽然 auto 的书写更加简洁，但 decltype 的使用更加灵活。

**对 cv 限定符的处理的不同：**

「cv 限定符」是 **const** 和 **volatile** 关键字的统称。在推导变量类型时，auto 和 decltype 对 cv 限制符的处理是不一样的。**decltype 会保留 cv 限定符，而 auto 有可能会去掉 cv 限定符**。

以下是 auto 关键字对 cv 限定符的推导规则：

- 如果表达式的类型不是指针或者引用(T)，auto 会把 cv 限定符直接抛弃，推导成 non-const 或者 non-volatile 类型。
- 如果表达式的类型是指针或者引用(T&)，auto 将保留 cv 限定符。

**对引用的处理的不同：**

当表达式的类型为引用时，decltype 会保留引用类型，而 auto 会抛弃引用类型，直接推导出它的原始类型。

auto 虽然在书写格式上比 decltype 简单，但是它的推导规则复杂，有时候会改变表达式的原始类型；而 decltype 比较纯粹，它一般会坚持保留原始表达式的任何类型，让推导的结果更加原汁原味。

值得一提的是，在学习auto&decltype的过程中有人提到了RTTI的概念，即”Runtime Type Information”的缩写，意思是运行时类型信息。但auto&decltype都不属于RTTI的概念，因为这两种类型推导和动态语言如python不同，这两个关键词所进行的类型推导都是在编译阶段由编译器执行的，而动态语言类型推导是在运行时进行的。

## switch内部变量定义问题

遇到一个在switch的case语句中定义变量的问题：

```cpp
bool b = false;
switch (false)
{
    case true:
        int ival;
        break;
    case false:
        ival = 3;
        cout << ival << endl;
}
```

段c++代码应该直接执行switch语句中的 case false，这样的话ival应该没有被定义，为什么程序能被编译通过，并顺利执行？

实际上变量的声明和定义的作用是在静态域，与是否执行到没有关系。定义变量并不存在执行动作。

为什么这个容易误解呢，因为c语言switch语句设计的比较悲剧，每个case部分是没有独立的作用域的。要理解它，一种方法是把它当作goto来看。 比如这个程序就是：

```cpp
int main() {
    bool b = false;
    goto case_false;
case_true:
    int ival;
    goto switch_end;
case_false:
    ival = 3;
    cout << ival << endl;
switch_end:
    return 0;
}
```

c++为了方便可以随时声明变量，但系统在函数调用时还是一次性分配所有函数变量所需空间，大部分编译期实现会选择在函数开始把所有局部变量的空间都分配好（只要改一下堆栈指针就好）。所以从这段代码来看，函数在开始执行之前就给那个变量分配了内存，通过变量名可以访问到对应内存的值，因为没有初始化，所以结果应该是随机的。