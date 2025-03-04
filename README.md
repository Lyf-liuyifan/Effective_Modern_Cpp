# Effective Modern C++学习笔记

## 1.类型推导

​	越学习越觉得c++类型名繁杂冗长，而且很多类型名解读很麻烦，所以c++推出两套规则，一套用于auto，一套用于decltype。但是类型推导并不总是如我们所期待的那样进行，如果对于类型推导操作没有一个扎实的理解，想要写出一个具有现代感的c++程序是很难得。

​	==这章解释了模板类型推导是如何工作的，auto是如何依赖类型推导的，以及decltype是如何按照它自己独特的规则工作。它甚至解释了你该如何强制编译器使类型推导的结果可视，这能让你确认编译器的类型推导是否按照你期望的那样进行。==

### 1.1理解模板类型推导

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
