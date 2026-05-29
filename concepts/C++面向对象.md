# C++面向对象

See also: [[C++语言特性]], [[C++特有面向对象特性]], [[静多态与动态多态]], [[设计模式]]

## 面向对象四大核心特性

C++ 作为面向对象语言，原生完整支持如下四大特性：

1. **封装（Encapsulation）**  
   - 将数据和操作数据的函数组织到类中，隐藏内部细节，仅通过公开接口与外界交互，提高安全性和灵活性。

2. **继承（Inheritance）**  
   - 新类可以复用/扩展已有类的成员和行为。支持单继承和多继承，实现代码重用及体系化结构。

3. **多态（Polymorphism）**  
   - 同一接口可对应不同实现，常用 `virtual` 虚函数实现运行时多态，使调用代码与具体对象解耦。

4. **抽象（Abstraction）**  
   - 通过抽象类（含纯虚函数）描述共性接口，仅暴露外部必要操作，屏蔽实现细节，让使用者聚焦"做什么"而不是"怎么做"。

这些特性都是通过 C++ 语言本身的机制实现（如类、继承、访问控制、虚函数、抽象类等），是其区别于 C 等过程式语言的核心基础。

### 封装

通过 `class` 和 `struct` 将数据和方法封装到类型内部，使用访问控制修饰符（`public`、`protected`、`private`）隐藏内部实现、暴露必要接口。

```cpp
class Person {
private:
    int age;   // 封装的内部数据

public:
    Person(int a) : age(a) {}
    void setAge(int a) { age = a; }
    int getAge() const { return age; }
};
```

**作用**：隐藏实现细节，提高安全性和可维护性。

### 继承

支持类之间的继承（单继承、多继承），子类可扩展和复用父类的成员和行为。支持 `public`/`protected`/`private` 继承方式。

```cpp
class Animal {
public:
    virtual void eat() {}
};

class Dog : public Animal {
public:
    void bark() {}
};

// 多继承（C++ 特色，Java 不支持）
class A {}; class B {};
class C : public A, public B {};
```

**特点**：支持虚函数表（vtable）、虚继承（解决菱形继承问题）。

**public/protected/private 继承方式说明：**

- `public 继承`：基类的 public 成员变为派生类的 public，protected 变为 protected，private 依然不可访问（最常用，继承"is-a"关系）
- `protected 继承`：基类的 public 和 protected 变为派生类的 protected（外界不可访问，常用于库底层或实现派生时限制外部访问）
- `private 继承`：基类的 public 和 protected 成员都变为派生类的 private（完全封装，派生类外不可访问基类成员，体现"实现"而非"接口/继承"关系）

**使用场景：**

- **public 继承**：最常见，表示"是一个"（is-a）关系。子类对外暴露基类接口，外部可以将子类对象视为基类对象使用。
- **protected 继承**：强调"实现"关系，而非"接口"关系。只允许派生类自身及其子类访问继承的成员，对外不可见。
- **private 继承**：最强封装，完全隐藏基类接口，只把基类作为实现细节（类似"用某种方式实现"）。典型应用：适配器模式、控制基类 API 暴露范围等。

```cpp
class Thread {
public:
    void start() { /* ... */ }
    void join()  { /* ... */ }
protected:
    int thread_id;
};

class Printer : private Thread {
public:
    void printDocument() {
        start();  // 直接复用 Thread 的接口
        // ... 打印相关逻辑
        join();
    }
};
// 外部代码无法通过 Printer 对象调用 start()/join()，也无法把 Printer* 强转为 Thread*
```

### 多态

通过虚函数（`virtual`）和纯虚函数（`=0`）实现运行时多态，支持基类指针/引用调用派生类实现。

```cpp
class Animal {
public:
    virtual void speak() = 0;  // 纯虚函数
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void speak() override { std::cout << "Woof\n"; }
};

Animal* animal = new Dog();
animal->speak();  // 输出 Woof，运行时多态
```

**两种多态**：
- **编译期多态（静多态）**：函数重载、运算符重载、模板、CRTP 等
- **运行期多态（动态多态）**：虚函数 + 动态绑定

**重载（overload） vs 重写/覆盖（override）：**

- **重载**  
  - 同一作用域/同一类中：**函数名相同，参数列表不同**（参数个数/类型/顺序不同）。  
  - 与继承无关，可在同一个类或同一命名空间内发生。  
  - 由编译器在**编译期根据实参类型**选择具体函数 → **静多态**。

- **重写/覆盖**  
  - 必须有**继承关系**，且基类函数是 `virtual`。  
  - 派生类提供与基类虚函数**相同签名**的实现，并建议显式写上 `override`：  
    `void func(int) override;`。  
  - 通过基类指针/引用在**运行期根据对象实际类型**选择实现 → **动态多态**。

一句话记忆：  
- 重载：**同名不同参，看参数列表，编译期决议**。  
- 重写：**子类改写基类虚函数，看继承 + virtual/override，运行期通过虚表决议**。

（"动态绑定"是指程序在运行时根据对象实际类型决定调用哪个虚函数的机制，即"晚绑定"。）

**虚表在子类和父类是一样的吗？**

不是完全一样。每个带有虚函数的类都对应自己的虚表。子类继承并重写虚函数时，编译器会为子类生成自己的虚表：
- 子类未重写：虚表项指向父类实现
- 子类重写：虚表项指向子类实现
- 子类可添加新虚函数，指针加入子类虚表

**虚表是在只读数据段（rodata）吗？**

在大多数编译器（GCC、Clang）和平台下，虚表一般会被放在只读数据段 `.rodata` 中。虚表内容编译后就定了，运行时不会改变。

### 抽象

抽象是指只暴露对象对外的必要接口，隐藏内部实现细节。在 C++ 中，接口通常用抽象类（至少包含一个纯虚函数）来表达。

```cpp
class Shape {
public:
    virtual double area() const = 0;     // 纯虚函数 -> 抽象接口
    virtual void info() const {
        std::cout << "This is a shape." << std::endl;
    }
    virtual ~Shape() = default;
};
```

**作用**：让使用者关注"做什么"，而不是"怎么做"，提高拓展性和可替换性。

[src: raw/ingested/2技术/cpp/C++基础语法手册-面向对象四大核心特性.md]

## Related Pages
- [[C++语言特性]]
- [[C++特有面向对象特性]]
- [[静多态与动态多态]]
- [[设计模式]]
- [[C++类型转换]]
