# concepts

## 什么是concepts?

concepts是C++中的一个语言扩展。
它能够定义和检查对于模板参数的限制，
在这些这些限制上可以进行函数重载和模板特化。
为此，它引入了两个新的关键字，分别是`concept`和`requires`

## concepts的形式

在技术说明中，concepts的被定义为以下的形式：

    template <template-parameter-list>
    concept concept-name = constraint-expression;

它可以分为*函数concepts*和*变量concepts*, 二者都必须以`bool`作为（返回）类型。从它的形式可以看出，对于concepts的实际定义都在等号后的`constraint-expression`（即限制表达式）中。而限制是指一系列能够定义对模板参数要求的逻辑操作，它们总共有9种形式：

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


## 如何使用concepts


