# 十分钟魔法练习：代数数据类型

### By 「玩火」改写「图斯卡蓝瑟」

> 前置技能：C++面向对象基础

## 积类型（Product type）

积类型是指同时包括多个值的类型，比如 C++ 中的 class 就会包括多个字段：


```cpp
class Student final {
    std::string name;
    int id;
};
```

而上面这段代码中 Student 的类型中既有 string 类型的值也有 int 类型的值，可以表示为 string 和 int 的「积」，即 `string * int` 。

## 和类型（Sum type）

和类型是指可以是某一些类型之一的类型，在 C++ 中可以用继承来表示：

```cpp
class VirtualClass {
public:
    virtual ~VirtualClass() {};
};

class SchoolPerson : public VirtualClass {};

class Student final : public SchoolPerson {
    std::string name;
    int id;
};

class Teacher final : public SchoolPerson {
    std::string name;
    std::string office;
};
```

SchoolPerson 可能是 Student 也可能是 Teacher ，可以表示为 Student 和 Teacher 的「和」，即 `string * int + string * string` 。而使用时只需要用 `typeid` 就能知道当前的 StudentPerson 具体是 Student 还是 Teacher [注1]。

## 代数数据类型（ADT, Algebraic Data Type）

由和类型与积类型组合构造出的类型就是代数数据类型，其中代数指的就是和与积的操作。

利用和类型的枚举特性与积类型的组合特性，我们可以构造出 C++ 中本来很基础的基础类型，比如枚举布尔的两个量来构造布尔类型：

```cpp
class Bool : public VirtualClass {};

class True final : public Bool {};

class False final : public Bool {};
```

然后用 `typeid(t) == typeid(True)` 就可以用来判定 t 作为 Bool 的值是不是 True [注2]。

比如利用S的数量表示的自然数：

```cpp
class Nat : public VirtualClass {};

class Z : public Nat {};

class S : public Nat {
    Nat* value;
public:
    S(Nat* v) : value(v) {}
};
```

这里提一下自然数的皮亚诺构造，一个自然数要么是 0 (也就是上面的 Z ) 要么是比它小一的自然数 +1 (也就是上面的 S ) ，例如3可以用 `new S(new S(new S(new Z))` 来表示。

再比如链表：

```cpp
template <typename T>
class List : public VirtualClass {};

template <typename T>
class Nil final : public List<T> {};

template <typename T>
class Cons final : public List<T> {
    T value;
    List<T>* next;
public:  
    Cons(T v, List<T>* n) : value(v), next(n) {}
};
```

`[1, 3, 4]` 就表示为 `List<int>* list = new Cons<int>(1, new Cons<int>(3, new Cons<int>(4, new Nil<int>())))`

更奇妙的是代数数据类型对应着数据类型可能的实例数量。

很显然积类型的实例数量来自各个字段可能情况的组合也就是各字段实例数量相乘，而和类型的实例数量就是各种可能类型的实例数量之和。

比如 Bool 的类型是 `1+1 `而其实例只有 True 和 False ，而 Nat 的类型是 `1+1+1+...` 其中每一个1都代表一个自然数，至于 List 的类型就是`1+x(1+x(...))` 也就是 `1+x^2+x^3...` 其中 x 就是 List 所存对象的实例数量。

## 实际运用

ADT 最适合构造树状的结构，比如解析 JSON 出的结果需要一个聚合数据结构。

```cpp
class JsonValue : public Virtualclass {};

class JsonBool final : public JsonValue {
    bool value;
};

class JsonInt final : public JsonValue {
    int value;
};

class JsonString final : public JsonValue {
    std::string value;
};

class JsonArray final : public JsonValue {
    std::list<JsonValue> value;
};

class JsonMap final : public JsonValue {
    std::map<std::string, JsonValue*> value;
};
```

> [注1]: 由于c++的RTTI机制, typeid运算符只会对指向子类的基类指针（需要解引用）或引用起作用，并且需要基类有虚函数，因此写了一个Virtualclass提供虚函数
>
> [注2]: 需要t是指向子类的基类指针（需要解引用）或引用
>
> [注3]: 上面的和类型代码都存在用户可能自己写一个子类的问题，可以做成类似 Java / C# 中的 sealed class，一种方法为将基类的析构函数设为private并将所有子类设为友元，这样可以组织用户自己写的子类的实例化。
>
> [注4]: 上面的和类型代码存在用户实例化基类的问题，可以将基类作为纯虚类（例如将VirtualClass的析构函数设为纯虚函数），子类的析构函数需要实例化，考虑到代码篇幅问题没有采用此方法
>
> [注5]: 上面的写法是基于变量非空假设的，也就是代码中不会出现 null ，所有变量也不为 null 。
