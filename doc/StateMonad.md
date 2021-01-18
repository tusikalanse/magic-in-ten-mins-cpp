# 十分钟魔法练习：状态单子

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础，HKT，Monad

## 函数容器

很显然c++标准库中的各类容器都是可以看成是单子的，在函数式的理论中单子不仅仅可以是实例意义上的容器，也可以是其他抽象意义上的容器，比如函数。

对于一个形如`std::function<std::complex<A>(S)>` 形式的函数来说，我们可以把它看成包含了一个 `A` 的惰性容器，只有在给出 `S` 的时候才能知道 `A` 的值。对于这样形式的函数我们同样能写出对应的 `flatMap` ，这里就拿状态单子举例子。

## 状态单子

状态单子（State Monad）是一种可以包含一个“可变”状态的单子，我这里用了引号是因为尽管状态随着逻辑流在变化但是在内存里面实际上都是不变量。

其本质就是在每次状态变化的时候将新状态作为代表接下来逻辑的函数的输入。比如对于：

```cpp
i++;
std::cout << i << std::endl;
```

可以用状态单子的思路改写成：

```cpp
[](int v) {std::cout << v << std::endl;}(i + 1);
```

最简单的理解就是这样的一个包含函数的对象：
`State` 是一个 Monad [注1]

```cpp
template <typename S>
class StateM
{
public:
  template <typename A>
  class State : public Monad<State>
  {
  public:
    std::function<StateData<S, A>(S)> runState;
    State() {}
    State(std::function<StateData<S, A>(S)> f) {
      runState = f;
    }
  };

  template <typename A>
  State<A> pure(A v)
  {
    return State<A>([v](S s) {
      return StateData<S, A>(s, v);
    });
  }

  template<typename A, typename B>
  State<B> flatMap(State<A> ma, std::function<State<B>(A)> f) 
  {
    return State<B>([ma, f] (S s) -> StateData<S, B> {
      StateData<S, A> r = ma.runState(s);
      return f(r.value).runState(r.state);
    });
  }
};
```

这里为了好看定义了一个 `StateData` 类，包含变化后的状态和计算结果。[注2]
```cpp
template <typename S, typename A>
class StateData {
public:
  S state;
  A value;
  StateData(S s, A v) {
    state = s; 
    value = v;
  }
};
```
而最核心的就是 `runState` 函数对象，通过组合这个函数对象来使变化的状态在逻辑间传递。

`pure` 操作直接返回当前状态和给定的值， `flatMap` 操作只需要把 `ma` 中的 `A` 取出来然后传给 `f` ，并处理好 `state` 。

仅仅这样的话 `State` 使用起来并不方便，还需要定义一些常用的操作来读取写入状态：

```cpp
// class StateM<S>
// 读取
  State<S> get = State<S>([](S s) {
    return StateData<S, S>(s, s);
  });
// 写入
  State<S> put(S s) 
  {
    return State<S>([s](S any){
      return StateData<S, S>(s, any);
    });
  }
// 修改
  State<S> modify(std::function<S(S)> f)
  {
    return State<S>([f] (S s) {
      return StateData<S, S>(f(s), s);
    });
  }
```

使用的话这里举个求斐波那契数列的例子：

```cpp

StateM<std::pair<int, int>>::State<int> fib(int n)
{
  StateM<std::pair<int, int>> m;
  if (n == 0)
    return m.flatMap<std::pair<int, int>, int>(m.get, [&m] (std::pair<int, int> x) {
      return m.pure<int>(x.first);
    });
  else if (n == 1)
    return m.flatMap<std::pair<int, int>, int>(m.get, [&m] (std::pair<int, int> x) {
      return m.pure<int>(x.second);
    });
  return m.flatMap<std::pair<int, int>, int>(m.modify([] (std::pair<int, int> x) {
    return std::make_pair(x.second, x.first + x.second);
  }),
  [n] (std::pair<int, int> unused) -> StateM<std::pair<int, int>>::State<int> {return fib(n - 1);}
  );
}

int main() {
  std::cout << fib(7).runState(std::make_pair(0, 1)).value << std::endl;
  return 0;
}
```

`fib` 函数对应的 Haskell 代码是：

```haskell
fib :: Int -> State (Int, Int) Int
fib 0 = do
  (_, x) <- get
  pure x
fib n = do
  modify (\(a, b) -> (b, a + b))
  fib (n - 1)
```

~~看上去比 c++ 版简单很多~~

## 有啥用

看到这里肯定有人会拍桌而起：求斐波那契数列我有更简单的写法！

```cpp
int fib(int n) {
    int a[3] = {0, 1, 1};
    for (int i = 0; i < n - 2; i++)
        a[(i + 3) % 3] = a[(i + 2) % 3] + 
                         a[(i + 1) % 3];
    return a[n % 3];
}
```

但问题是你用变量了啊， `State Monad` 最妙的一点就是全程都是常量而模拟出了变量的感觉。

更何况你这里用了数组而不是在递归，如果你递归就会需要在 `fib` 上加一个状态参数， `State Monad` 可以做到在不添加任何函数参数的情况下在函数之间传递参数。

同时它还是纯的，也就是说是**可组合**的，把任意两个状态类型相同的 `State Monad` 组合起来并不会有任何问题，比全局变量的解决方案不知道高到哪里去。


> [注1]：这里因为c++不太好实现模板参数转发因而采用了将State作为StateM的内部类的方案来减少一个模板变量

> [注2]：这里的构造函数变量顺序与java版不同，导致了put/modify也有少许不同

> [注3]：这里为了便于理解显式指明了某些lambda表达式的返回值
