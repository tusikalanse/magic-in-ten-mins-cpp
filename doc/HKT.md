# 十分钟魔法练习：高阶类型

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础，模板基础

## 常常碰到的困难

> 这一节请参见[原版](https://github.com/goldimax/magic-in-ten-mins/blob/main/doc/HKT.md)，c++并没有这个问题

## 高阶类型

> 建议参见原版，因为c++可以用模板模板参数不需要加一层HKT转发（而且因为没有类型通配符？不好实现HKT转发层），代码实现与原版有很大不同

假设类型的类型是 `Type` ，比如 `int` 和 `string` 类型都是 `Type` 。

而对于 `list` 这样带有一个泛型参数的类型来说，它相当于一个把类型 `T` 映射到 `list<T>` 的函数，其类型可以表示为 `Type -> Type` 。

同样的对于 `map` 来说它有两个泛型参数，类型可以表示为 `(Type, Type) -> Type` 。

像这样把类型映射到类型的非平凡类型就叫高阶类型（HKT, Higher Kinded Type）。

可以写出这样一个 `Functor` ：

```cpp
template <template <typename> typename F>
class Functor {
public:
    template <typename B, typename A>
    F<B> map(F<A> a, std::function<B(A)> f) {}
};
```

这样，实现 `Functor` 类就是一件简单的事情了：

```cpp
template<class T>
using Vec = std::vector<T, std::allocator<T>>;

template<>
class Functor<Vec> {
public:
    template <typename B, typename A>
    Vec<B> map(Vec<A> a, std::function<B(A)> f) {
        Vec<B> result(a.size());
        std::transform(a.begin(), a.end(), result.begin(), f);
        return result;
    }
};
```
这里用`Vec`作为别名的原因是`vector`需要两个模板参数（第二个`allocator`一般是默认值），不能直接作为`F`

举个例子
```cpp
int main() {
    Functor<Vec> fv;
    std::vector<int> v{1, 2, 3, 4, 5, 6};
    std::vector<double> dv = fv.map<double, int>(v, [](int i) { return 0.5 + i; });
    return 0;
}
```
依次输出`dv`会得到`1.5 2.5 3.5 4.5 5.5 6.5`

> [注]： 模板函数是不能设为virtual（无法建立vtable），如确有需要可以固定参数类型重载你需要的模板函数