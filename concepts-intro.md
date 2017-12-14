# concepts

## 什么是concepts?

concepts是C++中的一个语言扩展。
它能够定义和检查对于模板参数的限制，
在这些这些限制上可以进行函数重载和模板特化。
为此，它引入了两个新的关键字，分别是`concept`和`requires`

## concepts的类型

在[技术说明](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4549.pdf)中，concepts的被定义为以下的形式：

    template <template-parameter-list>
    concept concept-name = constraint-expression;

它可以分为[函数concept](#变量concept)和[变量concept](#变量concept), 二者都必须以`bool`作为（返回）类型。从它的形式可以看出，对于concepts的实际定义都在等号后。而限制是指一系列能够定义对模板参数要求的逻辑操作，它们总共有9种类型：

1. 合取限制
2. 析取限制
3. 断言限制
4. 表达式限制
5. 类型限制
6. 隐式转化限制
7. 参数推导限制
8. 异常限制
9. 参数化限制

其中第4到第9种限制只能在`requires`表达式中被定义。

### 合取限制

任意两个限制`P`和`Q`的合取是`P && Q`，当且仅当`P`、`Q`都被满足时它才被满足

### 析取限制

任意两个限制`P`和`Q`的析取是`P || Q`，`P`、`Q`中任意一者被满足它都被满足

### 断言限制

断言限制是一个类型为`bool`的常量表达式，当且仅当该表达式的值是`true`它被满足

### 表达式限制

表达式限制将会替换模板中的参数到表达式中，只有替换成功才会被满足
如：

```cpp
template <typename T>
concept bool C = requires (T t) { ++t; };
```

只有`++t`合法时`C`才被满足

### 类型限制

类型限制通过替换模板参数进行类型构造，只有替换后的类型规范时才满足
如：

```cpp
template <typename T>
concept bool C = requires () { typename T::type; };
```

### 隐式转换限制

隐式转换限制将一个表达式E隐式转化到一个类型T来定义限制，只有E能被隐式转化到T时才能满足限制

如：

```cpp
template <typename T>
concept bool C =
requires (T a, T b) {
    { a == b } -> bool;
}
```

C中包含两个限制，分别是表达式限制和隐式转换限制

### 参数推导限制

参数推导限制将一个含有占位符的类型T推导出表达式E的类型，只有T能成功推出E的类型时才被满足。

如：

```cpp
template <typename T>
concept bool C1 = true;

template <typename T>
concept bool C() {
    return requires(T t) {
        {*t} -> std::pair<C1&, auto>;
    };
}
```

### 异常限制

异常限制只有一个表达式E满足`noexcept(E)`时才被满足

### 参数化限制

参数化限制声明了一些列参数，叫做限制变量，它们在`requires`的参数列表中被声明，它的操作数被定义为一系列限制的合取。只有参数类型进行替换后仍然规范，并且操作数满足要求时，参数化限制才满足要求。

```cpp
template <typename T>
concept bool Eq =
requires (T a, T b) { 
    a == b;
    a != b;
};
```

## 如何定义concepts

使用`concept`关键字和`requires`语句可以定义concepts。
它们只能在命名空间作用域被定义（而不能在类作用域，函数作用域等等）

### 变量concept

有`concept`限定符的模板变量定义了变量concept
它有以下三点限制：

1. 声明的类型必须是`bool`
2. 声明中必须包含初始定义
3. 初始定义应是[表达式限制](#concepts的类型)

### 函数concept

有`concept`限定符的模板函数定义了函数concept
它有以下四点限制:

1. 所有函数限定符不能出现在它的声明中
2. 返回类型必须是`bool`
3. 声明的参数列表必须为空
4. 声明应该有一个这样子的函数体：`{ return E; }`，其中`E`是表达式限制

### requires语句

定义时，我们使用requires语句来定义[之前](#concepts的类型)提到的限制类型，
它会跟着参数列表，所以本身就能定义参数化限制。

* 简单requires语句能够定义一个表达式限制
* 类型requires语句能够定义一个类型限制
* 混合requires能够定义表达式限制、类型限制、异常限制、参数推导限制和隐藏式转换限制
* 嵌套requires语句可以定义局部参数的一些附加限制

## 如何使用concepts


### concepts限制模板参数

当使用concepts来作为模板参数的限制时也需要使用requires语句。
这里requires语句用来指定模板参数的限制，而不是定义这些限制。

```cpp
template <typename T>
requires Eq<T>
bool f(T t);
```

它定义了一个模板函数`f`，模板参数类型`T`必须满足`Eq`这个concept所定义的限制才能被实例化成功

上面的例子还有两种缩写形式:

```cpp
template <Eq T>
bool f(T t);

Eq{T} bool f(T t);

template<typename T>
bool f(T t) requires Eq<T>;

bool f(Eq);
```

由此可见各种使用方式都是等价的，但是这些限制在检查时有优先级关系，详细可以参考[技术说明](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4549.pdf)

### 作为占位符使用

当concepts作为占位符使用时能支持同`C++11`中引进的占位符`auto`一样的推导能力，
并且能在此基础上增加concepts本身定义的限制。如:

```cpp
Sortable x = f(y)
// x的类型从f的返回值中推断出来，只有f的返回值满足Sortable限制时才能通过编译

auto f(Container) -> Sortable;
// 声明一个函数f，它接受满足Container限制类型的参数，返回类型满足Sortable限制
```


目前GCC 6.0以上的版本可以通过`-fconcepts`选项来使用测试中的concepts特性。

## 总结

本文对concepts的语法和语义进行了初步的介绍。通过本文应该能理解什么是concepts，我们能用它做什么，如果想要了解更多关于cocepts的资料应该去阅读文末的参考文献。

## 参考文献

1. [a bit of background for concepts and C++17 - BJarn Stroustrup](https://isocpp.org/blog/2016/02/a-bit-of-background-for-concepts-and-cpp17-bjarne-stroustrup)
2. [Concepts TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4549.pdf)


