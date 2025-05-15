# Effective Modern C++学习笔记

## 1.类型推导

​	越学习越觉得c++类型名繁杂冗长，而且很多类型名解读很麻烦，所以c++推出两套规则，一套用于auto，一套用于decltype。但是类型推导并不总是如我们所期待的那样进行，如果对于类型推导操作没有一个扎实的理解，想要写出一个具有现代感的c++程序是很难得。

​	==这章解释了模板类型推导是如何工作的，auto是如何依赖类型推导的，以及decltype是如何按照它自己独特的规则工作。它甚至解释了你该如何强制编译器使类型推导的结果可视，这能让你确认编译器的类型推导是否按照你期望的那样进行。==

### 1.1理解模板类型推导

==类型推导和实际不是模板的参数匹配无关， 不要把自己学晕了==

​	auto是建立在模板推导的基础上的。现在我们来理解auto。看一下伪代码

```c++
template <typename T>
void f(ParamType param);
```

```c++
f(expr) //调用
```

在编译期间，编译器使用expr进行两个推导，一个是针对T的也就是模板参数推导，另一个是针对ParamType的。这两个类型通常是不同的，因为ParamType包含一些修辞，比如const和引用修饰符。举个例子

```c++
template <typename T>
void f(const T & param); //这个模板函数的类型是指对const T 类型起别名
```

然后这样进行调用：

```c++
int x = 0;
f(x);
```

T被推导为int,而ParamType 则是指函数调用的参数类型：const int &

我们可能期待T 和传进去的实参是相同的类型，也就是，T 为expr的类型。在上面的例子中，事实就是那样：x是int，T被推导为int。但是有时情况并非总是如此，==T的类型推导不仅取决于expr的类型，也取决于ParamType的类型==。这里有三种情况：

* `ParamType`是一个指针或引用，但不是通用引用（关于通用引用请参见[Item24](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item24.html)。在这里你只需要知道它存在，而且不同于左值引用和右值引用）
* `ParamType`是一个通用引用
* `ParamType`既不是指针也不是引用

我们拿三种情况进行讨论

```c++
template<typename T>
void f(ParamType param);

f(expr);                        //从expr中推导T和ParamType
```

#### 1.1.1ParamType是一个指针或引用，但不是通用引用

在这种情况下，类型推导会这样进行：

1. 如果`expr`的类型是一个引用，忽略引用部分
2. 然后`expr`的类型与`ParamType`进行模式匹配来决定`T`

举个例子，如果这是我们的模板，

```c++
template<typename T>
void f(T& param);               //param是一个引用
```

我们声明这些变量，

```c++
int x=27;                       //x是int
const int cx=x;                 //cx是const int
const int& rx=x;                //rx是指向作为const int的x的引用
```

在不同的调用中，对`param`和`T`推导的类型会是这样：

```c++
f(x);                           //T是int，param的类型是int&
f(cx);                          //T是const int，param的类型是const int&
f(rx);                          //T是const int，param的类型是const int&
```

​	当我们把一个const对象传递给一个引用的形参，他们期待对象保持不可改变性，也就是说形参是reference-to-const指向不可变对象的引用。所以我们也可以传递给形参一个const对象。常量性也会保留为T的一部分。

​	当传入的参数是const引用时，T会被推导为非引用，这是因为rx的引用性在类型推导中会被忽略。假如我们把形参类型改为const T &时， cx和rx的const仍然会被传入形参，但是我们把形参假设为const类型的引用，所以T的类型推导也会忽略const。

​		如果ParamType类型是指针，情况也是一样的。他会在类型推导里面忽略掉指针或者const。

**总结：**
	形参带引用时，会忽略掉实参的引用，而且形参是const的话也会忽略掉实参的const。

#### 1.1.2ParamType是一个通用引用

​	通用引用就是我们所说的万能引用，也就是完美转发，参数传入的类型是什么，T推导出来的类型就是什么。这样的形参被声明为像右值引用一样（也就是，在函数模板中假设有一个类型形参`T`，那么通用引用声明形式就是`T&&`)，它们的行为在传入左值实参时大不相同。完整的叙述请参见[Item24](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item24.html)，在这有些最必要的你还是需要知道：

- 如果`expr`是左值，`T`和`ParamType`都会被推导为左值引用。这非常不寻常，第一，这是模板类型推导中唯一一种`T`被推导为引用的情况。第二，虽然`ParamType`被声明为右值引用类型，但是最后推导的结果是左值引用。
- 如果`expr`是右值，就使用正常的（也就是**情景一**）推导规则

#### 1.1.3ParamType既不是指针也不是引用

​	当ParamType即不是指针也不是引用时，我们通过值发传递的方式处理：

```c++
template<typename T>
void f(T param);                //以传值的方式处理param
```

这说明无论传递什么实参进来，都只是把值拷贝进来，这也会影响T的类型推导。

1. 和之前一样，如果`expr`的类型是一个引用，忽略这个引用部分
2. 如果忽略`expr`的引用性（reference-ness）之后，`expr`是一个`const`，那就再忽略`const`。如果它是`volatile`，也忽略`volatile`（`volatile`对象不常见，它通常用于驱动程序的开发中。关于`volatile`的细节请参见[Item40](https://cntransgroup.github.io/EffectiveModernCppChinese/7.TheConcurrencyAPI/item40.html)）

因此

```cpp
int x=27;                       //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;              //如之前一样

f(x);                           //T和param的类型都是int
f(cx);                          //T和param的类型都是int
f(rx);                          //T和param的类型都是int
```

注意即使`cx`和`rx`表示`const`值，`param`也不是`const`。这是有意义的。`param`是一个完全独立于`cx`和`rx`的对象——是`cx`或`rx`的一个拷贝。具有常量性的`cx`和`rx`不可修改并不代表`param`也是一样。这就是为什么`expr`的常量性`const`ness（或易变性`volatile`ness)在推导`param`类型时会被忽略：==因为`expr`不可修改并不意味着它的拷贝也不能被修改，也不代表拷贝也是一个左值引用类型。==

​	所以我们认识到只有值传递的时候才会忽略掉const ness,这一点也是理解为什么只有形参是引用时才会保留const属性。举个例子`expr`是一个`const`指针，指向`const`对象，`expr`通过传值传递给`param`：

```cpp
template<typename T>
void f(T param);                //仍然以传值的方式处理param

const char* const ptr =         //ptr是一个常量指针，指向常量对象 
    "Fun with pointers";

f(ptr);                         //传递const char * const类型的实参
```

在这里，解引用符号（*）的右边的`const`表示`ptr`本身是一个`const`：`ptr`不能被修改为指向其它地址，也不能被设置为null（解引用符号左边的`const`表示`ptr`指向一个字符串，这个字符串是`const`，因此字符串不能被修改）。当`ptr`作为实参传给`f`，组成这个指针的每一比特都被拷贝进`param`。像这种情况，==`ptr`**自身的值会被传给形参**==，根据类型推导的第三条规则，==`ptr`自身的常量性`const`ness将会被省略==，所以`param`是`const char*`，也就是一个可变指针指向`const`字符串。在类型推导中，这个指针指向的数据的常量性`const`ness将会被保留，但是当拷贝`ptr`来创造一个新指针`param`时，`ptr`自身的常量性`const`ness将会被忽略。

**数组做实参**

​	上面的类型推导已经涵盖了很大一部分，但是还有个值得注意的地方就是数组类型不同于指针类型，虽然有时候可以互换。关于这个错觉最常见的例子是，在很多上下文中数组会退化为指向它的第一个元素的指针。这样的退化允许像这样的代码可以被编译：

```cpp
const char name[] = "J. P. Briggs";     //name的类型是const char[13]
const char * ptrToName = name;          //数组退化为指针
```

我们从一个简单的例子开始，这里有一个函数的形参是数组，是的，这样的语法是合法的，

```cpp
void myFunc(int param[]);
```

但是数组声明会被视作指针声明，这意味着`myFunc`的声明和下面声明是等价的：

```cpp
void myFunc(int* param);                //与上面相同的函数
```

因为数组形参会视作指针形参，所以传值给模板的一个数组类型会被推导为一个指针类型。这意味着在模板函数`f`的调用中，它的类型形参`T`会被推导为`const char*`：

```cpp
f(name);                        //name是一个数组，但是T被推导为const char*
```

但是现在难题来了，虽然函数不能声明形参为真正的数组，但是**可以**接受指向数组的**引用**！所以我们修改`f`为传引用：

```cpp
template<typename T>
void f(T& param);                       //传引用形参的模板
```

我们这样进行调用，

```cpp
f(name);                                //传数组给f
```

`T`被推导为了真正的数组！这个类型包括了数组的大小，在这个例子中`T`被推导为`const char[13]`，`f`的形参（该数组的引用）的类型则为`const char (&)[13]`。

有趣的是，可声明指向数组的引用的能力，使得我们可以创建一个模板函数来推导出数组的大小：

```cpp
//在编译期间返回一个数组大小的常量值（//数组形参没有名字，
//因为我们只关心数组的大小）
template<typename T, std::size_t N>                     //关于
constexpr std::size_t arraySize(T (&)[N]) noexcept      //constexpr
{                                                       //和noexcept
    return N;                                           //的信息
}                                                       //请看下面
```

在[Item15](https://cntransgroup.github.io/EffectiveModernCppChinese/3.MovingToModernCpp/item15.html)提到将一个函数声明为`constexpr`使得结果在编译期间可用。这使得我们可以用一个花括号声明一个数组，然后第二个数组可以使用第一个数组的大小作为它的大小，就像这样：

```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };             //keyVals有七个元素

int mappedVals[arraySize(keyVals)];                     //mappedVals也有七个
```

当然作为一个现代C++程序员，你自然应该想到使用`std::array`而不是内置的数组：

```cpp
std::array<int, arraySize(keyVals)> mappedVals;         //mappedVals的大小为7
```

至于`arraySize`被声明为`noexcept`，会使得编译器生成更好的代码，具体的细节请参见[Item14](https://cntransgroup.github.io/EffectiveModernCppChinese/3.MovingToModernCpp/item14.html)。

**函数做实参**

在C++中不只是数组会退化为指针，函数类型也会退化为一个函数指针，我们对于数组类型推导的全部讨论都可以应用到函数类型推导和退化为函数指针上来。结果是：

```cpp
void someFunc(int, double);         //someFunc是一个函数，
                                    //类型是void(int, double)

template<typename T>
void f1(T param);                   //传值给f1

template<typename T>
void f2(T & param);                 //传引用给f2

f1(someFunc);                       //param被推导为指向函数的指针，
                                    //类型是void(*)(int, double)
f2(someFunc);                       //param被推导为指向函数的引用，
                                    //类型是void(&)(int, double)
```

这个实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。

这里你需要知道：`auto`依赖于模板类型推导。正如我在开始谈论的，在大多数情况下它们的行为很直接。在通用引用中对于左值的特殊处理使得本来很直接的行为变得有些污点，然而，数组和函数退化为指针把这团水搅得更浑浊。有时你只需要编译器告诉你推导出的类型是什么。这种情况下，翻到[item4](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item4.html),它会告诉你如何让编译器这么做。

**请记住：**（总结）

- 在模板类型推导时，有引用的实参会被视为无引用，他们的引用会被忽略
- 对于通用引用的推导，左值实参会被特殊对待
- 对于传值类型推导，`const`和/或`volatile`实参会被认为是non-`const`的和non-`volatile`的
- 在模板类型推导时，数组名或者函数名实参会退化为指针，除非它们被用于初始化引用

### 1.2理解auto类型推导

​	没啥可以看的

- `auto`类型推导通常和模板类型推导相同，但是`auto`类型推导假定花括号初始化代表`std::initializer_list`，而模板类型推导不这样做
- 在C++14中`auto`允许出现在函数返回值或者*lambda*函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是`auto`类型推导

---

**补充：**

一、普通变量声明中的auto推导：

- 自动忽略顶层const和引用
- 数组会退化为指针
- 初始化列表{}推导为std::initializer_list

二、模板类型推导：

- 不会自动推导std::initializer_list
- 保留const和引用修饰符
- 数组不退化为指针

---

### 1.3理解decltyp

#### 1.3.1使用decltype是为了推导模板返回的左值引用类型

​	decltype通常用于声明函数模板，auto也可以用于推到函数返回值，但是问题就是auto作为函数返回值会采用模板推导类型， 而模板类型推导会自动忽略引用类型，所以这是弊端，而使用decltype就不会有这种问题。

```cpp
std::deque<int> d;
…
authAndAccess(d, 5) = 10;               //认证用户，返回d[5]，
                                        //然后把10赋值给它
                                        //无法通过编译！
```

​	

​	为什么需要推导引用类型呢？在C++11中，`decltype`最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型。举个例子，假定我们写一个函数，一个形参为容器，一个形参为索引值，这个函数支持使用方括号的方式（也就是使用“`[]`”）访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。函数的返回类型应该和索引操作返回的类型相同。

对一个`T`类型的容器使用`operator[]` 通常会返回一个`T&`对象，比如`std::deque`就是这样。但是`std::vector`有一个例外，对于`std::vector<bool>`，`operator[]`不会返回`bool&`，它会返回一个全新的对象（译注：MSVC的STL实现中返回的是`std::_Vb_reference<std::_Wrap_alloc<std::allocator<unsigned int>>>`对象）。关于这个问题的详细讨论请参见[Item6](https://cntransgroup.github.io/EffectiveModernCppChinese/2.Auto/item6.html)，这里重要的是我们可以看到对一个容器进行`operator[]`操作返回的类型取决于容器本身。

```cpp
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```

函数名称前面的`auto`不会做任何的类型推导工作。相反的，他只是暗示使用了C++11的**尾置返回类型**语法，即在函数形参列表后面使用一个”`->`“符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使用函数形参相关的信息。在`authAndAccess`函数中，我们使用`c`和`i`指定返回类型。如果我们按照传统语法把函数返回类型放在函数名称之前，`c`和`i`就未被声明所以不能使用。==尾置类型的好处==

在这种声明中，`authAndAccess`函数返回`operator[]`应用到容器中返回的对象的类型，这也正是我们期望的结果。

C++11允许自动推导单一语句的*lambda*表达式的返回类型， C++14扩展到允许自动推导所有的*lambda*表达式和函数，甚至它们内含多条语句。对于`authAndAccess`来说这意味着在C++14标准下我们可以忽略尾置返回类型，只留下一个`auto`。使用这种声明形式，auto标示这里会发生类型推导。更准确的说，编译器将会从函数实现中推导出函数的返回类型。

```cpp
template<typename Container, typename Index>    //C++14版本，
auto authAndAccess(Container& c, Index i)       //不那么正确
{
    authenticateUser();
    return c[i];                                //从c[i]中推导返回类型
}
```

要想让`authAndAccess`像我们期待的那样工作，我们需要使用`decltype`类型推导来推导它的返回值，即指定`authAndAccess`应该返回一个和`c[i]`表达式类型一样的类型。C++期望在某些情况下当类型被暗示时需要使用`decltype`类型推导的规则，C++14通过使用`decltype(auto)`说明符使得这成为可能。我们第一次看见`decltype(auto)`可能觉得非常的矛盾（到底是`decltype`还是`auto`？），实际上我们可以这样解释它的意义：`auto`说明符表示这个类型将会被推导，`decltype`说明`decltype`的规则将会被用到这个推导过程中。因此我们可以这样写`authAndAccess`：

```cpp
template<typename Container, typename Index>    //C++14版本，
decltype(auto)                                  //可以工作，
authAndAccess(Container& c, Index i)            //但是还需要
{                                               //改良
    authenticateUser();
    return c[i];
}
```

现在`authAndAccess`将会真正的返回`c[i]`的类型。现在事情解决了，一般情况下`c[i]`返回`T&`，`authAndAccess`也会返回`T&`，特殊情况下`c[i]`返回一个对象，`authAndAccess`也会返回一个对象。

==`decltype(auto)`的使用不仅仅局限于函数返回类型，当你想对初始化表达式使用`decltype`推导的规则==，你也可以使用：

```cpp
Widget w;

const Widget& cw = w;

auto myWidget1 = cw;                    //auto类型推导
                                        //myWidget1的类型为Widget
decltype(auto) myWidget2 = cw;          //decltype类型推导
                                        //myWidget2的类型是const Widget&
```

但是这里有两个问题困惑着你。一个是我之前提到的`authAndAccess`的改良至今都没有描述。让我们现在加上它。

再看看C++14版本的`authAndAccess`声明：

```cpp
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);
```

容器通过传引用的方式传递非常量左值引用（lvalue-reference-to-non-**const**），因为返回一个引用允许用户可以修改容器。但是这意味着在不能给这个函数传递右值容器，右值不能被绑定到左值引用上（除非这个左值引用是一个const（lvalue-references-to-**const**），但是这里明显不是）。

公认的向`authAndAccess`传递一个右值是一个[edge case](https://en.wikipedia.org/wiki/Edge_case)（译注：在极限操作情况下会发生的事情，类似于会发生但是概率较小的事情）。一个右值容器，是一个临时对象，通常会在`authAndAccess`调用结束被销毁，这意味着`authAndAccess`返回的引用将会成为一个悬置的（dangle）引用。但是使用向`authAndAccess`传递一个临时变量也并不是没有意义，有时候用户可能只是想简单的获得临时容器中的一个元素的拷贝，比如这样：

```cpp
std::deque<std::string> makeStringDeque();      //工厂函数

//从makeStringDeque中获得第五个元素的拷贝并返回
auto s = authAndAccess(makeStringDeque(), 5);
```

要想支持这样使用`authAndAccess`我们就得修改一下当前的声明使得它支持左值和右值。重载是一个不错的选择（一个函数重载声明为左值引用，另一个声明为右值引用），但是我们就不得不维护两个重载函数。另一个方法是使`authAndAccess`的引用可以绑定左值和右值，[Item24]解释了那正是通用引用能做的，所以我们这里可以使用通用引用进行声明：

```cpp
template<typename Containter, typename Index>   //现在c是通用引用
decltype(auto) authAndAccess(Container&& c, Index i);
```

在这个模板中，我们不知道我们操纵的容器的类型是什么，那意味着我们同样不知道它使用的索引对象（index objects）的类型，对一个未知类型的对象使用传值通常会造成不必要的拷贝，对程序的性能有极大的影响，还会造成对象切片行为（参见[item41](https://cntransgroup.github.io/EffectiveModernCppChinese/8.Tweaks/item41.html)），以及给同事落下笑柄。但是就容器索引来说，我们遵照标准模板库对于索引的处理是有理由的（比如`std::string`，`std::vector`和`std::deque`的`operator[]`），所以我们坚持传值调用。

然而，我们还需要更新一下模板的实现，让它能听从[Item25](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item25.html)的告诫应用`std::forward`实现通用引用：

```cpp
template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

这样就能对我们的期望交上一份满意的答卷，但是这要求编译器支持C++14。如果你没有这样的编译器，你还需要使用C++11版本的模板，它看起来和C++14版本的极为相似，除了你不得不指定函数返回类型之外：

```cpp
template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

另一个问题是就像我在条款的开始唠叨的那样，`decltype`通常会产生你期望的结果，但并不总是这样。在**极少数情况下**它产生的结果可能让你很惊讶。老实说如果你不是一个大型库的实现者你不太可能会遇到这些异常情况。

为了**完全**理解`decltype`的行为，你需要熟悉一些特殊情况。它们大多数都太过晦涩以至于几乎没有书进行有过权威的讨论，这本书也不例外，但是其中的一个会让我们更加理解`decltype`的使用。

将`decltype`应用于变量名会产生该变量名的声明类型。虽然变量名都是左值表达式，但这不会影响`decltype`的行为。（译者注：这里是说对于单纯的变量名，`decltype`只会返回变量的声明类型）然而，对于比单纯的变量名更复杂的左值表达式，`decltype`可以确保报告的类型始终是左值引用。也就是说，如果一个不是单纯变量名的左值表达式的类型是`T`，那么`decltype`会把这个表达式的类型报告为`T&`。这几乎没有什么太大影响，因为大多数左值表达式的类型天生具备一个左值引用修饰符。例如，返回左值的函数总是返回左值引用。

这个行为暗含的意义值得我们注意，在：

```cpp
int x = 0;
```

中，`x`是一个变量的名字，所以`decltype(x)`是`int`。但是如果用一个小括号包覆这个名字，比如这样`(x)` ，就会产生一个比名字更复杂的表达式。对于名字来说，`x`是一个左值，C++11定义了表达式`(x)`也是一个左值。因此`decltype((x))`是`int&`。用小括号覆盖一个名字可以改变`decltype`对于名字产生的结果。

在C++11中这稍微有点奇怪，但是由于C++14允许了`decltype(auto)`的使用，这意味着你在函数返回语句中细微的改变就可以影响类型的推导：

```cpp
decltype(auto) f1()
{
    int x = 0;
    …
    return x;                            //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);                          //decltype((x))是int&，所以f2返回int&
}
```

注意不仅`f2`的返回类型不同于`f1`，而且它还引用了一个局部变量！这样的代码将会把你送上未定义行为的特快列车，一辆你绝对不想上第二次的车。

当使用`decltype(auto)`的时候一定要加倍的小心，在表达式中看起来无足轻重的细节将会影响到`decltype(auto)`的推导结果。为了确认类型推导是否产出了你想要的结果，请参见[Item4](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item4.html)描述的那些技术。

同时你也不应该忽略`decltype`这块大蛋糕。没错，`decltype`（单独使用或者与`auto`一起用）可能会偶尔产生一些令人惊讶的结果，但那毕竟是少数情况。通常，`decltype`都会产生你想要的结果，尤其是当你对一个变量使用`decltype`时，因为在这种情况下，`decltype`只是做一件本分之事：它产出变量的声明类型。

**请记住：**

- `decltype`总是不加修改的产生变量或者表达式的类型。
- 对于`T`类型的不是单纯的变量名的左值表达式，`decltype`总是产出`T`的引用即`T&`。
- C++14支持`decltype(auto)`，就像`auto`一样，推导出类型，但是它使用`decltype`的规则进行推导。

### 1.4学会查看类型推导结果

选择什麼工具查看类型推导，取决于软件开发过程中你想在哪个阶段显示类型推导信息。我们探究三种方案：在你编辑代码的时候获得类型推导的结果，在编译期间获得结果，在运行时获得结果。

#### **1.4.1IDE编辑器**

在IDE中的代码编辑器通常可以显示程序代码中变量，函数，参数的类型，你只需要简单的把鼠标移到它们的上面，举个例子，有这样的代码中：

```cpp
const int theAnswer = 42;

auto x = theAnswer;
auto y = &theAnswer;
```

IDE编辑器可以直接显示`x`推导的结果为`int`，`y`推导的结果为`const int*`。

为此，你的代码必须或多或少的处于可编译状态，因为IDE之所以能提供这些信息是因为一个C++编译器（或者至少是前端中的一个部分）运行于IDE中。如果这个编译器对你的代码不能做出有意义的分析或者推导，它就不会显示推导的结果。

对于像`int`这样简单的推导，IDE产生的信息通常令人很满意。正如我们将看到的，如果更复杂的类型出现时，IDE提供的信息就几乎没有什么用了。

#### **1.4.2编译器诊断**

另一个获得推导结果的方法是使用编译器出错时提供的错误消息。这些错误消息无形的提到了造成我们编译错误的类型是什么。

举个例子，假如我们想看到之前那段代码中`x`和`y`的类型，我们可以首先声明一个类模板但**不定义**。就像这样：

```cpp
template<typename T>                //只对TD进行声明
class TD;                           //TD == "Type Displayer"
```

如果尝试实例化这个类模板就会引出一个错误消息，因为这里没有用来实例化的类模板定义。为了查看`x`和`y`的类型，只需要使用它们的类型去实例化`TD`：

```cpp
TD<decltype(x)> xType;              //引出包含x和y
TD<decltype(y)> yType;              //的类型的错误消息
```

我使用***variableName*****Type**的结构来命名变量，因为这样它们产生的错误消息可以有助于我们查找。对于上面的代码，我的编译器产生了这样的错误信息，我取一部分贴到下面：

```cpp
error: aggregate 'TD<int> xType' has incomplete type and 
        cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and
        cannot be defined
```

另一个编译器也产生了一样的错误，只是格式稍微改变了一下：

```cpp
error: 'xType' uses undefined class 'TD<int>'
error: 'yType' uses undefined class 'TD<const int *>'
```

除了格式不同外，几乎所有我测试过的编译器都产生了这样有用的错误消息。

#### **1.4.3运行时输出**

使用`printf`的方法（并不是说我推荐你使用`printf`）类型信息要在运行时才会显示出来，但是它提供了一种格式化输出的方法。现在唯一的问题是对于你关心的变量使用一种优雅的文本表示。“这有什么难的，“你这样想，”这正是`typeid`和`std::type_info::name`的价值所在”。为了实现我们想要查看`x`和`y`的类型的需求，你可能会这样写：

```cpp
std::cout << typeid(x).name() << '\n';  //显示x和y的类型
std::cout << typeid(y).name() << '\n';
```

这种方法对一个对象如`x`或`y`调用`typeid`产生一个`std::type_info`的对象，然后`std::type_info`里面的成员函数`name()`来产生一个C风格的字符串（即一个`const char*`）表示变量的名字。

调用`std::type_info::name`不保证返回任何有意义的东西，但是库的实现者尝试尽量使它们返回的结果有用。实现者们对于“有用”有不同的理解。举个例子，GNU和Clang环境下`x`的类型会显示为”`i`“，`y`会显示为”`PKi`“，这样的输出你必须要问问编译器实现者们才能知道他们的意义：”`i`“表示”`int`“，”`PK`“表示”pointer to ~~`konst`~~ `const`“（指向常量的指针）。（这些编译器都提供一个工具`c++filt`，解释这些“混乱的”类型）Microsoft的编译器输出得更直白一些：对于`x`输出”`int`“对于`y`输出”`int const *`“

因为对于`x`和`y`来说这样的结果是正确的，你可能认为问题已经接近了，别急，考虑一个更复杂的例子：

```cpp
template<typename T>                    //要调用的模板函数
void f(const T& param);

std::vector<Widget> createVec();        //工厂函数

const auto vw = createVec();            //使用工厂函数返回值初始化vw

if (!vw.empty()){
    f(&vw[0]);                          //调用f
    …
}
```

在这段代码中包含了一个用户定义的类型`Widget`，一个STL容器`std::vector`和一个`auto`变量`vw`，这个更现实的情况是你可能会遇到的并且想获得他们类型推导的结果，比如模板类型形参`T`，比如函数`f`形参`param`。

从这里中我们不难看出`typeid`的问题所在。我们在`f`中添加一些代码来显示类型：

```cpp
template<typename T>
void f(const T& param)
{
    using std::cout;
    cout << "T =     " << typeid(T).name() << '\n';             //显示T

    cout << "param = " << typeid(param).name() << '\n';         //显示
    …                                                           //param
}                                                               //的类型
```

GNU和Clang执行这段代码将会输出这样的结果

```cpp
T =     PK6Widget
param = PK6Widget
```

我们早就知道在这些编译器中`PK`表示“pointer to `const`”，所以只有数字`6`对我们来说是神奇的。其实数字是类名称（`Widget`）的字符串长度，所以这些编译器告诉我们`T`和`param`都是`const Widget*`。

Microsoft的编译器也同意上述言论：

```cpp
T =     class Widget const *
param = class Widget const *
```

三个独立的编译器都产生了相同的信息，这表明信息应该是准确的。但仔细观察一下，在模板`f`中，`param`的声明类型是`const T&`。难道你们不觉得`T`和`param`类型相同很奇怪吗？比如`T`是`int`，`param`的类型应该是`const int&`而不是相同类型才对吧。

遗憾的是，事实就是这样，`std::type_info::name`的结果并不总是可信的，就像上面一样，三个编译器对`param`的报告都是错误的。此外，它们在本质上必须是这样的结果，因为`std::type_info::name`规范批准像传值形参一样来对待这些类型。正如[Item1](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item1.html)提到的，如果传递的是一个引用，那么引用部分（reference-ness）将被忽略，如果忽略后还具有`const`或者`volatile`，那么常量性`const`ness或者易变性`volatile`ness也会被忽略。那就是为什么`param`的类型`const Widget * const &`会输出为`const Widget *`，首先引用被忽略，然后这个指针自身的常量性`const`ness被忽略，剩下的就是指针指向一个常量对象。

同样遗憾的是，IDE编辑器显示的类型信息也不总是可靠的，或者说不总是有用的。还是一样的例子，一个IDE编辑器可能会把`T`的类型显示为（我没有胡编乱造）：

```cpp
const
std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget,
std::allocator<Widget>>::_Alloc>::value_type>::value_type *
```

同样把`param`的类型显示为

```cpp
const std::_Simple_types<...>::value_type *const &
```

这个比起`T`来说要简单一些，但是如果你不知道“`...`”表示编译器忽略`T`的部分类型那么可能你还是会产生困惑。如果你运气好点你的IDE可能表现得比这个要好一些。

比起运气如果你更倾向于依赖库，那么你会很乐意被告知，在`std::type_info::name`和IDE失效的地方，Boost TypeIndex库（通常写作**Boost.TypeIndex**）被设计成可以正常运作。这个库不是标准C++的一部分，也不是IDE或者`TD`这样的模板。Boost库（可在[boost.com](http://boost.org/)获得）是跨平台，开源，有良好的开源协议的库，这意味着使用Boost和STL一样具有高度可移植性。

这里是如何使用Boost.TypeIndex得到`f`的类型的代码

```cpp
#include <boost/type_index.hpp>

template<typename T>
void f(const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;

    //显示T
    cout << "T =     "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';
    
    //显示param类型
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
}
```

`boost::typeindex::type_id_with_cvr`获取一个类型实参（我们想获得相应信息的那个类型），它不消除实参的`const`，`volatile`和引用修饰符（因此模板名中有“`with_cvr`”）。结果是一个`boost::typeindex::type_index`对象，它的`pretty_name`成员函数输出一个`std::string`，包含我们能看懂的类型表示。 基于这个`f`的实现版本，再次考虑那个使用`typeid`时获取`param`类型信息出错的调用：

```cpp
std::vetor<Widget> createVec();         //工厂函数
const auto vw = createVec();            //使用工厂函数返回值初始化vw
if (!vw.empty()){
    f(&vw[0]);                          //调用f
    …
}
```

在GNU和Clang的编译器环境下，使用Boost.TypeIndex版本的`f`最后会产生下面的（准确的）输出：

```cpp
T =     Widget const *
param = Widget const * const&
```

在Microsoft的编译器环境下，结果也是极其相似：

```cpp
T =     class Widget const *
param = class Widget const * const &
```

这样近乎一致的结果是很不错的，但是请记住IDE，编译器错误诊断或者像Boost.TypeIndex这样的库只是用来帮助你理解编译器推导的类型是什么。它们是有用的，但是作为本章结束语我想说它们根本不能替代你对[Item1](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item1.html)-[3](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item3.html)提到的类型推导的理解。

**请记住：**

- 类型推断可以从IDE看出，从编译器报错看出，从Boost TypeIndex库的使用看出
- 这些工具可能既不准确也无帮助，所以理解C++类型推导规则才是最重要的

## 2.auto

### 2.1优先考虑auto而非显示类型声明

​	从C++11开始我们可以用auto来表示很多很难写出来的类型（闭包类型，只有编译器才知道的类型）比如模板中迭代器解引用才能表达出来的

```cpp
template<typename It>           //对从b到e的所有元素使用
void dwim(It b, It e)           //dwim（“do what I mean”）算法
{
    while (b != e) {
        typename std::iterator_traits<It>::value_type
        currValue = *b;
        …
    }
}
```

但是有了auto就不需要这样了。auto甚至可以表示一些只有编译器才知道的类型：

```cpp
auto derefUPLess = 
    [](const std::unique_ptr<Widget> &p1,       //用于std::unique_ptr
       const std::unique_ptr<Widget> &p2)       //指向的Widget类型的
    { return *p1 < *p2; };                      //比较函数
```

很酷对吧，如果使用C++14，将会变得更酷，因为*lambda*表达式中的形参也可以使用`auto`：

```cpp
auto derefLess =                                //C++14版本
    [](const auto& p1,                          //被任何像指针一样的东西
       const auto& p2)                          //指向的值的比较函数
    { return *p1 < *p2; };
```

尽管这很酷，但是你可能会想我们完全不需要使用`auto`声明局部变量来保存一个闭包，因为我们可以使用`std::function`对象。

**function对象：**

​	std::function是一个C++11标准库中一个模板，它==泛化==了函数指针的概念。让它可以指向任何可调用对象。当你声明函数指针时你必须指定函数类型（即函数签名），同样当你创建`std::function`对象时你也需要提供函数签名，由于它是一个模板所以你需要在它的模板参数里面提供。举个例子，假设你想声明一个`std::function`对象`func`使它指向一个可调用对象，比如一个具有这样函数签名的函数

```cpp
bool(const std::unique_ptr<Widget> &,           //C++11
     const std::unique_ptr<Widget> &)           //std::unique_ptr<Widget>
                                                //比较函数的签名
```

你就得这么写：

```cpp
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)> func;
```

因为*lambda*表达式能产生一个可调用对象，所以我们现在可以把闭包存放到`std::function`对象中。这意味着我们可以不使用`auto`写出C++11版的`derefUPLess`：

```cpp
std::function<bool(const std::unique_ptr<Widget> &,
                   const std::unique_ptr<Widget> &)>
derefUPLess = [](const std::unique_ptr<Widget> &p1,
                 const std::unique_ptr<Widget> &p2)
                { return *p1 < *p2; };
```

用function和auto相比有很多缺点。

1. 消耗更多存储空间，且可能触发out-of-memory异常。
2. 而且function更慢。

**原因**：用`auto`声明的变量保存一个和闭包一样类型的（新）闭包，因此使用了与闭包相同大小存储空间。实例化`std::function`并声明一个对象这个对象将会有固定的大小。这个大小可能不足以存储一个闭包，这个时候`std::function`的构造函数将会在堆上面分配内存来存储，这就造成了使用`std::function`比`auto`声明变量会消耗更多的内存。

**auto可以避免一些移植时发生的问题**：

- 类型快捷方式（type shortcuts）有关的问题：

  - ```cpp
    std::vector<int> v;
    …
    unsigned sz = v.size();
    ```

    `v.size()`的标准返回类型是`std::vector<int>::size_type`，但是只有少数开发者意识到这点。`std::vector<int>::size_type`实际上被指定为无符号整型，所以很多人都认为用`unsigned`就足够了，写下了上述的代码。这会造成一些有趣的结果。举个例子，在**Windows 32-bit**上`std::vector<int>::size_type`和`unsigned`是一样的大小，但是在**Windows 64-bit**上`std::vector<int>::size_type`是64位，`unsigned`是32位。这意味着这段代码在Windows 32-bit上正常工作，但是当把应用程序移植到Windows 64-bit上时就可能会出现一些问题。谁愿意花时间处理这些细枝末节的问题呢？

    所以使用`auto`可以确保你不需要浪费时间：

    ```cpp
    auto sz =v.size();                      //sz的类型是std::vector<int>::size_type
    ```

    你还是不相信使用`auto`是多么明智的选择？考虑下面的代码：

    ```cpp
    std::unordered_map<std::string, int> m;
    …
    
    for(const std::pair<std::string, int>& p : m)
    {
        …                                   //用p做一些事
    }
    ```

    看起来好像很合情合理的表达，但是这里有一个问题，你看到了吗？

    要想看到错误你就得知道`std::unordered_map`的*key*是`const`的，所以*hash table*（`std::unordered_map`本质上的东西）中的`std::pair`的类型不是`std::pair<std::string, int>`，而是`std::pair<const std::string, int>`。但那不是在循环中的变量`p`声明的类型。编译器会努力的找到一种方法把`std::pair<const std::string, int>`（即*hash table*中的东西）转换为`std::pair<std::string, int>`（`p`的声明类型）。它会成功的，因为它会通过拷贝`m`中的对象创建一个临时对象，这个临时对象的类型是`p`想绑定到的对象的类型，即`m`中元素的类型，然后把`p`的引用绑定到这个临时对象上。在每个循环迭代结束时，临时对象将会销毁，如果你写了这样的一个循环，你可能会对它的一些行为感到非常惊讶，因为你确信你只是让成为`p`指向`m`中各个元素的引用而已。

    使用`auto`可以避免这些很难被意识到的类型不匹配的错误：

    ```cpp
    for(const auto& p : m)
    {
        …                                   //如之前一样
    }
    ```

    这样无疑更具效率，且更容易书写。而且，这个代码有一个非常吸引人的特性，如果你获取`p`的地址，你确实会得到一个指向`m`中元素的指针。在没有`auto`的版本中`p`会指向一个临时变量，这个临时变量在每次迭代完成时会被销毁。

    前面这两个例子——应当写`std::vector<int>::size_type`时写了`unsigned`，应当写`std::pair<const std::string, int>`时写了`std::pair<std::string, int>`——说明了显式的指定类型可能会导致你不想看到的类型转换。如果你使用`auto`声明目标变量你就不必担心这个问题。

    基于这些原因我建议你优先考虑`auto`而非显式类型声明。然而`auto`也不是完美的。每个`auto`变量都从初始化表达式中推导类型，有一些表达式的类型和我们期望的大相径庭。关于在哪些情况下会发生这些问题，以及你可以怎么解决这些问题我们在[Item2](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item2.html)和[6](https://cntransgroup.github.io/EffectiveModernCppChinese/2.Auto/item6.html)讨论，所以这里我不再赘述。我想把注意力放到你可能关心的另一点：使用auto代替传统类型声明对源码可读性的影响。

    首先，深呼吸，放松，`auto`是**可选项**，不是**命令**，在某些情况下如果你的专业判断告诉你使用显式类型声明比`auto`要更清晰更易维护，那你就不必再坚持使用`auto`。但是要牢记，C++没有在其他众所周知的语言所拥有的类型推导（*type inference*）上开辟新土地。其他静态类型的过程式语言（如C#、D、Sacla、Visual Basic）或多或少都有等价的特性，更不必提那些静态类型的函数式语言了（如ML、Haskell、OCaml、F#等）。在某种程度上，这是因为动态类型语言，如Perl、Python、Ruby等的成功；在这些语言中，几乎没有显式的类型声明。软件开发社区对于类型推导有丰富的经验，他们展示了在维护大型工业强度的代码上使用这种技术没有任何争议。

    一些开发者也担心使用`auto`就不能瞥一眼源代码便知道对象的类型，然而，IDE扛起了部分担子（也考虑到了[Item4](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item4.html)中提到的IDE类型显示问题），在很多情况下，少量显示一个对象的类型对于知道对象的确切类型是有帮助的，这通常已经足够了。举个例子，要想知道一个对象是容器还是计数器还是智能指针，不需要知道它的确切类型。一个适当的变量名称就能告诉我们大量的抽象类型信息。

    事实是显式指定类型通常只会引入一些微妙的错误，无论是在正确性还是效率方面。而且，如果初始化表达式的类型改变，则`auto`推导出的类型也会改变，这意味着使用`auto`可以帮助我们完成一些重构工作。举个例子，如果一个函数返回类型被声明为`int`，但是后来你认为将它声明为`long`会更好，调用它作为初始化表达式的变量会自动改变类型，但是如果你不使用`auto`你就不得不在源代码中挨个找到调用地点然后修改它们。

    **请记住：**

    - `auto`变量必须初始化，通常它可以避免一些移植性和效率性的问题，也使得重构更方便，还能让你少打几个字。
    - 正如[Item2](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item2.html)和[6](https://cntransgroup.github.io/EffectiveModernCppChinese/2.Auto/item6.html)讨论的，`auto`类型的变量可能会踩到一些陷阱。

### 2.2auto推导若非己愿，使用显示类型初始化惯用法

​	**假如我们的类是用代理类代理的，那么auto的值推导的值可能并不是我们想要的。**

一些代理类被设计于用以对客户可见。比如`std::shared_ptr`和`std::unique_ptr`。其他的代理类则或多或少不可见，比如`std::vector<bool>::reference`就是不可见代理类的一个例子，还有它在`std::bitset`的胞弟`std::bitset::reference`。

在后者的阵营（注：指不可见代理类）里一些C++库也是用了表达式模板（*expression templates*）的黑科技。这些库通常被用于提高数值运算的效率。给出一个矩阵类`Matrix`和矩阵对象`m1`，`m2`，`m3`，`m4`，举个例子，这个表达式

```cpp
Matrix sum = m1 + m2 + m3 + m4;
```

可以使计算更加高效，只需要使让`operator+`返回一个代理类代理结果而不是返回结果本身。也就是说，对两个`Matrix`对象使用`operator+`将会返回如`Sum<Matrix, Matrix>`这样的代理类作为结果而不是直接返回一个`Matrix`对象。在`std::vector<bool>::reference`和`bool`中存在一个隐式转换，同样对于`Matrix`来说也可以存在一个隐式转换允许`Matrix`的代理类转换为`Matrix`，这让表达式等号“`=`”右边能产生代理对象来初始化`sum`。（这个对象应当编码整个初始化表达式，即类似于`Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>`的东西。客户应该避免看到这个实际的类型。）

作为一个通则，==不可见的代理类通常不适用于`auto`==。==这样类型的对象的生命期通常不会设计为能活过一条语句==，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路。`std::vector<bool>::reference`就是这种情况，==我们看到违反这个基本假设将导致未定义行为==。

因此你想避开这种形式的代码：

```cpp
auto someVar = expression of "invisible" proxy class type;
```

但是你怎么能意识到你正在使用代理类？应用他们的软件不可能宣告它们的存在。它们被设计为**不可见**，至少概念上说是这样！每当你发现它们，你真的应该舍弃[Item5](https://cntransgroup.github.io/EffectiveModernCppChinese/2.Auto/item5.html)演示的`auto`所具有的诸多好处吗？

让我们首先回到如何找到它们的问题上。虽然“不可见”代理类都在程序员日常使用的雷达下方飞行，但是很多库都证明它们可以上方飞行。当你越熟悉你使用的库的基本设计理念，你的思维就会越活跃，不至于思维僵化认为代理类只能在这些库中使用。

当缺少文档的时候，可以去看看头文件。很少会出现源代码全都用代理对象，它们通常用于一些函数的返回类型，所以通常能从函数签名中看出它们的存在。这里有一份`std::vector<bool>::operator[]`的说明书：

```cpp
namespace std{                                  //来自于C++标准库
    template<class Allocator>
    class vector<bool, Allocator>{
    public:
        …
        class reference { … };

        reference operator[](size_type n);
        …
    };
}
```

假设你知道对`std::vector<T>`使用`operator[]`通常会返回一个`T&`，在这里`operator[]`不寻常的返回类型提示你它使用了代理类。==多关注你使用的接口可以暴露代理类的存在==。

实际上， 很多开发者都是在跟踪一些令人困惑的复杂问题或在单元测试出错进行调试时才看到代理类的使用。不管你怎么发现它们的，一旦看到`auto`推导了代理类的类型而不是被代理的类型，解决方案并不需要抛弃`auto`。`auto`本身没什么问题，问题是`auto`不会推导出你想要的类型。解决方案是强制使用一个不同的类型推导形式，这种方法我通常称之为显式类型初始器惯用法（*the explicitly typed initialized idiom*)。

**我们看到auto推导代理类并不需要放弃auto，只需要强制显示转换即可：**

显式类型初始器惯用法使用`auto`声明一个变量，然后对表达式强制类型转换（*cast*）得出你期望的推导结果。举个例子，我们该怎么将这个惯用法施加到`highPriority`上？

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

 这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`对象，就像之前那样，但是这个转型使得表达式类型为`bool`，然后`auto`才被用于推导`highPriority`。在运行时，对`std::vector<bool>::operator[]`返回的`std::vector<bool>::reference`执行它支持的向`bool`的转型，在这个过程中指向`std::vector<bool>`的指针已经被解引用。这就避开了我们之前的未定义行为。然后5将被用于指向*bit*的指针，`bool`值被用于初始化`highPriority`。

对于`Matrix`来说，显式类型初始器惯用法是这样的：

```cpp
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```

应用这个惯用法不限制初始化表达式产生一个代理类。它也可以用于强调你声明了一个变量类型，它的类型不同于初始化表达式的类型。举个例子，假设你有这样一个表达式计算公差值：

```cpp
double calcEpsilon();                           //返回公差值
```

`calcEpsilon`清楚的表明它返回一个`double`，但是假设你知道对于这个程序来说使用`float`的精度已经足够了，而且你很关心`double`和`float`的大小。你可以声明一个`float`变量储存`calEpsilon`的计算结果。

```cpp
float ep = calcEpsilon();                       //double到float隐式转换
```

但是这几乎没有表明“我确实要减少函数返回值的精度”。使用显式类型初始器惯用法我们可以这样：

```cpp
auto ep = static_cast<float>(calcEpsilon());
```

出于同样的原因，如果你故意想用整数类型存储一个表达式返回的浮点数类型的结果，你也可以使用这个方法。假如你需要计算一个随机访问迭代器（比如`std::vector`，`std::deque`或者`std::array`）中某元素的下标，你被提供一个`0.0`到`1.0`的`double`值表明这个元素离容器的头部有多远（`0.5`意味着位于容器中间）。进一步假设你很自信结果下标是`int`。如果容器是`c`，`d`是`double`类型变量，你可以用这样的方法计算容器下标：

```cpp
int index = d * c.size();
```

但是这种写法并没有明确表明你想将右侧的`double`类型转换成`int`类型，显式类型初始器可以帮助你正确表意：

```cpp
auto index = static_cast<int>(d * size());
```

**请记住：**

- 不可见的代理类可能会使`auto`从表达式中推导出“错误的”类型
- 显式类型初始器惯用法强制`auto`推导出你想要的结果

## 3.移步现代C++

### 3.1区别使用（）和{}创建对象

初始化方式

```cpp
int x(0);               //使用圆括号初始化

int y = 0;              //使用"="初始化

int z{ 0 };             //使用花括号初始化
```

在很多情况下可以使用

```cpp
int z = { 0 };          //使用"="和花括号
```

但是我们通常忽略掉这种形式，因为在C++中这种就和直接使用大括号初始化是一样的。

**自定义类没有定义这种构造函数也可以这么初始化，是一样的**

“乱的一塌糊涂”是指在初始化中使用"="可能会误导C++新手，使他们以为这里发生了赋值运算，然而实际并没有。对于像`int`这样的内置类型，研究两者区别就像在做学术，但是对于用户定义的类型而言，区别赋值运算符和初始化就非常重要了，因为它们涉及不同的函数调用：

```cpp
Widget w1;              //调用默认构造函数

Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数

w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
```

甚至对于一些初始化语法，在一些情况下C++98没有办法表达预期的初始化行为。举个例子，要想直接创建并初始化一个存放一些特殊值的STL容器是不可能的（比如1,3,5）。

C++11使用统一初始化（*uniform initialization*）来整合这些混乱且不适于所有情景的初始化语法，所谓==统一初始化==是指在任何涉及初始化的地方都使用单一的初始化语法。 它基于花括号，出于这个原因我更喜欢称之为括号初始化。（**译注：注意，这里的括号初始化指的是花括号初始化，在没有歧义的情况下下文的括号初始化指的都是用花括号进行初始化；当与圆括号初始化同时存在并可能产生歧义时我会直接指出。**）统一初始化是一个概念上的东西，而括号初始化是一个具体语法结构。

括号初始化让你可以表达以前表达不出的东西。使用花括号，创建并指定一个容器的初始元素变得很容易：

```cpp
std::vector<int> v{ 1, 3, 5 };  //v初始内容为1,3,5
```

括号初始化也能被用于为**类内非静态数据成员指定默认初始值**。C++11允许"="初始化不加花括号也拥有这种能力：

```cpp
class Widget{
    …

private:
    int x{ 0 };                 //没问题，x初始值为0
    int y = 0;                  //也可以
    int z(0);                   //错误！
}
```

另一方面，==不可拷贝的对象==（例如`std::atomic`——见[Item40](https://cntransgroup.github.io/EffectiveModernCppChinese/7.TheConcurrencyAPI/item40.html)）可以使用花括号初始化或者圆括号初始化，但是不能使用"="初始化：

```cpp
std::atomic<int> ai1{ 0 };      //没问题
std::atomic<int> ai2(0);        //没问题
std::atomic<int> ai3 = 0;       //错误！
```

因此我们很容易理解为什么括号初始化又叫统一初始化，在C++中这三种方式都被看做是初始化表达式，但是只有花括号任何地方都能被使用。

括号表达式还有一个少见的特性，即它不允许内置类型间隐式的变窄转换（*narrowing conversion*）。如果一个使用了括号初始化的表达式的值，不能保证由被初始化的对象的类型来表示，代码就不会通过编译：

```cpp
double x, y, z;

int sum1{ x + y + z };          //错误！double的和可能不能表示为int
```

使用圆括号和"="的初始化不检查是否转换为变窄转换，因为由于历史遗留问题它们必须要兼容老旧代码：

```cpp
int sum2(x + y +z);             //可以（表达式的值被截为int）

int sum3 = x + y + z;           //同上
```

另一个值得注意的特性是括号表达式对于C++最令人头疼的解析问题有天生的免疫性。（译注：所谓最令人头疼的解析即*most vexing parse*，更多信息请参见https://en.wikipedia.org/wiki/Most_vexing_parse。）==C++规定任何*可以被解析*为一个声明的东西*必须被解析*为声明==。这个规则的副作用是让很多程序员备受折磨：他们可能想创建一个使用默认构造函数构造的对象，却不小心变成了函数声明。问题的根源是如果你调用带参构造函数，你可以这样做：

```cpp
Widget w1(10);                  //使用实参10调用Widget的一个构造函数
```

但是如果你尝试使用相似的语法调用`Widget`无参构造函数，它就会变成函数声明：

```cpp
Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
```

由于函数声明中形参列表不能带花括号，所以使用花括号初始化表明你想调用默认构造函数构造对象就没有问题：

```cpp
Widget w3{};                    //调用没有参数的构造函数构造对象
```

关于括号初始化还有很多要说的。它的语法能用于各种不同的上下文，它防止了隐式的变窄转换，而且对于C++最令人头疼的解析也天生免疫。既然好到这个程度那为什么这个条款不叫“优先考虑括号初始化语法”呢？

**有initializer_list形参的构造函数，初始化会优先选择initializer_list版本**

括号初始化的缺点是有时它有一些令人惊讶的行为。这些行为使得括号初始化、`std::initializer_list`和构造函数参与重载决议时本来就不清不楚的暧昧关系进一步混乱。把它们放到一起会让看起来应该左转的代码右转。举个例子，[Item2](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item2.html)解释了当`auto`声明的变量使用花括号初始化，变量类型会被推导为`std::initializer_list`，但是使用相同内容的其他初始化方式会产生更符合直觉的结果。所以，你越喜欢用`auto`，你就越不能用括号初始化。

在构造函数调用中，只要不包含`std::initializer_list`形参，那么花括号初始化和圆括号初始化都会产生一样的结果：

```cpp
class Widget { 
public:  
    Widget(int i, bool b);      //构造函数未声明
    Widget(int i, double d);    //std::initializer_list这个形参 
    …
};
Widget w1(10, true);            //调用第一个构造函数
Widget w2{10, true};            //也调用第一个构造函数
Widget w3(10, 5.0);             //调用第二个构造函数
Widget w4{10, 5.0};             //也调用第二个构造函数
```

然而，如果有一个或者多个构造函数的声明包含一个`std::initializer_list`形参，那么使用括号初始化语法的调用更倾向于选择带`std::initializer_list`的那个构造函数。如果编译器遇到一个括号初始化并且有一个带std::initializer_list的构造函数，那么它一定会选择该构造函数。如果上面的`Widget`类有一个`std::initializer_list<long double>`作为参数的构造函数，就像这样：

```cpp
class Widget { 
public:  
    Widget(int i, bool b);      //同上
    Widget(int i, double d);    //同上
    Widget(std::initializer_list<long double> il);      //新添加的
    …
}; 
```

`w2`和`w4`将会使用新添加的构造函数，即使另一个非`std::initializer_list`构造函数和实参更匹配：

```cpp
Widget w1(10, true);    //使用圆括号初始化，同之前一样
                        //调用第一个构造函数

Widget w2{10, true};    //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 true 转化为long double)

Widget w3(10, 5.0);     //使用圆括号初始化，同之前一样
                        //调用第二个构造函数 

Widget w4{10, 5.0};     //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 5.0 转化为long double)
```

==甚至普通构造函数和移动构造函数都会被带`std::initializer_list`的构造函数劫持：==

```cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    Widget(std::initializer_list<long double> il);      //同之前一样
    operator float() const;                             //转换为float
    …
};

Widget w5(w4);                  //使用圆括号，调用拷贝构造函数

Widget w6{w4};                  //使用花括号，调用std::initializer_list构造
                                //函数（w4转换为float，float转换为double）

Widget w7(std::move(w4));       //使用圆括号，调用移动构造函数

Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
                                //函数（与w6相同原因）
```

编译器一遇到括号初始化就选择带`std::initializer_list`的构造函数的决心是如此强烈，以至于就算带`std::initializer_list`的构造函数不能被调用，它也会硬选。

```cpp
class Widget { 
public: 
    Widget(int i, bool b);                      //同之前一样
    Widget(int i, double d);                    //同之前一样
    Widget(std::initializer_list<bool> il);     //现在元素类型为bool
    …                                           //没有隐式转换函数
};

Widget w{10, 5.0};              //错误！要求变窄转换
```

这里，编译器会直接忽略前面两个构造函数（其中第二个构造函数是所有实参类型的最佳匹配），然后尝试调用`std::initializer_list<bool>`构造函数。调用这个函数将会把`int(10)`和`double(5.0)`转换为`bool`，由于会产生变窄转换（`bool`不能准确表示其中任何一个值），括号初始化拒绝变窄转换，所以这个调用无效，代码无法通过编译。

只有当没办法把括号初始化中实参的类型转化为`std::initializer_list`时，编译器才会回到正常的函数决议流程中。比如我们在构造函数中用`std::initializer_list<std::string>`代替`std::initializer_list<bool>`，这时非`std::initializer_list`构造函数将再次成为函数决议的候选者，因为没有办法把`int`和`bool`转换为`std::string`:

```cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    //现在std::initializer_list元素类型为std::string
    Widget(std::initializer_list<std::string> il);
    …                                                   //没有隐式转换函数
};

Widget w1(10, true);     // 使用圆括号初始化，调用第一个构造函数
Widget w2{10, true};     // 使用花括号初始化，现在调用第一个构造函数
Widget w3(10, 5.0);      // 使用圆括号初始化，调用第二个构造函数
Widget w4{10, 5.0};      // 使用花括号初始化，现在调用第二个构造函数
```

代码的行为和我们刚刚的论述如出一辙。这里还有一个有趣的[边缘情况](https://en.wikipedia.org/wiki/Edge_case)。假如你使用的花括号初始化是空集，并且你欲构建的对象有默认构造函数，也有`std::initializer_list`构造函数。你的空的花括号意味着什么？如果它们意味着没有实参，就该使用默认构造函数，但如果它意味着一个空的`std::initializer_list`，就该调用`std::initializer_list`构造函数。

最终会调用默认构造函数。空的花括号意味着没有实参，不是一个空的`std::initializer_list`：

```cpp
class Widget { 
public:  
    Widget();                                   //默认构造函数
    Widget(std::initializer_list<int> il);      //std::initializer_list构造函数

    …                                           //没有隐式转换函数
};

Widget w1;                      //调用默认构造函数
Widget w2{};                    //也调用默认构造函数
Widget w3();                    //最令人头疼的解析！声明一个函数
```

如果你想用空`std::initializer`来调用`std::initializer_list`构造函数，你就得创建一个空花括号作为函数实参——把空花括号放在圆括号或者另一个花括号内来界定你想传递的东西。

```cpp
Widget w4({});                  //使用空花括号列表调用std::initializer_list构造函数
Widget w5{{}};                  //同上
```

此时，括号初始化，`std::initializer_list`和构造函数重载的晦涩规则就会一下子涌进你的脑袋，你可能会想研究半天这些东西在你的日常编程中到底占多大比例。可能比你想象的要有用。因为`std::vector`作为受众之一会直接受到影响。`std::vector`有一个非`std::initializer_list`构造函数允许你去指定容器的初始大小，以及使用一个值填满你的容器。但它也有一个`std::initializer_list`构造函数允许你使用花括号里面的值初始化容器。如果你创建一个数值类型的`std::vector`（比如`std::vector<int>`），然后你传递两个实参，把这两个实参放到圆括号和放到花括号中有天壤之别：

```cpp
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20
```

让我们回到之前的话题。从以上讨论中我们得出两个重要结论。第一，作为一个类库作者，你需要意识到如果一堆重载的构造函数中有一个或者多个含有`std::initializer_list`形参，用户代码如果使用了括号初始化，可能只会看到你`std::initializer_list`版本的重载的构造函数。因此，你最好把你的构造函数设计为不管用户是使用圆括号还是使用花括号进行初始化都不会有什么影响。换句话说，了解了`std::vector`设计缺点后，你以后设计类的时候应该避免诸如此类的问题。

这里的暗语是如果一个类没有`std::initializer_list`构造函数，然后你添加一个，用户代码中如果使用括号初始化，可能会发现过去被决议为非`std::initializer_list`构造函数而现在被决议为新的函数。当然，这种事情也可能发生在你添加一个函数到那堆重载函数的时候：过去被决议为旧的重载函数而现在调用了新的函数。`std::initializer_list`重载不会和其他重载函数比较，它直接盖过了其它重载函数，其它重载函数几乎不会被考虑。所以如果你要加入`std::initializer_list`构造函数，请三思而后行。
	**总结就是引入花括号参数很有可能影响其他函数的调用，所以添加大括号初始化构造函数得三思**

==第二==，作为一个类库使用者，你必须认真的在花括号和圆括号之间选择一个来创建对象。大多数开发者都使用其中一种作为默认情况，只有当他们不能使用这种的时候才会考虑另一种。默认使用花括号初始化的开发者主要被适用面广、禁止变窄转换、免疫C++最令人头疼的解析这些优点所吸引。这些开发者知道在一些情况下（比如给定一个容器大小和一个初始值创建`std::vector`）要使用圆括号。默认使用圆括号初始化的开发者主要被C++98语法一致性、避免`std::initializer_list`自动类型推导、避免不会不经意间调用`std::initializer_list`构造函数这些优点所吸引。这些开发者也承认有时候只能使用花括号（比如创建一个包含着特定值的容器）。关于花括号初始化和圆括号初始化哪种更好大家没有达成一致，所以我的建议是选择一种并坚持使用它。

如果你是一个模板的作者，花括号和圆括号创建对象就更麻烦了。通常不能知晓哪个会被使用。举个例子，假如你想创建一个接受任意数量的参数来创建的对象。使用可变参数模板（*variadic template*）可以非常简单的解决：

```cpp
template<typename T,            //要创建的对象类型
         typename... Ts>        //要使用的实参的类型
void doSomeWork(Ts&&... params)
{
    create local T object from params...
    …
} 
```

在现实中我们有两种方式实现这个伪代码（关于`std::forward`请参见[Item25](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item25.html)）：

```cpp
T localObject(std::forward<Ts>(params)...);             //使用圆括号
T localObject{std::forward<Ts>(params)...};             //使用花括号
```

考虑这样的调用代码：

```cpp
std::vector<int> v; 
…
doSomeWork<std::vector<int>>(10, 20);
```

如果`doSomeWork`创建`localObject`时使用的是圆括号，`std::vector`就会包含10个元素。如果`doSomeWork`创建`localObject`时使用的是花括号，`std::vector`就会包含2个元素。哪个是正确的？`doSomeWork`的作者不知道，只有调用者知道。

这正是标准库函数`std::make_unique`和`std::make_shared`（参见[Item21](https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item21.html)）面对的问题。它们的解决方案是使用圆括号，并被记录在文档中作为接口的一部分。（注：更灵活的设计——允许调用者决定从模板来的函数应该使用圆括号还是花括号——是有可能的。详情参见[Andrzej’s C++ blog](http://akrzemi1.wordpress.com/)在2013年6月5日的文章，“[Intuitive interface — Part I.](http://akrzemi1.wordpress.com/2013/06/05/intuitive-interface-part-i/)”）

**请记住：**

- 花括号初始化是最广泛使用的初始化语法，它防止变窄转换，并且对于C++最令人头疼的解析有天生的免疫性
- 在构造函数重载决议中，编译器会尽最大努力将括号初始化与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
- 对于数值类型的`std::vector`来说使用花括号初始化和圆括号初始化会造成巨大的不同
- 在模板类选择使用圆括号初始化或使用花括号初始化创建对象是一个挑战。

### 3.2优先考虑nullptr而非0和NULL

==这个就记住函数重载会出问题就好了==

**当函数参数有多个重载时，使用0或NULL可能不能调用你想使用的那个函数：**

​	字面值`0`是一个`int`不是指针。如果C++发现在当前上下文只能使用指针，它会很不情愿的把`0`解释为指针，但是那是最后的退路。一般来说C++的解析策略是把`0`看做`int`而不是指针。

实际上，`NULL`也是这样的。但在`NULL`的实现细节有些不确定因素，因为实现被允许给`NULL`一个除了`int`之外的整型类型（比如`long`）。这不常见，但也算不上问题所在。这里的问题不是`NULL`没有一个确定的类型，而是`0`和`NULL`都不是指针类型。

在C++98中，对指针类型和整型进行重载意味着可能导致奇怪的事情。如果给下面的重载函数传递`0`或`NULL`，它们绝不会调用指针版本的重载函数：

```cpp
void f(int);        //三个f的重载函数
void f(bool);
void f(void*);

f(0);               //调用f(int)而不是f(void*)

f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)
```

而`f(NULL)`的不确定行为是由`NULL`的实现不同造成的。如果`NULL`被定义为`0L`（指的是`0`为`long`类型），这个调用就具有二义性，因为从`long`到`int`的转换或从`long`到`bool`的转换或`0L`到`void*`的转换都同样好。有趣的是源代码**表现出**的意思（“我使用空指针`NULL`调用`f`”）和**实际表达出**的意思（“我是用整型数据而不是空指针调用`f`”）是相矛盾的。这种违反直觉的行为导致C++98程序员都将避开同时重载指针和整型作为编程准则（译注：请务必注意结合上下文使用这条规则）。在C++11中这个编程准则也有效，因为尽管我这个条款建议使用`nullptr`，可能很多程序员还是会继续使用`0`或`NULL`，哪怕`nullptr`是更好的选择。`nullptr`的优点是它不是整型。老实说它也不是一个指针类型，但是你可以把它认为是**所有**类型的指针。`nullptr`的真正类型是`std::nullptr_t`，在一个完美的循环定义以后，`std::nullptr_t`又被定义为`nullptr`。`std::nullptr_t`可以隐式转换为指向任何内置类型的指针，这也是为什么`nullptr`表现得像所有类型的指针。

使用`nullptr`调用`f`将会调用`void*`版本的重载函数，因为`nullptr`不能被视作任何整型：

```cpp
f(nullptr);         //调用重载函数f的f(void*)版本
```

使用`nullptr`代替`0`和`NULL`可以避开了那些令人奇怪的函数重载决议，这不是它的唯一优势。它也可以使代码表意明确，尤其是当涉及到与`auto`声明的变量一起使用时。举个例子，假如你在一个代码库中遇到了这样的代码：

```cpp
auto result = findRecord( /* arguments */ );
if (result == 0) {
    …
} 
```

如果你不知道`findRecord`返回了什么（或者不能轻易的找出），那么你就不太清楚到底`result`是一个指针类型还是一个整型。毕竟，`0`（用来测试`result`的值的那个）也可以像我们之前讨论的那样被解析。但是换一种假设如果你看到这样的代码：

```cpp
auto result = findRecord( /* arguments */ );

if (result == nullptr) {  
    …
}
```

这就没有任何歧义：`result`的结果一定是指针类型。

当模板出现时`nullptr`就更有用了。假如你有一些函数只能被合适的已锁互斥量调用。每个函数都有一个不同类型的指针：

```cpp
int    f1(std::shared_ptr<Widget> spw);     //只能被合适的
double f2(std::unique_ptr<Widget> upw);     //已锁互斥量
bool   f3(Widget* pw);                      //调用
```

如果这样传递空指针：

```cpp
std::mutex f1m, f2m, f3m;       //用于f1，f2，f3函数的互斥量

using MuxGuard =                //C++11的typedef，参见Item9
    std::lock_guard<std::mutex>;
…

{  
    MuxGuard g(f1m);            //为f1m上锁
    auto result = f1(0);        //向f1传递0作为空指针
}                               //解锁 
…
{  
    MuxGuard g(f2m);            //为f2m上锁
    auto result = f2(NULL);     //向f2传递NULL作为空指针
}                               //解锁 
…
{
    MuxGuard g(f3m);            //为f3m上锁
    auto result = f3(nullptr);  //向f3传递nullptr作为空指针
}                               //解锁 
```

令人遗憾前两个调用没有使用`nullptr`，但是代码可以正常运行，这也许对一些东西有用。但是重复的调用代码——为互斥量上锁，调用函数，解锁互斥量——更令人遗憾。它让人很烦。模板就是被设计于减少重复代码，所以让我们模板化这个调用流程：

```cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,                 
                 MuxType& mutex,                 
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);  
    return func(ptr); 
}
```

如果你对函数返回类型（`auto ... -> decltype(func(ptr))`）感到困惑不解，[Item3](https://cntransgroup.github.io/EffectiveModernCppChinese/1.DeducingTypes/item3.html)可以帮助你。在C++14中代码的返回类型还可以被简化为`decltype(auto)`：

```cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,       //C++14
                           MuxType& mutex,
                           PtrType ptr)
{ 
    MuxGuard g(mutex);  
    return func(ptr); 
}
```

可以写这样的代码调用`lockAndCall`模板（两个版本都可）：

```cpp
auto result1 = lockAndCall(f1, f1m, 0);         //错误！
...
auto result2 = lockAndCall(f2, f2m, NULL);      //错误！
...
auto result3 = lockAndCall(f3, f3m, nullptr);   //没问题
```

代码虽然可以这样写，但是就像注释中说的，前两个情况不能通过编译。在第一个调用中存在的问题是当`0`被传递给`lockAndCall`模板，模板类型推导会尝试去推导实参类型，`0`的类型总是`int`，所以这就是这次调用`lockAndCall`实例化出的`ptr`的类型。不幸的是，这意味着`lockAndCall`中`func`会被`int`类型的实参调用，这与`f1`期待的`std::shared_ptr<Widget>`形参不符。传递`0`给`lockAndCall`本来想表示空指针，但是实际上得到的一个普通的`int`。把`int`类型看做`std::shared_ptr<Widget>`类型给`f1`自然是一个类型错误。在模板`lockAndCall`中使用`0`之所以失败是因为在模板中，传给的是`int`但实际上函数期待的是一个`std::shared_ptr<Widget>`。

第二个使用`NULL`调用的分析也是一样的。当`NULL`被传递给`lockAndCall`，形参`ptr`被推导为整型（译注：由于依赖于具体实现所以不一定是整数类型，所以用整型泛指`int`，`long`等类型），然后当`ptr`——一个`int`或者类似`int`的类型——传递给`f2`的时候就会出现类型错误，`f2`期待的是`std::unique_ptr<Widget>`。

然而，使用`nullptr`是调用没什么问题。当`nullptr`传给`lockAndCall`时，`ptr`被推导为`std::nullptr_t`。当`ptr`被传递给`f3`的时候，隐式转换使`std::nullptr_t`转换为`Widget*`，因为`std::nullptr_t`可以隐式转换为任何指针类型。

模板类型推导将`0`和`NULL`推导为一个错误的类型（即它们的实际类型，而不是作为空指针的隐含意义），这就导致在当你想要一个空指针时，它们的替代品`nullptr`很吸引人。使用`nullptr`，模板不会有什么特殊的转换。另外，使用`nullptr`不会让你受到同重载决议特殊对待`0`和`NULL`一样的待遇。当你想用一个空指针，使用`nullptr`，不用`0`或者`NULL`。

**记住**

- 优先考虑`nullptr`而非`0`和`NULL`
- 避免重载指针和整型

### 3.3优先考虑别名声明而非typedef

我相信每个人都同意使用STL容器是个好主意，并且我希望Item18能说服你让你觉得使用`std:unique_ptr`也是个好主意，但我猜没有人喜欢写上几次 `std::unique_ptr<std::unordered_map<std::string, std::string>>`这样的类型，它可能会让你患上腕管综合征的风险大大增加。

避免上述医疗悲剧也很简单，引入`typedef`即可：

```cpp
typedef
    std::unique_ptr<std::unordered_map<std::string, std::string>>
    UPtrMapSS; 
```

但`typedef`是C++98的东西。虽然它可以在C++11中工作，但是C++11也提供了一个别名声明（*alias declaration*）：

```cpp
using UPtrMapSS =
    std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

由于这里给出的`typedef`和别名声明做的都是完全一样的事情，我们有理由想知道会不会出于一些技术上的原因两者有一个更好。

这里，在说它们之前我想提醒一下很多人都发现当声明一个函数指针时别名声明更容易理解：

```cpp
//FP是一个指向函数的指针的同义词，它指向的函数带有
//int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&);    //typedef

//含义同上
using FP = void (*)(int, const std::string&);   //别名声明
```

当然，两个结构都不是非常让人满意，没有人喜欢花大量的时间处理函数指针类型的别名（译注：指`FP`），所以至少在这里，没有一个吸引人的理由让你觉得别名声明比`typedef`好。

==**当你用using起别名的时候可以省去很多地方的麻烦，比如模板中的麻烦**==

不过有一个地方使用别名声明吸引人的理由是存在的：模板。特别地，别名声明可以被模板化（这种情况下称为别名模板*alias template*s）但是`typedef`不能。这使得C++11程序员可以很直接的表达一些C++98中只能把`typedef`嵌套进模板化的`struct`才能表达的东西。考虑一个链表的别名，链表使用自定义的内存分配器，`MyAlloc`。使用别名模板，这真是太容易了：

```cpp
template<typename T>                            //MyAllocList<T>是
using MyAllocList = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                //的同义词

MyAllocList<Widget> lw;                         //用户代码
```

使用`typedef`，你就只能从头开始：

```cpp
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};

MyAllocList<Widget>::type lw;                   //用户代码
```

更糟糕的是，如果你想使用在一个模板内使用`typedef`声明一个链表对象，而这个对象又使用了模板形参，你就不得不在`typedef`前面加上`typename`：

```cpp
template<typename T>
class Widget {                              //Widget<T>含有一个
private:                                    //MyAllocLIst<T>对象
    typename MyAllocList<T>::type list;     //作为数据成员
    …
}; 
```

这里`MyAllocList<T>::type`使用了一个类型，这个类型依赖于模板参数`T`。因此`MyAllocList<T>::type`是一个依赖类型（*dependent type*），在C++很多讨人喜欢的规则中的一个提到必须要在依赖类型名前加上`typename`。

如果使用别名声明定义一个`MyAllocList`，就不需要使用`typename`（同时省略麻烦的“`::type`”后缀）：

```cpp
template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>;   //同之前一样

template<typename T> 
class Widget {
private:
    MyAllocList<T> list;                        //没有“typename”
    …                                           //没有“::type”
};
```

对你来说，`MyAllocList<T>`（使用了模板别名声明的版本）可能看起来和`MyAllocList<T>::type`（使用`typedef`的版本）一样都应该依赖模板参数`T`，但是你不是编译器。==当编译器处理`Widget`模板时遇到`MyAllocList<T>`（使用模板别名声明的版本），它们知道`MyAllocList<T>`是一个类型名，因为`MyAllocList`是一个别名模板：它**一定**是一个类型名。因此`MyAllocList<T>`就是一个**非依赖类型**（*non-dependent type*），就不需要也不允许使用`typename`修饰符==。

当编译器在`Widget`的模板中看到`MyAllocList<T>::type`（使用`typedef`的版本），它不能确定那是一个类型的名称。因为可能存在一个`MyAllocList`的它们没见到的特化版本，那个版本的`MyAllocList<T>::type`指代了一种不是类型的东西（**比如？搜一下举个例子**）。那听起来很不可思议，但不要责备编译器穷尽考虑所有可能。因为人确实能写出这样的代码。

举个例子，一个误入歧途的人可能写出这样的代码：

```cpp
class Wine { … };

template<>                          //当T是Wine
class MyAllocList<Wine> {           //特化MyAllocList
private:  
    enum class WineType             //参见Item10了解  
    { White, Red, Rose };           //"enum class"

    WineType type;                  //在这个类中，type是
    …                               //一个数据成员！
};
```

==就像你看到的，`MyAllocList<Wine>::type`不是一个类型。如果`Widget`使用`Wine`实例化，在`Widget`模板中的`MyAllocList<Wine>::type`将会是一个数据成员，不是一个类型。在`Widget`模板内，`MyAllocList<T>::type`是否表示一个类型取决于`T`是什么，这就是为什么编译器会坚持要求你在前面加上`typename`==。

如果你尝试过模板元编程（*template metaprogramming*，TMP）， 你一定会碰到取模板类型参数然后基于它创建另一种类型的情况。举个例子，给一个类型`T`，如果你想去掉`T`的常量修饰和引用修饰（`const`- or reference qualifiers），比如你想把`const std::string&`变成`std::string`。又或者你想给一个类型加上`const`或变为左值引用，比如把`Widget`变成`const Widget`或`Widget&`。（如果你没有用过模板元编程，太遗憾了，因为如果你真的想成为一个高效C++程序员，你需要至少熟悉C++在这方面的基本知识。你可以看看在[Item23](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item23.html)，[27](https://cntransgroup.github.io/EffectiveModernCppChinese/5.RRefMovSemPerfForw/item27.html)里的TMP的应用实例，包括我提到的类型转换）。

C++11在*type traits*（类型特性）中给了你一系列工具去实现类型转换，如果要使用这些模板请包含头文件`<type_traits>`。里面有许许多多*type traits*，也不全是类型转换的工具，也包含一些可预测接口的工具。给一个你想施加转换的类型`T`，结果类型就是`std::`transformation`<T>::type`，比如：

```cpp
std::remove_const<T>::type          //从const T中产出T
std::remove_reference<T>::type      //从T&和T&&中产出T
std::add_lvalue_reference<T>::type  //从T中产出T&
```

注释仅仅简单的总结了类型转换做了什么，所以不要太随便的使用。在你的项目使用它们之前，你最好看看它们的详细说明书。

尽管写了一些，但我这里不是想给你一个关于*type traits*使用的教程。注意类型转换尾部的`::type`。如果你在一个模板内部将他们施加到类型形参上（实际代码中你也总是这么用），你也需要在它们前面加上`typename`。至于为什么要这么做是因为这些C++11的*type traits*是通过在`struct`内嵌套`typedef`来实现的。是的，它们使用类型同义词（译注：根据上下文指的是使用`typedef`的做法）技术实现，而正如我之前所说这比别名声明要差。

关于为什么这么实现是有历史原因的，但是我们跳过它（我认为太无聊了），因为标准委员会没有及时认识到别名声明是更好的选择，所以直到C++14它们才提供了使用别名声明的版本。这些别名声明有一个通用形式：对于C++11的类型转换`std::`transformation`<T>::type`在C++14中变成了`std::`transformation`_t`。举个例子或许更容易理解：

```cpp
std::remove_const<T>::type          //C++11: const T → T 
std::remove_const_t<T>              //C++14 等价形式

std::remove_reference<T>::type      //C++11: T&/T&& → T 
std::remove_reference_t<T>          //C++14 等价形式

std::add_lvalue_reference<T>::type  //C++11: T → T& 
std::add_lvalue_reference_t<T>      //C++14 等价形式
```

C++11的的形式在C++14中也有效，但是我不能理解为什么你要去用它们。就算你没办法使用C++14，使用别名模板也是小儿科。只需要C++11的语言特性，甚至每个小孩都能仿写，对吧？如果你有一份C++14标准，就更简单了，只需要复制粘贴：

```cpp
template <class T> 
using remove_const_t = typename remove_const<T>::type;

template <class T> 
using remove_reference_t = typename remove_reference<T>::type;

template <class T> 
using add_lvalue_reference_t =
  typename add_lvalue_reference<T>::type; 
```

看见了吧？不能再简单了。

**请记住：**

- `typedef`不支持模板化，但是别名声明支持。
- 别名模板避免了使用“`::type`”后缀，而且在模板中使用`typedef`还需要在前面加上`typename`
- C++14提供了C++11所有*type traits*转换的别名声明版本
