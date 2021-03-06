# 十分钟魔法练习：单位半群

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础

## 半群（Semigroup）

半群是一种代数结构，在集合 `A` 上包含一个将两个 `A` 的元素映射到 `A` 上的运算即 `<> : (A, A) -> A​` ，同时该运算满足**结合律**即 `(a <> b) <> c == a <> (b <> c)` ，那么代数结构 `{<>, A}` 就是一个半群。

比如在自然数集上的加法或者乘法可以构成一个半群，再比如字符串集上字符串的连接构成一个半群。

## 单位半群（Monoid）

单位半群（也称为幺半群，独异点）是一种带单位元的半群，对于集合 `A` 上的半群 `{<>, A}` ， `A` 中的元素 `a` 使 `A` 中的所有元素 `x` 满足 `x <> a` 和 `a <> x` 都等于 `x`，则 `a` 就是 `{<>, A}` 上的单位元。

举个例子， `{+, 自然数集}` 的单位元就是 0 ， `{*, 自然数集}` 的单位元就是 1 ， `{+, 字符串集}` 的单位元就是空串 `""` 。

用 C++ 代码可以表示为[注1]：

```cpp
template <typename T>
class Monoid {
public:
    Monoid() {};
    virtual T empty() = 0;
    virtual T append(T a, T b) = 0;
    T appends(const std::vector<T> &x) {
        return std::reduce(
            x.begin(), x.end(), empty(),
            std::bind(&Monoid<T>::append, this, std::placeholders::_1,
            std::placeholders::_2));
    }
};
```

## 应用：optional[注2]

在 c++17 中有个新的类 `optional` 可以用来表示可能有值的类型，而我们可以对它定义个 Monoid ：

```cpp
template <typename T>
class OptionalM : public Monoid<std::optional<T>> {
public:
    std::optional<T> empty() {
        return std::optional<T>();
    }
    std::optional<T> append(std::optional<T> a, std::optional<T> b) {
        if (a.has_value()) return a;
        return b;
    }
};
```

这样对于 appends 来说我们将获得一串 optional 中第一个不为空的值，对于需要进行一连串尝试操作可以这样写：

```cpp
std::optional<int> firstValue = OptionalM<int>().appends({
    try1(), 
    try2(),
    try3(),
    try4()});
```

## 应用：Ordering

使用类似java中的`compare`和`compareTo`可以构造出[注3]：

```cpp
class OrderingM : public Monoid<int> {
public:
    int empty() { return 0; }
    int append(int a, int b) {
        if (a == 0) return b;
        else return a;
    }
};

template <typename T>
int compare(const T &a, const T &b) {
    if (a > b) return 1;
    if (a == b) return 0;
    return -1;
}
```

同样如果有一串带有优先级的比较操作就可以用 appends 串起来，比如：

```cpp
class Student {
public:
    std::string name;
    std::string sex;
    int age;
    std::string from;
    int compareTo(const Student& other) const {
        Student s = (Student)(other);
        return OrderingM().appends({
          compare(name, s.name),
          compare(sex, s.sex),
          compare(age, s.age),
          compare(from, s.from)
        });
    }
};
```

这样的写法比一连串 `if-else` 优雅太多。

## 扩展

在 Monoid 接口里面添加方法可以支持更多方便的操作：

```cpp
template <typename T>
class Monoid {
    //...
    T when(bool c, T then) {
        if (c) return then;
        else return empty();
    }
    T cond(bool c, T then, T els) {
        if (c) return then;
        else return els;
    }
};

class Todo : public Monoid<std::function<void()>> {
public:
    std::function<void()> empty() {
        return [](){};
    }
    std::function<void()> append(std::function<void()> a, std::function<void()> b) {
        return [a, b](){a(); b();};
    }
};
```

然后就可以像下面这样使用上面的定义:

```cpp
Todo().appends({
        Todo().when(false, logic_when),
        logic1,
        Todo().when(true, logic_when),
        Todo().cond(false, logic_cond, logic_els),
        [&logic2](){logic2();},
        Todo().cond(true, logic_cond, logic_els),
    })();
```

该段代码中的logic均为打印自身变量名的匿名函数，结果为依次输出logic1, logic_when, logic_els, logic2, logic_cond

> [注1]：需要c++17， 这里用`vector`只是为了方便演示，若有需要也可以换成`list`或是两个迭代器，`appends`，也可以用```std::accumulate```代替```std::reduce```，只是```reduce```可以并行，更能展示单位半群可结合的特点。

> [注2]：需要c++17, optional 并不是 lazy 的，实际运用中加上非空短路能提高效率（需要更改`appends`的实现，可以派生出新类或是直接提供另一个接口）。

> [注3]：这里实现了类似c++20中的三路比较运算符的功能，即按字典序比较每一个成员，实际使用中可以修改`compare`的定义使其返回值更精确而非只有0和正负1。
