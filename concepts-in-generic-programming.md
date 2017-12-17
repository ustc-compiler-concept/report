# concepts作为一种编程范式

在[concepts简介](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-intro.md)中，
我们介绍了为什么要引入concepts，但是没有介绍concepts如何解决现在的问题。
而在[基于模板元编程的concepts实现](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-by-TMP.md)
中，我们又介绍了concepts是可以通过现有的模板技术实现类似功能的。
在本篇中我们将会介绍concepts如何提供一个良好的接口，以及为什么
把它变成语言机制会更好。


## concepts的使用

sort是非常常见的需要泛型编程的东西，只要元素之间存在全序，任何一个
正确的排序算法都能将表的所有元素按照全序顺序排列，我们看一下一个使用
模板编程的例子：

```cpp
template <typename T>
void sort(T& a)
{
    // ... 排序的实际代码，其中包含了对operator[]的调用
}
```

如果一个实现了`operator[]`的类被传入，那么程序可以正常通过编译，但是
例如`double`被传入，它并没有`operator[]`，那么编译器会得到：

```
q.cpp:6:5: error: subscripted value is not an array, pointer, or vector
    a[2] = 0;
    ^ ~
q.cpp:12:5: note: in instantiation of function template specialization
      'sort<double>' requested here
    sort(d);
    ^
```

可以看到这里有几点常见的问题:

*   编译器报错的信息并不准确，追踪起来并不方便。例如在使用STL的代码中出现了错误，编译器会在不断向内实例化时才发现问题，然后打印出整个栈。一般人很难从海量的错误信息中找出真正有用的东西
*   sort函数对模板参数类型的需求隐藏在实现的代码之中，而不是显式地去说明它
*   `template<typename T>`需要重复出现，非常繁琐且不讨人喜欢

而在有了concepts之后我们可以这样写：

```cpp
template <typename T>
concept bool Sortable =
    requires (T t) {
        t[0];
        // ... 其它限制
    }

void sort(Sortable& s) {
    // ... 排序的实际代码，其中包含了对operator[]的调用
}
```

和上面一样，我们传入一个`double`类型：

```
q.cpp: In function ‘int main()’:
q.cpp:17:11: error: cannot call function ‘void sort(auto:1&) [with auto:1 = double]’
     sort(d);
           ^
q.cpp:9:6: note:   constraints not satisfied
 void sort(Sortable& a)
      ^~~~
q.cpp:4:14: note: within ‘template<class T> concept const bool Sortable<T> [with T = double]’
 concept bool Sortable =
              ^~~~~~~~
q.cpp:4:14: note:     with ‘double t’
q.cpp:4:14: note: the required expression ‘t[0]’ would be ill-formed
```

可以看到这次编译提示的信息非常丰富，它告诉我们在调用`sort`函数时传入的参数是`double`类型
而`double`类型并不满足`Sortable`这个concept的约束。具体来说就是表达式`t[0]`是不合法的。
我们可以看出这种方式显式地指定了约束，编译器的报错也友好了很多。不仅如此，缩写的concepts用法
不仅减少了重复，而且增加了可读性。

## concepts作为模板使用的接口

正如前面说道的，使用concepts来约束函数可以快速地让用户理解函数对参数
的要求，又能很方便地进行调试。所以，使用concepts来给来模板作为接口是
非常合适的。特别是标准模板库的使用中，经常会遇到由于自己对类的设计失误
导致编译器报一堆错误，如果模板库中的算法全面使用了concepts，那么使用
会方便很多。

在实践中，用户会同时作为concepts的使用者和设计者，我们已经看到作为使用者
时，一个使用了concepts的算法非常好用。但是如果面对一个设计不完善的concept，
那么使用者也会感到头疼，可以考虑下面这个例子：

```cpp
template <typename T, typename U>
concept Incrementable = requires (T t, U u) { t += u; };

template <ForwardIterator Iter, typename Val>
    requires Incrementable<Val, ValueType<Iter>>
Val accumulate(Iter first, Iter last, Val acc)
{
    while(first != last) {
        acc += *first;
        ++first;
    }
    return acc;
}
```

在`Incrementable`设计时只用了一条关于`operator+=`的限制，这样
有些类只实现了`operator+`和`operator=`就不能使用这个算法了。
这样一来用户就会感到非常困惑。在这里一种比较好的方式并不是
仅仅定义一个当前使用时最小的限制，而是对`Val`的类型做出约束

```cpp
// concept Number约束这个类型必须实现了+, -, *, /, += ...这些运算符
template<ForwardIterator Iter, Number<ValueType<Iter>> Val>
Val sum(Iter first, Iter last, Val acc)
{
    while(first != last) {
        acc += *first;
        ++first;
    }
    return acc;
}
```

尽管在使用了`Number`作为约束后用户自定义的类变得繁琐，但是如果换一种
角度来看这又十分自然。在用户使用concepts时如果都是记着某个
concepts的约束具体是什么显然是糟糕的，在用了`Number`之后，
用户马上会从`Number`这个概念(Concept)联想到了它应该支持的具体操作。
针对抽象数据类型而不是具体实现进行程序设计非常适合人类的思维模式，
所以用户反而 更加不容易出错。在[Haskell](https://www.haskell.org/)
中也存在[typeclass](http://learnyouahaskell.com/types-and-typeclasses)这个概念，
它与concepts 有所不同，但相似的是它们都会约束类型的行为。
所以在使用concepts时 可以借鉴已经存在的typeclass在设计时的一些方式。
值得一提的是， Haskell作为一门函数式语言，在使用时会用到很多数学中的概念，
比如 群，半群，整数等等，可以预计如果concepts被正式使用，那么也会出现
很多基于数学概念的concepts设计。

尽管使用一个定义良好的完整concept是比较好的方式，“残缺”的concept
并不是无用武之地。在定义concepts时，经常会有很多重复的限制，使用
一些未完全设计好的concepts作为组件来帮助构造一个完整的concept能带
来许多便利。在[concepts简介](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-intro.md#requires语句)中，
我们有提到requires语句的混合语法就可以用来组合concepts。

## concepts作为类型设计的辅助

有了concepts以后，我们开始设计类型时将会用一种全新的角度来看待所需要
的函数。从前我们总是考虑有什么样算法需要实现，然后一个个去实现。但是
在有了concepts后，由于concepts的名字通常带有语义信息，
我们会自然而然地开始思考每一个类应该满足哪个concepts。与原先
任意想算法名字不同，concepts约束了它们，所以面向concepts设计的类型
会非常规范。

在测试面向concepts设计的类型时，由于concepts都是布尔类型，
我们可以使用`static_assert`来大大减轻 调试难度：

```cpp
static_assert<Number<MyType::value_type>>
static_assert<Sortable<MyType>>
```

## 将concepts加入到语言中的必要性

根据我们前面的[基于模板元编程的concepts实现](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-by-TMP.md)
中，concepts可以使用一些元编程的手段实现，但是同样的`for`和`while`
也可以用`if`和`goto`实现，但是将它正式加入到语言中意味着抽象能力的大幅提升。
人们不必依赖一些第三方实现的concepts库（这对商业化使用较多的C++来说是很重要的），
而可以直接使用。并且特别设计的语法会比模板库的使用方法更简单，编译器也会
提供语法级别的报错支持。除此以外，作为一个有着众多优点的特性加入到语言特性
也表示了一种官方的支持，更加容易推广它的使用。

以上几点是使用模板模拟不了的事情，因此将它作为语言扩展加入到语言中
也就显得很有必要了

## 总结

concepts的设计契合了C++的设计思想:

*   为用户提供了良好的接口
*   在语义上一致
*   不让用户去做适合机器去做的事情
*   零运行时开销
*   设计尽量简单

再回到最开始的[问题](https://github.com/ustc-compiler-concepts/report/blob/master/concepts-intro.md#C++中的泛型编程)，
concepts也的确是一个解决模板接口的好方法。

## 参考文献

1. [Concepts: The Future of Generic Programming](http://www.stroustrup.com/good_concepts.pdf)
2. [Haskell](https://www.haskell.org/)
