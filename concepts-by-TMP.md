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


## 实例-EqualityComparable

EqualityComparable定义了两个类型可比较相等的限制，
即它们之间有`==`和`!=`的操作符重载

### EqualityComparable in concepts

```cpp
template <typename T>
concept bool EqualityComparable = 
        requires (T t) {            // 参数化限制
            {t == t} -> bool,       // 表达式限制， 隐式转化限制
            {t != t} -> bool
        }

template <typename T, typename U = T>
concept bool EqualityComparable =
        EqualityComparable<T> &&    // 合取限制
        EqualityComparable<U> &&
        requires (T t, U u) {
            {t == u} -> bool,
            {u == t} -> bool,
            {t != u} -> bool
            {u != t} -> bool
        }
```

### EqualityComparable in template metaprogramming

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

template <typename T>
using Arithmatic = typename std::enable_if<_Arithmatic<T>::value>::type;

template <typename T, typename Arithmatic<T>>
void f(T p)
{
    std::cout << typeid(T).name() <<  " is Arithmatic" << std::endl;
}
```

To understand these code, you should have a basic idea of SFINAE (a method that makes template powerful), and template matching rules
in the template parameters, decltype is used to check existence of operator== and operator!=, and for different types, it has to check their self comparability before mutual comparability

a concept version:
```cpp

```

