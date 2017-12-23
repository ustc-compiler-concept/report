# Haskell中的type class

concepts的引入使得C++在泛型编程上的抽象机制变得更好, 但concept的思想其实在编程语言设计中早就出现, 比如Haskell中就很早实现了类似的机制, 叫做**type class(类型类)** (需要注意的是, 这里的类型类与OO语言中的class并不是同一概念).

## 什么是type class

类型类(type class)是Haskell最强大的功能之一: 它用于定义通用接口, 为各种不同的类型提供一组公共特性集. type class定义了一系列函数, 这些函数对于不同类型的值使用不同的函数实现.

以最经典的相等性测试举例, 我们如下定义一个类型类`BasicEq` : 

```haskell
class BasicEq t where
    isEqual :: t -> t -> Bool
```

其中跟在 `class` 之后的 `BasicEq` 是类型类的名字, 之后的 `t` 表示实例类型的类型变量. `BasicEq` 类型类只定义了一个 `isEqual` 函数, 它接受两个参数作为输入, 并且这两个参数都指向同一种实例类型.

以下代码将 `Bool` 类型作为 `BasicEq` 的实例类型，实现了 `isEqual` 函数:

```haskell
instance BasicEq Bool where
    isEqual True  True  = True
    isEqual False False = True
    isEqual _     _     = False
```

如果试图将不是 `BasicEq` 实例类型的值作为输入调用 `isEqual` 函数, 就会引发错误.



## Type Classes vs Concepts

根据\[2\]的研究, 在26个主要机制中, 16个是type class 与concept 都支持的, 其余只有1到2个是不能移植的, 并且二者在术语上可以很好地映射, 因此二者是非常相似的.

下面是同样的一段泛型编程代码在两种语言下的实现.

```cpp
// concept
template <typename X>
concept bool Hashset = HasEmpty<X> && requires () {
    typename X::element;
    requires Hashable<element>;
    { X.size() } -> int;
}

// modelling
template<Hashable K>
struct intmap<list<K>> {
    typedef K element;
    int size(intmap<list<K>> m) { ... }
}

// algorithm
bool almostFull(HashSet h) { ... }

// instantiation
intmap<list<int>> h;
bool test = almostFull(h);
```

```haskell
-- Haskell

-- concept
class (HasEmpty x, Hashable (Element x)) => Hashset x where
    type Element x
    size :: x -> Int

-- modelling
instance Hashable k => Hashset (IntMap (List k)) where
    type Element (IntMap (List k)) = k
    size m = ...

-- algorithm
almostFull :: Hashset t => t -> Bool
almostFull h = ...

-- instantiation
h :: IntMap (List Int)
test = almostFull h
```



## 总结

本文横向对比了Haskell中相似于C++中的concept的概念与机制, 说明concept的本质并不局限于某一具体语言的某一语法机制, 而是泛型编程中诞生的一种重要的抽象思想。它们都试图实现类型推导与重载函数的兼容, 在对函数进行类型扩展的同时, 加以语义上的限制, 让机器更好地服务于编程者.

## 参考文献

1. [yet another Haskell tutorial](https://www.umiacs.umd.edu/~hal/docs/daume02yaht.pdf)
2. [A comparison of C++ concepts and Haskell type classes](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.566.8506&rep=rep1&type=pdf)

