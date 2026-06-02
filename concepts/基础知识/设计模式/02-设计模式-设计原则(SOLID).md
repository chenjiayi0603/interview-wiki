# 设计原则（SOLID）

> 面向对象设计的五个基本原则，是学习设计模式的基础。

---

## 一、单一职责原则（SRP）

一个类只负责一个职责，只有一个引起变化的原因。

```cpp
// ❌ 违反 SRP：一个类做太多事
class User {
public:
    void save() { /* 保存用户 */ }
    void sendEmail() { /* 发送邮件 */ }
    void generateReport() { /* 生成报告 */ }
};

// ✅ 遵循 SRP：职责分离
class UserRepository { void save(const User&) { } };
class EmailService { void send(const User&) { } };
class ReportGenerator { void generate(const User&) { } };
```

---

## 二、开闭原则（OCP）

对扩展开放，对修改关闭。通过抽象和多态实现。

```cpp
// ❌ 违反 OCP：扩展需要修改类
class Shape {
public:
    enum Type { CIRCLE, SQUARE };
    Type type;
    double radius;
    double side;
    double area() {
        if (type == CIRCLE) return 3.14 * radius * radius;
        if (type == SQUARE) return side * side;
    }
};

// ✅ 遵循 OCP：扩展新形状只需新增子类
class Shape { public: virtual double area() = 0; };

class Circle : public Shape {
    double radius;
public:
    double area() override { return 3.14 * radius * radius; }
};

class Square : public Shape {
    double side;
public:
    double area() override { return side * side; }
};
```

---

## 三、里氏替换原则（LSP）

子类对象必须能替换父类对象且不改变程序正确性。

```cpp
// ❌ 违反 LSP：正方形继承长方形导致行为不一致
class Rectangle {
protected:
    double width, height;
public:
    void setWidth(double w) { width = w; }
    void setHeight(double h) { height = h; }
};

class Square : public Rectangle {
public:
    void setWidth(double w) override { width = height = w; }   // ❌
    void setHeight(double h) override { width = height = h; }  // ❌
};

// ✅ 遵循 LSP：正方形不应该继承长方形，用组合或继承 Shape
```

**判断依据**：如果替换后程序行为不符合预期，则违反 LSP。

---

## 四、接口隔离原则（ISP）

客户端不应被迫依赖它不使⽤的接口。用多个专⻔接口代替一个大而全的接口。

```cpp
// ❌ 违反 ISP：大而全的接口
class IMachine {
public:
    virtual void print() = 0;
    virtual void scan() = 0;
    virtual void fax() = 0;
};

// ✅ 遵循 ISP：拆分为小接口
class IPrinter { virtual void print() = 0; };
class IScanner { virtual void scan() = 0; };
class IFax { virtual void fax() = 0; };

class MultiFunctionPrinter : public IPrinter, IScanner, IFax {
    void print() override { }
    void scan() override { }
    void fax() override { }
};
```

---

## 五、依赖倒置原则（DIP）

高层模块不应依赖低层模块，两者都应依赖抽象（接口）。

```cpp
// ❌ 违反 DIP：高层直接依赖具体实现
class UserService {
    MySQLDatabase db;  // 直接依赖具体类
public:
    void getUser(int id) {
        db.connect();
        db.query("SELECT...");
    }
};

// ✅ 遵循 DIP：依赖抽象接口
class IDatabase {
public:
    virtual void connect() = 0;
    virtual void query(const char* sql) = 0;
};

class MySQLDatabase : public IDatabase { /* 实现 */ };

class UserService {
    IDatabase* db;  // 依赖抽象
public:
    UserService(IDatabase* database) : db(database) {}
    void getUser(int id) {
        db->connect();
        db->query("SELECT...");
    }
};
```

---

## 六、记忆口诀

- **S**RP — **单**一职责：一个类只做一件事
- **O**CP — **开**闭：对扩展开放，对修改关闭
- **L**SP — **里**氏替换：子类可替换父类
- **I**SP — **接**口隔离：小接口优于大接口
- **D**IP — **依**赖倒置：依赖抽象不依赖具体

> 谐音记忆：**SOLD I** → "Sold I"（卖出我）→ 设计模式就是卖我的技巧 😄

---

## 七、SOLID 综合对比

| 原则 | 核心问题 | 违反症状 | 解决手段 |
|------|---------|---------|---------|
| SRP | 一个类承担太多职责 | 类过大、修改频繁 | 拆分为多个小类 |
| OCP | 扩展需要改现有代码 | switch/if-else 链 | 抽象基类 + 多态 |
| LSP | 子类破坏父类契约 | 重写方法改变原行为 | 用组合代替继承 |
| ISP | 接口过于庞大 | 实现类有很多空方法 | 拆分小接口 |
| DIP | 高层依赖低层实现 | 直接 new 具体类 | 依赖抽象（DI） |

**实践要点**：
- SOLID 不是教条，过度遵循也会导致代码膨胀
- 先识别"变化点"（需求中最常改的地方），再用 SOLID 衡量
- **最常见面试题**：给一段代码，指出违反了什么原则并改正
