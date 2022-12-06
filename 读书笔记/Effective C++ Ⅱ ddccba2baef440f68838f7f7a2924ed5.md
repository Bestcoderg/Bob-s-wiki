# Effective C++ Ⅱ

## 条款 20：宁以 pass-by-reference-to-const 替换 pass-by-value

- 函数参数的传递尽量使用常量引用的形式替换值传递。并且常量引用的形式比较高效，避免参数的构造和析构，也避免了”对象切割“问题
- 但是对于内置类型以及 STL 迭代器和函数对象而言，值传递比较合适。

对象切割问题：如果形参是base class，如果采用值传递，即使传的是派生类调用的也是基类的copy

## 条款21：必须返回对象时，别妄想返回其reference

这个条款主要是防止矫枉过正，滥用返回reference。绝不要返回pointer或**reference指向一个local stack对象**，或者heap-allocated对象；也不要返回pointer或referece指向local static对象而有可能同时需要多个这样的对象。

通常来说，并不太需要担心返回对象的消耗，**编译器是存在RVO优化的，**尤其是C++11引入move后，选择就更加多样了。

## 条款22：将成员变量声明为private

将成员变量声明为private提供了更好的封装性，为**所有可能的实现提供弹性**，还可以验证class的约束条件以及函数的前提和时候状态等等。这样在改变成员变量提供的值或是添加check方式时，就只用改变接口的实现，不需要改所有引用的地方，这对于库来说非常的重要。

总之，将成员变量声明为private，可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。除此之外，protected并不比public好到哪去，因为protected改变后需要改变所有派生类调用的地方。

## 条款23：宁以non-member、non-friend替换member函数

与一般直觉相反，non-member函数的封装性要比member函数高(为什么这么说呢？因为member函数将增加对于类中数据的访问途径，这样的行为是在减少封装性)。**因为对于对象内部的数据，越少代码可以看到数据，其封装性就越高（我认为EC中对于封装的定义非常有启发性：愈多的东西被封装，愈少的人可以看到它。而愈少的人看到它，我们就有俞强大的弹性去变化它）。non-member non-friend函数不会增加可访问数据的函数数量。**对于这样的函数，将其与相应的类组织在同一个namespace是一个比较合理的选择。总之，使用non-member non-friend函数替换member函数，可以增加封装性、包裹弹性和机能扩充性。

一般来说现实的使用方式是，我们会将一些提供“便利”的函数放到工具类中，这个工具类将与其将调用的类处于同一命名空间。

## 条款 24：若所有参数皆需要类型转换，请为此采用non-member 函数

通常情况，class 不应该支持隐式类型转换，因为这样可能导致我们想不到的问题。这个规则也有例外，最常见的例外是建立数值类型时。例如编写一个分数管理类，允许隐式类型转换

```cpp
class Rational{
public:
	Rational(int numerator=0, int denominator=1);//非explicit，允许隐式转换
	……
};
```

如果要支持加减乘除等运算，这时重载运算符时是应该重载为 member 函数还是 non-member 函数呢，或者 non-member friend 函数？

如果写成 member 函数：

```cpp
class Rational{
public:
	……
	const Rational operator*(const Rational& rhs);
	……
};
```

这样编写可以 使得将两个有理数相乘

```cpp
Rational onEight(1,8);
Rational oneHalf(1,2);
Rational result=onEight*oneHalf;
result=result*onEight;
```

如果进行混合运算

```cpp
result=oneHalf*2;//正确，相当于oneHalf.operator*(2);
result=2*oneHalf;//错误，相当于2.operator*(oneHalf);
```

不能满足交换律。因为 2 不是 Rational 类型，不能作为左操作数。oneHalf*2 会把 2 隐式转换为 Rational 类型。

上面两种做法，第一种可以发生隐式转换，第二种却不可以，这是因为**只有当参数被列于参数列（parameter list）内，这个参数才是隐式类型转换的合格参与者。第二种做法，还没到到” 参数被列于参数列内 “，2 不是 Rational 类型，不会调用 operator*。**

如果要支持混合运算，可以让 operator * 成为一个 non-member 函数，这样编译器可以在实参身上执行隐式类型转换。

```cpp
class Rational{
……
}
const Rational operator*(const Rational& lhs, const Rational& rhs)
{
……
}
```

**编译器在进行运算时会先去找到类内的operator函数，在找不到member的operator函数后则会去命名空间和global中找non-member的operator函数。以上写法就是基于这个事实的。**

这样就可以进行混合运算了。那么还有一个问题就是，是否应该是 operator * 成为 friend 函数。如果可以通过 public 接口，来获取内部数据，那么可以不是 friend 函数，否则，如果读取 private 数据，那么要成为 friend 函数。这里还有一个重要结论：member 函数的反面是 non-member 函数，不是 friend 函数。如果可以避免成为 friend 函数，那么最好避免，因为 friend 的封装低于非 friend。

当需要考虑 template 时，让 class 变为 class template 时，又有一些新的解法。这个在后面条款 46 有讲到。

## 条款25：考虑写出一个不抛异常的 swap 函数

缺省情况下 swap 动作可由标准程序库提供的 swap 算法：

```cpp
namespace std { 
    template<typename T> 
    void swap(T& a, T& b) { 
        T temp(a); 
        a = b; 
        b = temp; 
    } 
}
```

这个函数是异常安全性编程的核心，而且是用来处理自我赋值可能性的一个常见机制

可是对某些类型而言，这些复制动作无一必要：当中基本的就是 “以指针指向一个对象，内含真正数据” 那种类型。多为“pimpl 手法”（pointer to implementation 的缩写）

```cpp
class WidgetImpl { 
private: 
    int a, b, c; 
    std::vector<double> v; 
}; 
class Widget { 
public: 
    Widget(const Widget& rhs); 
    Widget& operator=(const Widget& rhs) { 
        *pImpl = *(rhs.pImpl); 
    } 
private: 
    WidgetImpl* pImpl; 
};
```

要置换两个 Widget 对象值。唯一要做的就是置换 pImpl 指针，缺省的 swap 算法不知道这一点。不仅仅复制 3 个 Widget 还复制 3 个 WidgetImpl 对象。很缺乏效率！

解决的方法:

我们能够令 Widget 声明一个 swap 的 public 成员函数做真正的替换工作，然后将 std::swap 特化。令他调用该成员函数：

```cpp
class Widget { 
public: 
    Widget(const Widget& rhs); 
    Widget& operator=(const Widget& rhs) { 
        *pImpl = *(rhs.pImpl); 
    } 
    void swap(Widget& other) { 
        using std::swap; 
        swap(pImpl, other.pImpl); 
    } 
private: 
    WidgetImpl* pImpl; 
};

class WidgetImpl { 
private: 
    int a, b, c; 
    std::vector<double> v; 
};
namespace std{
	template<> void swap<Widget>(Widget& a, Widget& b) { a.swap(b); } 
}
```

这样的做法不仅仅可以通过编译，还与 STL 容器有一致性。由于全部 STL 容器也都提供有 public swap 成员函数和 std::swap 特化版本号（用以调用前者）。

**如果 Widget 和 WidgetImpl 都是 class templates 而非 classes：**

```cpp
class WidgetImpl{…};
template <typename T>
class Widget{…};
```

所谓的 “partially specialize“：C++ 同意对类模板进行”部分特化 “。但不同意对模板函数进行” 部分特化“。

以下的定义就是错误的（将上例中的 Widget 类写成模板类）：

```cpp
namespace std{
	template<typename T>
	void swap<Widget<T>>(Widget<T>& a, Widget<T>& b){		//错误。不合法！
		a.swap(b);
	}
}
```

即使加入重载版本号也行不通。由于标准命名空间std是一个特殊的命名空间，客户能够全特化里面的模板，可是不能加入新的模板（包含类模板和函数模板）到标准命名空间中去。所以以下这样的方法不行！

```cpp
namespace std{
	template<typename T>
	void swap(Widget<T>& a, Widget<T>& b){		//试图重载，不合法。
		a.swap(b);
	}
}
```

一般而言，重载 function template 没有问题，但 std 是个特殊的命名空间。管理也就比較特殊。客户能够全特化 std 内的 templates。但不能够加入新的 templates（或 class 或 function 或不论什么其它东西）到 std 里头。事实上跨越红线的程序差点儿仍可编译运行，但他们行为没有明白定义。所以不要加入不论什么新东西到 std 里头。

**解决方法：**

为了提供较高效的 template 特定版本号。我们还是声明一个 non-member swap 但不再是 std::swap 的特化版本号或重载版本号：

```cpp
namespace WidgetStuff {
    template<typename T>
        class Widget{...};
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
}
template<typename T> 
void doSomething(T& obj1, T& obj2) { 
    swap(obj1, obj2); 
}
```

上面的应该使用哪个 swap？是 std 既有的那个一般化版本号还是某个可能存在的特化版本号等？你希望应该是调用 T 专属版本号。并在该版本号不存在的情况下调用 std 内的一般化版本号。以下是你希望发生的事：

```cpp
template<typename T> 
void doSomething(T& obj1, T& obj2) { 
    using std::swap;    //令std::swap在此函数内可用 
    swap(obj1, obj2); 
}
```

c++ 的名称查找法则（name lookup rules）确保将找到 global 作用域或 T 所在之命名空间内的不论什么 T 专属的 swap。假设 T 是 Widget 而且位于命名空间 WidgetStuff 内，编译器会找出 WidgetStuff 内的 swap。[C++ ADL](C++%20ADL%201209bf3c47844a5388fa6cad54a2aa38.md) 

假设没有 T 专属的 swap 存在。编译器就是用 std 内的 swap，然而即便如此。编译器还是比較喜欢 std::swap 的 T 专属特化版本号，而非一般化的那个 template。

总结：

（1）假设 swap 的缺省实现码对你的 class 或 class template 提供可接受的效率。那么我们不须要额外做不论什么事。

（2）假设 swap 缺省实现版效率不足（某种 pimpl)，那么我们就像这样做：

请记住：

（1）当 std::swap 对你的类型效率不高时, 提供一个 swap 成员函数, 并确定这个函数不抛出异常。

（2）假设你提供一个 member swap, 也该提供一个 non-member swap 用来调用前者. 对于 classes(而非 template), 也请特化 std::swap。

（3）调用 swap 时应针对 std::swap 使用 using 声明式, 然后调用 swap 而且不带不论什么 " 命名空间资格修饰 “。

（4）为 "用户定义类型" 进行 std templates 全特化是好的, 但千万不要尝试在 std 内增加某些 std 而言全新的东西。

## 条款26：尽可能延后变量定义式的出现时间

尽量延后变量定义的时间，因为可以减少中间中断/返回导致的无意义的构造&析构，同时由于“直接在构造函数时指定初值”比 “通过 default 构造函数构造出一个对象然后对它赋值” 效率更高。所以我们希望能够尽量延迟创建对象，即通过前面提到的RAII来减少不必要的消耗。

## 条款27：尽量减少转型动作

C风格的两种“旧式转型”：

```jsx
(T)expression
T(expression)
```

C++风格的四种“新式转型”：

- **const_cast**:通常用来将对象的常量性移除
- **dynamic_cast**:主要用来执行安全向下转型(基类转派生类)，也就是用来决定某个对象是否归属继承体系中的某个类型.
- **reinterpret_cast**:意图执行低级转型，比如将一个int*转换成一个int
- **static_cast**:用来强迫隐式转换，比如将 no-const 对象转换为 const 对象，将 int 转为 double 等等.

旧式转型仍然合法，但新式转型比较受欢迎. 原因是:**1. 它们很容易在代码中被辨识出来2. 转型动作的目标越窄化，编译器越可能诊断出错误的运用。比如: 如果你打算将常量性去掉，除非使用 const-cast 否则无法通过编译.**

很多人认为，转型其实什么都没做，只是告诉编译器把某种类型视为另一种类型. 这是错误的观念. 任何一个类型转换往往真的令编译器编译出运行期间执行的代码.

```jsx
int x, y;
double d = static_cast<double>(x) / y;
```

将 int x 转型为 double 肯定会产生一些代码。这或许不会让你惊讶，但下面的例子就很不同了:

```jsx
class Base{...};
class Derived: public Base{...};
Derived d;
Base * pb = &d;
```

这里只不过是建立一个基类指针指向派生类对象，但有时上诉的两个指针值并不相同。这种情况下会有**偏移量**在运行期被施行于派生类身上，用来获取正确的基类指针值.

也就是说，单一对象比如`Derived d`可能拥有一个以上的地址，当`base*`指向它的地址和`Derived*`指向它的地址。同时，我们应该减少对于对象在内存中布局的猜测，因为对象的布局方式与地址计算方式随着编译器的不同而不同。

很多应用框架都要求派生类内的虚函数代码的第一个动作就先调用基类对应的函数，比如：

```jsx
class window
{
public:
	virtual void OnResize(){...}
	...
};
class SpecialWindow:public Window
{
public:
	virtual void OnResize()
	{
		static_cast<Window>(*this).onResize();
	}
	
};
```

在`SpecialWindow`中进行了转型，为了调用`Window`的`onResize()`.如你预料的意义，这段程序将`*this`转型为`Window`, 但恐怕你没想到的是，**这个函数它调用的并不是当前对象上的函数，而是转型动作建立的一个 * this 对象之基类成分的暂时副本的 onResize.**也就是说，在一个副本上调用了`Window::onResize`, 然后在当前对象上执行 SpecialWindow 专属动作。如果`Window::onResize`修改了对象内容，当前对象其实也没被改动，改动的是副本，然而如果`SpecialWindow::onResize`内如果也修改了对象，当前对象真的会被改动，也就导致了没有改动完全：**基类没有改动，而派生类却改动成功。**

解决办法就是：直接调用 Window 的 onResize()

```jsx
	virtual void OnResize()
	{
		Window::onResize();
		...
	}
```

- **尽量避免转型，特别是在注重效率的代码中避免 dynamic-cast.如果有个设计需要转型动作，试着发展不需要转型的替代设计**
一般来说至少我们有两种手段避免频繁使用dynamic_cast转型，1、将derived class对象的指针根据类型分容器存储。2、基类写一个缺省的virtual函数，这样通过多态，基类指针能够直接调用派生类方法

## 条款28