# 十分钟魔法练习：单子

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础, HKT

## 单子

单子(Monad)是指一种有一个类型参数的数据结构，拥有 `pure` （也叫 `unit` 或者 `return` ）和 `flatMap` （也叫 `bind` 或者 `>>=` ）两种操作：

```cpp
class Monad {
public:
  template <typename A>
  M<A> pure(A v);

  template <typename A, typename B>
  M<B> flatMap(M<A> ma, std::function<M<B>(A)> f);
};
```

其中 `pure` 要求返回一个包含参数类型内容的数据结构， `flatMap` 要求把 `ma` 中的值经过 `f` 以后再串起来。

举个最经典的例子：

## List Monad

```cpp
class HKTListM : public Monad<Vec> {
public:
  template <typename A>
  Vec<A> pure(A v) {
    Vec<A> list;
    list.push_back(v);
    return list;
  }

  template <typename A, typename B>
  Vec<B> flatMap(Vec<A> ma, std::function<Vec<B>(A)> f) {
    Vec<B> mb;
    for (A a : ma) {
      Vec<B> b = f(a);
      mb.insert(mb.end(), b.begin(), b.end());
    }
    return mb;
  }
};
```

其中`Vec = std::vector<T, std::allocator<T>>`
简单来说 `pure(v)` 将得到 `Vec<int>{v}` ，而 
```cpp
HKTListM().flatMap<int, double>(Vec<int>{1, 2, 3}, [] (int v) {
  return Vec<double>{v - 0.5, v + 0.5, v + 1.5};
});
``` 
将得到 `Vec<double>{0.5, 1.5, 2.5, 1.5, 2.5, 3.5, 2.5, 3.5, 4.5}` 。

## Maybe Monad

C++ 不是一个空安全的语言，也就是说任何指针都有可能为 `nullptr` 。对于一串可能出现空值的逻辑来说，判空常常是件麻烦事：

```cpp
Maybe<int> addI(Maybe<int> ma, Maybe<int> mb) {
  if (ma.value == nullptr) return Maybe<int>();
  if (mb.value == nullptr) return Maybe<int>();
  return Maybe<int>(new int(*ma.value + *mb.value));
}
```

其中 `Maybe` 是个 `HKT` 类型：

```cpp
template<typename T>
class Maybe {
public:
  T* value;
  Maybe() {value = nullptr;}
  Maybe(T p) {value = new T(p);}
};
```

像这样定义 `Maybe Monad` ：

```cpp
class MaybeM : Monad<Maybe> {
public:
  template <typename A>
  Maybe<A> pure(A v) {
    return Maybe<A>(v);
  }

  template <typename A, typename B>
  Maybe<B> flatMap(Maybe<A> ma, std::function<Maybe<B>(A)> f) {
    A* v = ma.value;
    if (v == nullptr) return Maybe<B>();
    else return f(*v);
  }
};
```

上面 `addI` 的代码就可以改成：

```cpp
Maybe<int> addM(Maybe<int> ma, Maybe<int> mb) {
  MaybeM m;
  return m.flatMap<int, int>(ma, [&] (int a) mutable {    // a <- ma
    return m.flatMap<int, int>(mb, [&] (int b) mutable {  // b <- mb
      return m.pure(a + b);                               // pure (a + b)
    });
  });
}
```

这样看上去就比上面的连续 `if-return` 优雅很多。在一些有语法糖的语言 (`Haskell`) 里面 Monad 的逻辑甚至可以像上面右边的注释一样简单明了。

> 有些人可能会说使用`try...catch`来判断，如以下代码（为了和java原版对应）
>
> ```cpp
> Maybe<int> addE(Maybe<int> ma, Maybe<int> mb) {
>   try { 
>     return Maybe<int>(*ma.value + *mb.value);
>   } 
>   catch (...) {
>     return Maybe<int>();
>   }
> }
> ```
>
> 遗憾的是，C++并没有空指针异常这种东西，try里的那句return可能会直接引发core dump
> `Maybe Monad` 在的 C++ 里面不是一个很好的例子，因为C++17标准库提供了`std::optional`（见单位半群章节）。
> 不过 `Maybe Monad` 确实是在其他没有异常的函数式语言里面最为常见的 Monad 用法之一。
> 不过即使C++有空指针异常，上面那段`addE`可以用，异常对另外一些 Monad 用法依然无能为力，在接下来的章节中会介绍。

> [注1]： C++标准库里并没有类似java中`stream`的`flatMap`方法，只能手动`for`循环实现一下
> [注2]： 因为java原版用的NPE作为Mabye的例子，而C++不允许空引用，只能有着类似现象的指针来展示，写起来有些地方不太美观（指`addM`）