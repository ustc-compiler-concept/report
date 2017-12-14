# 使用模板元编程实现concepts

根据[concepts简介](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-intro.md)，我们了解到了总共有九种对类型的限制，它们分别是：

1. 合取限制
2. 析取限制
3. 断言限制
4. 表达式限制
5. 类型限制
6. 隐式转化限制
7. 参数推导限制
8. 异常限制
9. 参数化限制

我们将通过一个例子来说明模板元编程可以实现concepts的部分功能，
再给出concepts到模板元编程的翻译方案（并不语法制导）来说明concepts能用元编程实现。


## 实例-Arithmatic

Arithmatic定义了四则运算限制，即所限制的类型必须有`+`、`-`、`*`、`/`四个操作符

### 使用concepts定义Arithmatic

```cpp
template <typename T>
concept bool Arithmatic = 
        requires (T t) {            // 参数化限制
            {t + t} -> T,       // 表达式限制， 隐式转化限制
            {t - t} -> T,
            {t * t} -> T,
            {t / t} -> T
        }
```

它可以这样来使用：

```cpp
Arithmatic{T}
void f(T t)
{
    std::cout << typeid(T).name() << " is Arithmatic." << std::endl;
}
```

### 使用模板元变成定义Arithmatic

```cpp
// void_t只接收构造参数，并不产生任何类型
template <typename ...>
using void_t = void;

// 类似std::common_type
template <typename T, typename U>
struct common_type {  };

// 匹配失败
template <typename T, typename = void>
struct _Arithmatic : std::false_type { };


// 三元运算符对第一个操作数有
template <typename T>
struct _Arithmatic<T, void_t<
    typename std::enable_if<
        std::is_same<
            decltype(static_cast<T&&>(*(T*)0) + static_cast<T&&>(*(T*)0)),
            T
            >::value
        >::type,
    typename std::enable_if<
        std::is_same<
            decltype(static_cast<T&&>(*(T*)0) - static_cast<T&&>(*(T*)0)),
            T
            >::value
        >::type,
    typename std::enable_if<
        std::is_same<
            decltype(static_cast<T&&>(*(T*)0) * static_cast<T&&>(*(T*)0)),
            T
            >::value
        >::type,
    typename std::enable_if<
        std::is_same<
            decltype(static_cast<T&&>(*(T*)0) / static_cast<T&&>(*(T*)0)),
            T>::value
        >::type
>> : std::true_type { };

// 封装以方便使用
template <typename T>
using Arithmatic = typename std::enable_if<_Arithmatic<T>::value>::type;
```

我们通过关联类型来使用它：

```cpp
template <typename T, typename Arithmatic<T>>
void f(T p)
{
    std::cout << typeid(T).name() <<  " is Arithmatic" << std::endl;
}
```

只要有着基本的模板知识，这些代码并不难理解。

## 将限制翻译到模板

在本节，将会一一介绍每种类型的限制能否翻译到模板和它们的翻译方案。

### 合取限制

封装类型`P`和`Q`的合取可以直接将它们加入到`void_t`的参数列表中，未封装的类`_P`和`_Q`的合取可以表示为将`std::enable_if<_P::value && _Q::value>::type`加入到`void_t`的参数列表中

### 析取限制

封装后的类型并不能表示析取，但是未封装的类`_P`和`_Q`的析取可以表示为将`std::enable_if<_P::value || _Q::value>::type`加入到`void_t`的参数列表中

### 断言限制

任一断言`P`，可以通过将`std::enable_if<P>::type`加入到`void_t`的参数列表中来增加断言限制

### 表达式限制

任一表达式`E`，可以通过将`decltype(E)`加入到`void_t`的参数中来增加表达式限制

### 类型限制

任一类型`T`，可以通过将`T`加入到`void_t`的参数列表中来增加类型限制

### 隐式转换限制

任一隐式转换限制`{ E } -> T`可以同过将`decltype(static_cast<T>(E))`加入到`void_t`的参数列表中来增加类型限制

### 参数推导限制

由于`auto`当前的推导功能有限，所以无法支持参数推导限制

### 异常限制

任一异常限制`noexcept(E)`可以通过将`std::enable_if<noexcept(E)>::type`加入参数列表中来增加异常限制

### 参数化限制

参数化限制只是一个用来增加可读性的语法，在模板元编程中我们直接对类型计算，所以并不需要这个

## 总结

我们可以通过上述方法来模拟出concepts的大部分功能，这说明concepts并没有很大程度的扩展了C++语言的表达能力

## 参考文献

1. [boost hana](https://github.com/boostorg/hana/tree/master/include/boost/hana/concept)
