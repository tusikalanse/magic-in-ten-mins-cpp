# 十分钟魔法练习：广义代数数据类型

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础，ADT

在 ADT 中可以构造出如下类型：

```cpp
// 构造函数已省去
class Expr : public VirtualClass {};

class IVal : public Expr {
    int value;
};

class BVal : public Expr {
    bool value;
};

class Add : public Expr {
    Expr *e1, *e2;
};

class Eq : public Expr {
    Expr *e1, *e2;
};
```

但是这样构造有个问题，很显然 `BVal` 是不能相加的而这样的构造并不能防止构造出这样的东西。实际上在这种情况下ADT的表达能力是不足的。

一个比较显然的解决办法是给 `Expr` 添加一个类型参数用于标记表达式的类型：

```cpp
// 构造函数已省去
template<typename T>
class Expr : public VirtualClass {};

class IVal : public Expr<int> {
    int value;
};

class BVal : public Expr<bool> {
    bool value;
};

class Add : public Expr<int> {
    Expr *e1, *e2;
};

class Eq : public Expr<bool> {
    Expr *e1, *e2;
};
```

这样就可以避免构造出两个类型为 `Bval` 的表达式相加，能构造出的表达式都是类型安全的。

注意到四个 `class` 的父类都不是 `Expr<T>` 而是包含参数的 `Expr` ，这和 ADT 并不一样。而这就是广义代数数据类型（Generalized Algebraic Data Type, GADT）。

