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

​	decltype通常用于声明函数模板，auto也可以用于推到函数返回值，但是问题就是auto作为函数返回值会采用模板推导类型， 而模板类型推导会自动忽略引用类型，所以这是弊端，而使用decltype就不会有这种问题。

```cpp
std::deque<int> d;
…
authAndAccess(d, 5) = 10;               //认证用户，返回d[5]，
                                        //然后把10赋值给它
                                        //无法通过编译！
```

为什么需要推导引用类型呢？在C++11中，`decltype`最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型。举个例子，假定我们写一个函数，一个形参为容器，一个形参为索引值，这个函数支持使用方括号的方式（也就是使用“`[]`”）访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。函数的返回类型应该和索引操作返回的类型相同。

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
